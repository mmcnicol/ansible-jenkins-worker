# Jenkins Worker Node on RHEL 10 — design notes

Status: **draft for team review**
Connection method assumed: **SSH, controller-initiated** ("Launch agents via SSH").

---

## 1. What the `jenkins_worker` role will do

1. Create the `jenkins` system user (see §3).
2. Install **2 × OpenJDK** and **2 × Maven** as pinned tarballs under `/Dev_Data/tools`
   (see §2).
3. Deploy a managed Maven `settings.xml` (no credentials in it).
4. Install and configure **rootless Podman** for `jenkins`, storage on `/Dev_Data`
   (see §4).
5. Lay down the **prune** user systemd timer (see §5).
6. Install one or two packages via `dnf` — example placeholder, swap for the real tool
   (see §6).
7. Publish host facts/labels so the controller knows what the node offers.

Controller-side node creation is **not** done by this role (see §7).

---

## 2. JDK and Maven — install approach

**Recommendation: pinned vendor tarballs to `/Dev_Data/tools`, exposed to Jenkins as named
tools.** Rationale:

- You explicitly want *specific* versions, and two of each **side by side** — `alternatives`
  juggling and "whatever AppStream ships this month" both fight that.
- Decouples the workers from the OS update cadence.
- This is exactly how Jenkins' own tool installers work; we're just pre-staging them.

| Tool | Source | Path | Verify |
|------|--------|------|--------|
| OpenJDK ×2 | **Adoptium Temurin** tarballs (e.g. 17.0.x, 21.0.x — exact builds set in `group_vars`) | `/Dev_Data/tools/jdk/temurin-<version>` | SHA-256 checksum (mandatory) |
| Maven ×2 | **Apache Maven** binary tarballs (e.g. 3.8.8 and 3.9.x) | `/Dev_Data/tools/maven/apache-maven-<version>` | SHA-512 checksum + PGP signature |

- Keep a **packaged** JDK (`dnf install java-21-openjdk-headless`) as the OS default for
  system tooling; the Temurin builds are for *builds*, selected per job.
- Do **not** put either Maven ambiguously on the global `PATH`. Pick one default via a
  `profile.d` script if you want a shell default; let Jenkins select per job otherwise.
- Download source: upstream now (with `checksum:` always set) via an `artifact_base_url`
  variable; flip that to the Nexus `raw` repo in Phase 2.

In **Jenkins → Manage Jenkins → Tools**, add fixed entries (install automatically = off):

- `jdk-17` → `/Dev_Data/tools/jdk/temurin-17.0.x`
- `jdk-21` → `/Dev_Data/tools/jdk/temurin-21.0.x`
- `maven-3.8` → `/Dev_Data/tools/maven/apache-maven-3.8.8`
- `maven-3.9` → `/Dev_Data/tools/maven/apache-maven-3.9.x`

Jobs then request the tool they need. Because every worker uses the same paths, this
config is identical across the fleet and belongs in JCasC (Phase 3).

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

## 6. The `dnf` example

The role installs a short `worker_packages` list via `ansible.builtin.dnf`. Ships with a
harmless placeholder (`git`, `jq`) — replace with the one or two tools you actually
install this way. This is the sanctioned use of approach (1) in the policy: OS utilities,
not services.

---

## 7. Manual / prerequisite steps (not done by the worker playbook)

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

## 8. SSH keys / known_hosts — what's actually needed

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

## 9. Testing

Per your answer, testing will use **AlmaLinux / Rocky 10** cloud VMs (RHEL 10 rebuilds, no
subscription friction, behaviour effectively identical for this work). Spun up only when
role development starts, torn down the same session. Not created now — this phase is
advice only.
