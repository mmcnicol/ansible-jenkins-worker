# Jenkins Worker Node on RHEL 10 — design notes

Status: **draft for team review**
Connection method assumed: **SSH, controller-initiated** ("Launch agents via SSH").

---

## 1. What the `jenkins_worker` role will do

1. Create the `jenkins` system user (see §3).
2. Install the pinned **JDKs**, **Maven**, **Flyway** and **SQLcl** as tarballs under
   `/Dev_Data/tools` (see §2, §2.1).
3. Deploy a managed Maven `settings.xml` (no credentials in it, see §2.2).
4. Install and configure **rootless Podman** for `jenkins` (storage on `/Dev_Data`,
   network, `podman-docker`, `docker-compose` binary, see §4).
5. Lay down the **prune** user systemd timer (see §5).
6. Install the OS packages via `dnf` (see §6).
7. Install **Node.js**, and **Newman** via npm, for Postman API tests (see §7).
8. Publish host facts/labels so the controller knows what the node offers.

Controller-side node creation is **not** done by this role (see §10).

---

## 2. JDK and Maven — install approach

**Recommendation: pinned vendor tarballs to `/Dev_Data/tools`, exposed to Jenkins as named
tools.** Rationale:

- You want *specific* versions, and **four** JDKs / **at least one** Maven side by side —
  `alternatives` juggling and "whatever AppStream ships this month" both fight that.
- Decouples the workers from the OS update cadence.
- This is exactly how Jenkins' own tool installers work; we're just pre-staging them.

Real fleet shape (versions genericised — **do not commit real version numbers to this
public repo**; set them as `host_vars`/`group_vars` in a private inventory or vault, using
the placeholders below only as examples of the variable shape):

| Role var (example name) | Example placeholder | Real-world equivalent you described |
|---|---|---|
| `jdk_versions.jdk8` | `8u<build>` | Currently **Oracle** JDK 8 — see licensing note below |
| `jdk_versions.jdk11` | `11.0.x` | Red Hat build of OpenJDK |
| `jdk_versions.jdk17_a` | `17.0.x-a` | Red Hat build of OpenJDK (older patch, pinned for a project) |
| `jdk_versions.jdk17_b` | `17.0.x-b` | Red Hat build of OpenJDK (newer patch) |
| `maven_versions.default` | `3.8.x` | your current `/opt/maven` |

> **Oracle JDK 8 licensing flag.** Since January 2019, Oracle's JDK 8 updates require a
> commercial license (Java SE subscription) for most production/commercial use — "free"
> Oracle downloads are for personal use or specific OTN terms only. You're already
> standardising the newer JDKs on the **Red Hat build of OpenJDK**; the same free, actively
> patched OpenJDK 8 builds exist (Red Hat's `java-1.8.0-openjdk` from AppStream, or
> Temurin 8 tarballs) and are drop-in for a build JDK. Worth raising with whoever owns
> license compliance before replicating Oracle JDK 8 onto 6 more nodes.

| Tool | Source | Path | Verify |
|------|--------|------|--------|
| JDKs (×4) | **Red Hat build of OpenJDK** tarballs where available, else **Temurin** | `/Dev_Data/tools/jdk/<label>-<version>` | SHA-256 checksum (mandatory) |
| Maven | **Apache Maven** binary tarball | `/Dev_Data/tools/maven/apache-maven-<version>` | SHA-512 checksum + PGP signature |

- Keep a **packaged** JDK (`dnf install java-21-openjdk-headless`) as the OS default for
  system tooling; the pinned tarball builds are for *builds*, selected per job.
- Do **not** put any JDK/Maven ambiguously on the global `PATH`. Let Jenkins tool
  configuration select per job (see below); optionally set one default via `profile.d`
  for interactive troubleshooting only.
- Download source: upstream now (with `checksum:` always set) via an `artifact_base_url`
  variable; flip that to the Nexus `raw` repo in Phase 2.
- **`JAVA_HOME`**: you noted `/opt/maven` needed `JAVA_HOME` set and it hadn't been. The
  role must set this reliably two ways: (a) Jenkins' JDK tool config sets `JAVA_HOME` for
  the duration of a build automatically — this is the primary mechanism, works
  per-job/per-JDK; (b) a `/etc/profile.d/jdk-default.sh` exported by the role, pointing at
  one designated default JDK, so `ssh jenkins@host` / ad-hoc shell troubleshooting and any
  tool invoked outside a Jenkins job (SQLcl, Flyway — both are Java apps) also finds a
  JDK without extra steps.
- **`.m2` location**: you found `.m2` currently at `/home/jenkins/.m2/repository` — an
  unbounded, ever-growing local repository cache sitting wherever `jenkins`'s home happens
  to be. This is a live example of why §3 puts `jenkins`'s home on `/Dev_Data`: doing that
  on the new nodes puts `.m2` on external storage automatically, no separate
  `<localRepository>` override needed.

In **Jenkins → Manage Jenkins → Tools**, add fixed entries (install automatically = off),
one per row in the table above, using the paths the role lays down. Jobs then request the
tool they need by label. Because every worker uses identical paths, this config is
identical across the fleet and belongs in JCasC (Phase 3).

### 2.1 Other pinned Java/CLI tools — same pattern

You're also installing **Flyway** and **SQLcl**, both distributed only as tarballs and
both Java applications (need the `JAVA_HOME` handling above). Treat them exactly like
Maven:

| Tool | Path | Notes |
|------|------|-------|
| Flyway | `/Dev_Data/tools/flyway/flyway-<version>` | pin, checksum |
| SQLcl | `/Dev_Data/tools/sqlcl/sqlcl-<version>` | pin, checksum; needs a JDK on `JAVA_HOME`/`PATH` |

### 2.2 Maven `settings.xml` and the embedded Nexus credentials

You mentioned the current `settings.xml` embeds username/password for a generic Nexus
`maven` user in plain text on the worker. That's exactly the kind of thing
[devops-policy.md §7](devops-policy.md#7-users-ssh-and-secrets) says shouldn't sit in a
file on disk. You already flagged this as a known-debt item, not something to fix inside
this project right now — noting it here so it's tracked. Better shape, worth doing when
Nexus lands in Phase 2 (ties into "one Nexus role for humans, one for CI" from the
roadmap):

- Keep `settings.xml` itself free of secrets — reference `${env.NEXUS_MAVEN_PASSWORD}` (or
  a Maven `<server><password>` that reads from an environment variable via a settings
  interpolation, or `settings-security.xml` with a master-password-encrypted value).
  Maven doesn't support `${env.*}` in `<password>` natively — the common patterns are
  either a Jenkins **Credentials Binding** step that writes a short-lived
  `settings-local.xml`/`-s` file per build, or an encrypted password via
  `settings-security.xml` (better than plaintext, still a static secret on disk).
  Short-lived, per-build injection from Jenkins Credentials is the stronger fix.
- The repo/settings structure (RH repo, PrimeFaces repo, your Nexus-hosted repos) is fine
  as-is and belongs in the role as a managed template — just without the password baked
  in.

---

## 3. The `jenkins` user

```
useradd --system --create-home --home-dir /Dev_Data/jenkins --shell /bin/bash jenkins
```

- **Home on `/Dev_Data`** so workspace + rootless Podman storage sit on external storage.
- **Password locked** (`!` in `/etc/shadow` — Ansible `user: password: '!'`, or `passwd -l`).
- **SSH key-only.** `~jenkins/.ssh/authorized_keys` gets the **Jenkins controller's**
  credential public key. Global sshd config already forbids password auth.
- **`loginctl enable-linger jenkins`** so `systemctl --user` timers and rootless Podman
  work with no active login session.
- Verify **`/etc/subuid`** and **`/etc/subgid`** have a `jenkins` range (RHEL 10 `useradd`
  normally adds it) — required for rootless Podman user-namespacing.

### Should `jenkins` have a password / bash login initially, to set things up?

**No.** Everything in worker setup is done by Ansible as the `ansible` user (with
`become_user: jenkins` where a step must run as `jenkins`) or as root. Nothing requires an
interactive `jenkins` login. Keep the account locked from creation.

### Does this satisfy "no password, cannot log in to a bash prompt"?

Yes, in the sense the security policy means it: no password, no password-auth over SSH, no
console access. The account *can* execute a shell — but only when the **controller** opens
an SSH session with its key, which is the whole point of the SSH agent launch method. The
alternative (`/sbin/nologin`) breaks the SSH agent launch and complicates rootless Podman;
only switch to it if you move the fleet to **inbound/JNLP** agents running as a system
service.

---

## 4. Rootless Podman (migrating from Docker)

- Install: `dnf install podman` (from AppStream — a legitimate package-manager install).
- **Rootless storage on `/Dev_Data`:** `~jenkins/.config/containers/storage.conf` with
  `graphroot = "/Dev_Data/jenkins/containers/storage"`. Ask the DC team to map
  `/Dev_Data/jenkins/containers` to expandable external storage — **same rationale as
  Docker**: a full image store must not wedge the host.
- **SELinux:** set the right fcontext on the graphroot and `restorecon` (exact type to be
  confirmed on the test VM — expect `container_var_lib_t` territory). Record what you set
  in the role.
- **linger** must be enabled for `jenkins` (see §3) or containers die when the SSH session
  ends.
- **Networking:** rootless uses `pasta`/`slirp4netns` with `netavark` + `aardvark-dns`
  (defaults on RHEL 10). Rootless can't bind ports < 1024 without extra config.

### Docker → Podman migration gotchas for job authors

- Jobs that talk to `/var/run/docker.sock` → enable the **user** `podman.socket` and set
  `DOCKER_HOST=unix://$XDG_RUNTIME_DIR/podman/podman.sock`.
- `docker-compose` → `podman compose` / `podman-compose`.
- File ownership in bind-mounted volumes shifts under user-namespace mapping — expect to
  fix a few builds.
- `docker build` → `podman build` is drop-in; consider `buildah` for advanced cases.

### 4.1 What you're actually running — matches this design, made idempotent

Confirmed shape: **`podman-docker`** (gives a `docker` → `podman` CLI shim) +
**rootless `podman.socket`** + the real **`docker-compose`** binary (not
`podman-compose`, deliberately) pointed at that socket via `DOCKER_HOST`. That's a sound
combination — `docker-compose` needs a Docker-API-speaking socket, and Podman's socket
speaks that API. The role will make each step idempotent instead of the one-off manual
commands:

| Manual command you ran | Role equivalent |
|---|---|
| `sudo -u jenkins systemctl --user enable --now podman.socket` | `become_user: jenkins` + `ansible.builtin.systemd` (`scope: user`), gated on linger being enabled first |
| `sudo -u jenkins podman network create MyNetwork` | `containers.podman.podman_network` (module is idempotent; add `containers.podman` to `collections/requirements.yml`); network name as a `group_vars` variable |
| `docker-compose` downloaded from the GitHub releases page | Same "pinned binary" pattern as JDK/Maven: `get_url` with a pinned version + checksum to `/Dev_Data/tools/docker-compose/<version>/docker-compose`, symlinked into a `jenkins`-owned bin dir on `PATH` |
| `podman-docker` package | `dnf` (approach 1 — legitimate OS package) |

**Environment variables** (`DOCKER_HOST`, `TESTCONTAINERS_RYUK_DISABLED`) are currently set
in the Jenkins node's *Node Properties* in the UI — manual, controller-side, not in this
repo. Two ways to make that reproducible instead, in order of preference:

1. **`~jenkins/.ssh/environment`**, populated by the role, with `PermitUserEnvironment yes`
   added to `sshd_config` (scoped — only takes effect for keys/sessions where the file
   exists). This is visible to the exact process the controller launches over SSH,
   regardless of shell init files, and is fully Ansible-managed.
2. Keep them as manual Jenkins **Node Properties** for now, added to the controller-side
   checklist in §10 — simplest, but drifts from "config as code" and has to be repeated by
   hand for 6 nodes (and every node after).

Recommendation: (1), rolled into the `jenkins_worker` role; migrate to JCasC node
definitions in Phase 3 either way.

`TESTCONTAINERS_RYUK_DISABLED=true` is a known workaround for Ryuk (Testcontainers'
cleanup sidecar) under rootless Podman. Disabling it means Testcontainers won't
auto-remove containers after a crashed/killed JVM — which is exactly what the **prune
timer** in §5 exists to mop up. Keep both.

> **Live-test finding (corrected after a second round of testing):** an actual
> `ssh jenkins@host '...'` — the real shape of a controller-launched SSH agent — sources
> `/etc/profile.d/*.sh` fine: OpenSSH runs both interactive and command-mode sessions as a
> **login shell**, so `~/.bash_profile` → `~/.bashrc` → `/etc/bashrc` → `/etc/profile.d/*`
> all fire. The `newman: node: No such file or directory` failure that first suggested
> otherwise came from testing via `ansible.builtin.command` with `become_user: jenkins`
> (i.e. `sudo -u jenkins ...`) — `sudo` does **not** invoke a login shell, so it sees
> neither `/etc/profile.d` nor `~/.ssh/environment` (confirmed directly: `sudo -u jenkins
> bash -c 'echo $PATH'` on the test VM returns a bare `/sbin:/bin:/usr/sbin:/usr/bin`).
> That's a gap in `sudo -u jenkins ...`-style troubleshooting, not in the real Jenkins
> connection.
>
> `JAVA_HOME`/`PATH` are still set in **both** places: `/etc/profile.d` for interactive
> login convenience, and `~jenkins/.ssh/environment` as a second, more direct route that
> doesn't depend on `jenkins` keeping `/bin/bash` as its login shell or on the
> `.bash_profile`/`.bashrc` chain staying intact. Redundant on a stock setup (you'll see
> each tool's directory in `$PATH` twice), harmless, and cheap insurance.

---

## 5. Image prune

A **user** systemd timer for `jenkins`, laid down by the role (not a Jenkins job — see
[devops-policy.md §5](devops-policy.md#5-container-image-cleanup-prune)):

```
# ~jenkins/.config/systemd/user/podman-prune.service  (oneshot)
podman system prune -f --filter until=720h
podman volume prune -f
```

with a `podman-prune.timer` (e.g. weekly). Tune the `until=` filter; optionally skip if a
build container is running. Logs land in `journalctl --user -u podman-prune`.

---

## 6. Packages installed via `dnf`

Turns out this list is a bit longer than "one or two" — all legitimate approach-(1) OS
packages, not services:

| Package | Why |
|---|---|
| `git` | SCM |
| `python3` | Ansible itself needs an interpreter on the target; also general scripting. Worth an explicit task even though RHEL 10 likely ships it, so the role doesn't silently depend on the base image. |
| `podman-docker` | `docker` CLI shim over Podman (§4.1) |
| `sshpass` | see §11 — kept for now as a known-debt item, not removed by this role |

Modelled as a `worker_packages` list var in the role so it's easy to add/remove per
project without touching tasks.

## 7. Node.js, npm, Newman (Postman API tests)

Same pinned-version reasoning as JDK/Maven: **tarball to `/Dev_Data/tools`**, not `dnf`
and not `nvm` (version managers are for laptops per
[devops-policy.md §2](devops-policy.md#2-how-to-install-a-software-tool-or-run-a-service)).

- **Node.js**: official prebuilt `linux-x64` tarball, pinned version + checksum, to
  `/Dev_Data/tools/node/node-<version>`, exposed on `PATH` the same way as Maven (Jenkins
  tool config if you install the NodeJS plugin, or a fixed `PATH` entry per job/label —
  your call once you see how many jobs need it).
- **npm global installs (Newman)**: default `npm install -g` wants to write under the
  Node install's own `lib/node_modules`, which would need root if Node lives under a
  root-owned path. Fix: set npm's global prefix to a `jenkins`-owned directory
  (`~jenkins/.npm-global`, via `.npmrc`) so `newman` installs without `sudo` and Ansible
  can manage it idempotently — the `community.general.npm` module supports `state:
  present` with a pinned `version:`, which is preferable to a hand-rolled shell task.
- Pin Newman's version explicitly; it's what actually executes the Postman collections,
  so an unpinned `npm install -g newman` (always-latest) is exactly the kind of drift this
  whole project exists to avoid.
- **Future option, not a change for now**: since you're already on rootless Podman, the
  official `postman/newman` container image is an alternative to installing Node/npm/
  Newman on the host at all — worth a look once Phase 1 is stable, not a blocker today.

## 8. Known technical debt inherited from the current fleet

Flagging these because they came up above; none block the RHEL 10 rollout, but they're
worth a backlog item each:

1. **Oracle JDK 8 licensing** — see §2. Likely fixable by swapping to a free OpenJDK 8
   build with no behaviour change for most projects.
2. **Nexus credentials embedded in `settings.xml`** — see §2.2. Fix ties naturally into
   Nexus Phase 2 (per-purpose service accounts, secrets injected per-build rather than
   baked into a file).
3. **`sshpass`** — you already flagged this yourself as a future improvement (replace
   password-based SSH in scripts with key-based auth per job/target). The role installs
   it as-is for parity with the current fleet; don't extend its use in new projects.

## 9. Open decision: firewalld posture on new nodes

The existing RHEL 9 workers run with no firewall enabled. [devops-policy.md
§8](devops-policy.md#8-other-server-setup-topics-to-standardise) treats `firewalld` as a
baseline `common`-role item. Two ways to reconcile that for the RHEL 10 fleet:

- **Enable it on the new nodes**, with the exact ports these tools need opened explicitly
  (SSH in from the controller; anything the job workloads themselves need to expose, e.g.
  a port a Testcontainers-based service binds for a test run) — a real hardening
  improvement, paid for by having to enumerate every port a build might need up front and
  keeping that list current as projects change.
- **Match the existing fleet** (leave it disabled) for consistency, and track "enable
  firewalld fleet-wide" as its own later hardening project covering RHEL 9 and RHEL 10
  together, rather than having the new nodes behave differently from the old ones.

**Decision:** match the existing fleet — `firewalld` stays disabled on the new RHEL 10
nodes for now (`firewalld_enabled: false` in the role). Track "enable firewalld
fleet-wide, RHEL 9 and RHEL 10 together" as its own hardening backlog item rather than
diverging the new nodes from the old ones.

---

## 10. Manual / prerequisite steps (not done by the worker playbook)

### Before the playbook

1. **Service-desk ticket** → DC provisions the RHEL 10 VM per
   [server-profiles.md](server-profiles.md): `/Dev_Data` (and
   `/Dev_Data/jenkins/containers` as its own volume) on expandable storage, `ansible`
   public key seeded, subscription attached, base repos reachable.
2. Confirm **forward + reverse DNS** between controller and worker, and **time sync**.
3. Add the host to `inventories/dev/`.

### Run

4. `ansible-playbook playbooks/common.yml -l <host>` then
   `ansible-playbook playbooks/jenkins_worker.yml -l <host>`.

### On the Jenkins controller (one-time per node)

5. **New Node → Permanent Agent**:
   - Remote root directory: `/Dev_Data/jenkins/agent`
   - Launch method: **Launch agents via SSH**
   - Host / port 22
   - Credentials: the **SSH private-key credential** whose public key the role installed
     in `~jenkins/.ssh/authorized_keys`
   - **Host Key Verification Strategy**: "Manually trusted key" (accept on first connect)
     **or** "Known hosts file" — in which case add the worker's host key to
     **`~jenkins/.ssh/known_hosts` on the controller** (`ssh-keyscan <worker>`).
   - Labels (e.g. `rhel10 maven java podman`), executor count.
6. Run a **smoke job**: `java -version` for each JDK, `mvn -v` for each Maven,
   `podman run --rm hello-world` as `jenkins`, plus the existing "CPU cores + memory"
   check.

Replace step 5 with **JCasC** (nodes defined in `jenkins.yaml`) or a small Ansible play
against the Jenkins REST API once you have more than two nodes — you will have six.

---

## 11. SSH keys / known_hosts — what's actually needed

| Direction | Needed? | Where |
|-----------|---------|-------|
| Controller → worker (agent launch) | **Yes** — `jenkins` `authorized_keys` on the worker gets the controller's public key. | worker (role does it) |
| Worker → controller | No (only for inbound/JNLP agents, which we're not using) | — |
| Controller's trust of the worker host key | Yes — via the node's Host Key Verification Strategy; if "Known hosts file", seed `known_hosts` **on the controller**. | controller |
| Worker → other systems inside jobs (git over SSH, deploys) | Only if such jobs exist — then seed `known_hosts` for `jenkins` and supply keys via **Jenkins credentials**, not baked into the node. | worker + Jenkins credentials |

Key point you half-remembered: the `known_hosts` concern is on the **controller**, about
the **worker's** host key. The worker itself needs `known_hosts` only when *jobs* SSH
outward. Java must already be present on the worker before the controller connects — the
role guarantees that.

---

## 12. Testing

Per your answer, testing uses **AlmaLinux / Rocky 10** cloud VMs (RHEL 10 rebuilds, no
subscription friction, behaviour effectively identical for this work), spun up only when
needed and torn down the same session.

### 12.1 Live-test results (Rocky Linux 10.2, GCP `e2-medium`)

The full `common` + `jenkins_worker` role pair was run end-to-end against a real
throwaway VM (real JDK/Maven/Flyway/Node/docker-compose downloads and checksums — SQLcl
skipped via `--tags`, since it has no public URL, see §2). Confirmed working, verified via
a genuine `ssh jenkins@host` login (the actual shape of a controller-launched SSH agent,
not just `sudo -u jenkins`): `java -version`, `mvn -v` with correct `JAVA_HOME`, `newman
--version`, `podman run --rm hello-world` under rootless Podman, `docker-compose
version`, the shared Podman network, and the `podman-prune.timer` active. A second,
identical run afterwards reported **zero changes** — the role is idempotent.

Five real bugs surfaced and were fixed as a direct result of this test (none of them
visible from `ansible-lint`, `--syntax-check`, or reading the role):

1. **`community.general.sefcontext` needs `python3-policycoreutils`** on the target (the
   `seobject` Python module) — added to `jenkins_worker_packages`.
2. **`become_user: jenkins` tasks (Podman, npm) need the `acl` package** — without it,
   Ansible's privilege-escalation temp-file handoff fails with a BSD-ACL-flavoured `chmod`
   error on Linux — added to `jenkins_worker_packages`.
3. **SELinux blocks sshd from reading `authorized_keys` when a user's home is outside
   `/home`** — `/Dev_Data` isn't covered by the system's default home-directory file
   contexts, so sshd logged `Could not open user 'jenkins' authorized keys ...
   Permission denied` and key-based login failed outright. Fixed by adding the same
   three-rule SELinux context pattern the default policy uses for `/home/<user>`
   (`user_home_dir_t` / `user_home_t` / `ssh_home_t`) targeted at `/Dev_Data/jenkins`
   instead — see `roles/jenkins_worker/tasks/user.yml`. This is a direct, concrete
   consequence of choosing `/Dev_Data` over `/home` for service-account homes (policy
   §2/§8's SELinux note) and is worth knowing before it surprises anyone doing this by
   hand on a real node.
4. **`getent subuid <user>` isn't a valid database on Rocky/RHEL 10** (`Unknown database:
   subuid`) — the subuid/subgid idempotency check always returned non-zero regardless of
   actual state, so `usermod --add-subuids` re-ran (and reported changed) on every single
   invocation. Fixed by checking `/etc/subuid` / `/etc/subgid` directly instead.
5. **The Maven `bin` directory was missing from `PATH`** in both
   `/etc/profile.d/jdk-default.sh` and `~jenkins/.ssh/environment` — a plain oversight
   when those were first written, caught immediately by `mvn: command not found` over a
   real `ssh jenkins@host` session.

One documentation correction came out of this too: an earlier draft of §4.1 claimed
`/etc/profile.d` never reaches the process a controller launches over SSH. Live testing
showed that's wrong — OpenSSH runs both interactive and command-mode sessions as **login
shells**, so `/etc/profile.d` fires normally over a real `ssh jenkins@host` connection.
The original failure that suggested otherwise came from testing via `sudo -u jenkins`
(Ansible's `become_user`), which does **not** invoke a login shell and sees neither
`/etc/profile.d` nor `~jenkins/.ssh/environment` — confirmed directly (`sudo -u jenkins
bash -c 'echo $PATH'` returns a bare `/sbin:/bin:/usr/sbin:/usr/bin`). Both mechanisms are
kept regardless — `/etc/profile.d` for interactive convenience, `~/.ssh/environment` as a
route that doesn't depend on the `.bash_profile`/`.bashrc` chain staying intact — but the
root-cause explanation is corrected here, and `playbooks/smoke_test.yml`'s Newman check
now sets `PATH` explicitly rather than assuming ambient dotfiles, since its own
`become_user`-based checks have exactly the same "no login shell" gap as the scenario
above.
