# DevOps Install & Operations Policy

Status: **draft for team review**
Scope: dev servers managed by the platform team (Jenkins workers, Jenkins controller,
Nexus, container registry, Prometheus, Grafana).

This document is deliberately generic. It contains no hostnames, credentials, internal
URLs or employer-identifying detail, because the repository is public. Environment-specific
values live in the private inventory / secret server, not here.

---

## 1. Guiding principles

1. **Config as code.** Every server is reproducible from this repo plus the private
   inventory. No undocumented manual changes. Drift is a bug.
2. **Pin everything.** Every tool and service has an explicit version recorded in
   `group_vars`. "Whatever the repo served today" is not a version.
3. **State lives on external storage.** Anything that grows without bound
   (container/image stores, build workspaces, artifact/blob stores, metric TSDBs, logs)
   lives under `/Dev_Data`, which is independently expandable. A full data volume must
   never stop a host from booting. See [server-profiles.md](server-profiles.md).
4. **Upgrades are deliberate events**, done in a maintenance window, preceded by a VM
   snapshot *and* a data backup, with a known rollback. Never automatic.
5. **One approach per service.** Don't run Nexus as a container on one box and a tarball
   on another. Consistency beats theoretical optimality.
6. **Least privilege.** Service accounts have no interactive password and only the access
   they need. Humans get named accounts. Break-glass passwords live in the secret server.

---

## 2. How to install a software tool or run a service

Five sanctioned approaches, roughly in order of "reach for this first":

| # | Approach | Use it for | Cost / caveats |
|---|----------|-----------|----------------|
| 1 | **dnf from RHEL BaseOS / AppStream** | OS utilities and dependencies: `git`, `jq`, `chrony`, `firewalld`, `podman`, `python3-*`, container tooling. Language runtimes when only the *major* version matters. | Red Hat controls the exact version and patch cadence. Fine for things you don't need to pin tightly. |
| 2 | **dnf from a vendor repo** (Jenkins, Grafana, PGDG, container tools upstream) | A service with a well-maintained repo where you want native systemd integration and do **not** need multiple versions side by side. | The package gets pulled in by `dnf upgrade` unless you stop it. **Must** `versionlock` it and keep the repo `enabled=0`, enabling it only for the explicit upgrade play. |
| 3 | **Vendor tarball / static binary → `/Dev_Data`, run via systemd** | Build tools that must match CI/prod exactly or run side by side (JDK 17 **and** 21, Maven 3.8 **and** 3.9). Stateful services on dedicated servers: Nexus, Prometheus, `node_exporter`, Grafana (optionally), Jenkins (WAR). | You own CVE tracking and patching for that component. Upgrade = lay down new release dir, swap a `current` symlink, restart. Rollback = swap back. |
| 4 | **Container image via rootless Podman + Quadlet + systemd** | Components shipped only as an image (`registry:2`), things that need clean isolation, or where the team already has strong container ops. | Needs image-registry access, rootless storage config, rootless networking quirks (ports < 1024, DNS). Backup = the bind-mounted data dir. **Never** use `:latest` — pin tag **and** digest. |
| 5 | **Language version managers** (sdkman, nvm, pyenv) | Developer laptops only. | **Not permitted on servers.** Not reproducible, not auditable. |

### 2.1 Decision guide

- Base OS utility or dependency? → **(1)**.
- Build tool that must exactly match prod, or needs multiple concurrent versions? →
  **(3)** tarball to `/Dev_Data/tools/...`, exposed to Jenkins as a named tool.
- Stateful service on its own dedicated server (Nexus, Jenkins, Grafana, Prometheus)? →
  **(3)** tarball + systemd + data dir on `/Dev_Data`. Use **(4)** instead only if the
  team has stronger container skills than sysadmin skills — then commit to it everywhere.
- Small / stateless-ish, or only shipped as an image (`registry:2`)? → **(4)**.
- Whatever you pick: pin the version in `group_vars`, verify checksum/GPG on download,
  exclude it from automatic updates, write down the upgrade and rollback steps.

### 2.2 Where do tarballs come from?

| Option | Verdict |
|--------|---------|
| Commit the tarball into this git repo | **No.** Bloats history, redistribution concerns, and the repo is public. |
| Control node holds the files, `get_url`/`copy` from there | Works offline, but the control node becomes a hard dependency and you push large files every run. |
| Target downloads straight from upstream (`get_url` with mandatory `checksum:`) | Acceptable for the prototype. Every node hits the internet; upstream outages and rate limits bite across 6+ nodes. |
| **Target downloads from an internal source of truth** (Nexus `raw` hosted repo, or a `raw` proxy of upstream) | **Target state.** One control point, checksummed, audit trail, survives upstream outages. |

Until Nexus exists: `get_url` from upstream **with a `checksum:` that is always set**, and a
single `artifact_base_url` variable so the switch to Nexus later is a one-line change.
This is one reason Nexus is early in the [roadmap](roadmap.md).

---

## 3. Automatic OS updates

We have been burned by unattended `dnf` updates. Policy:

- **Disable unattended upgrades.** Mask `dnf-automatic.timer` and `packagekit` if present.
  (RHEL 10 does not enable `dnf-automatic` by default — confirm per host anyway.)
- **Patch on a schedule** via a dedicated `patching` playbook, run in a maintenance
  window (monthly baseline), nodes batched (`serial:`), with a health check after each
  batch and automatic stop on failure.
- **`versionlock`** (`python3-dnf-plugin-versionlock`) on: `podman`, `container-selinux`,
  any vendor-repo application packages (`jenkins`, `grafana`), and packaged JDKs if used.
- **Vendor repos stay `enabled=0`.** Enable per-command (`--enablerepo=`) only for the
  explicit, reviewed upgrade of that one component.
- **Security errata** get a faster, separate cadence: review `dnf updateinfo`, apply
  `dnf upgrade --security` out of band when severity warrants.
- **Kernel updates** imply a reboot → maintenance window only. Use `needs-restarting -r`
  to detect when a reboot is outstanding.

---

## 4. Upgrade guidance

### 4.1 General procedure (all services)

1. Announce the window. Confirm no critical jobs/builds in flight.
2. **VM snapshot** (DC team, or cloud API). Snapshot ≠ backup — also take a **data
   backup** of the app's data dir.
3. Read the vendor's upgrade notes for that specific version jump (DB migrations,
   breaking config changes, minimum-version stepping stones).
4. Apply per the per-approach method below.
5. Verify: service up, version correct, a functional smoke test passes.
6. Keep the previous release available for rollback for at least one window.

### 4.2 By install approach

- **Tarball + systemd:** lay down `/Dev_Data/<app>/releases/<version>` alongside the
  current one; `stop`; repoint the `current` symlink; run any migration; `start`; verify;
  keep the old release dir. Rollback = repoint the symlink, `start`. Data dir is external
  and untouched.
- **Container + Quadlet:** pin image by tag **and** digest in `group_vars`; `podman pull`
  the new digest; update the `.container` unit; `systemctl daemon-reload`; restart.
  Bind-mounted data on `/Dev_Data` persists. Rollback = previous digest.
- **dnf vendor package:** snapshot; `dnf upgrade --enablerepo=<vendor> <pkg>`; re-apply
  `versionlock` at the new version; verify. Rollback = `dnf downgrade` (only works if the
  old RPM is still cached/mirrored — keep a copy) then re-lock.

### 4.3 By service

| Service | Recommended install | Upgrade notes |
|---------|--------------------|---------------|
| **Nexus** | Tarball + systemd, `nexus` service user, app at `/Dev_Data/nexus/releases/<ver>` via `current` symlink, data at `/Dev_Data/nexus/sonatype-work` (own volume, backed up). Move off the embedded DB to **PostgreSQL**. | Read Sonatype's upgrade matrix — there are DB-format migrations between some majors. Snapshot, stop, swap symlink, start, watch the log through the migration. If currently a container with a mapped volume: you can keep that and upgrade by bumping the pinned digest; migrating container→tarball is "stop container, point a fresh tarball install at the same `sonatype-work`, start" — rehearse on a clone first. |
| **Jenkins controller** | WAR on `/Dev_Data` + systemd **or** the official RPM. RPM is well made and acceptable, but it puts `JENKINS_HOME` at `/var/lib/jenkins` — override to `/Dev_Data/jenkins_home` — and it **will** appear in `dnf upgrade`, so `versionlock` it. Stay on the **LTS** line. | Plugins are the real risk, not the core. Pin plugin versions in a `plugins.txt` lockfile, test upgrades on a staging instance, drive config with **JCasC**. WAR gives the cleanest rollback (keep the old `.war`). |
| **Container / OCI registry** | `registry:2` (CNCF Distribution) as rootless Podman + Quadlet, bind mount `/Dev_Data/registry`, TLS from day one. | Drop-in binary/image upgrades; blob format is stable. Configure a GC / retention policy early. For auth, UI, scanning, replication, evaluate **Harbor** or **Nexus Docker repos** rather than running bare `registry:2` long term. |
| **Prometheus / node_exporter** | Tarball + systemd via the `prometheus.prometheus` Ansible collection. Data at `/Dev_Data/prometheus`. | Usually a drop-in binary swap; check release notes for the rare TSDB bump. Keep `node_exporter` version identical fleet-wide (a role var). |
| **Grafana** | RPM from Grafana's repo (`enabled=0`, `versionlock`) or container. Provision datasources + dashboards **as code**. Move the DB off SQLite to PostgreSQL; if staying on SQLite, put `/var/lib/grafana` on `/Dev_Data`. | Snapshot + DB backup, `dnf upgrade --enablerepo=grafana grafana`, check dashboards and datasources render. |

### 4.4 Long-term metrics retention (3–5 years)

A single vanilla Prometheus with `--storage.tsdb.retention.time=5y` is **not** a sound
long-term store: no downsampling (disk grows linearly forever), heavy compaction and slow
restarts as the TSDB grows, and a single local-disk SPOF. Options, best first for a small
team:

1. **VictoriaMetrics (single-node binary)** as the long-term backend. Prometheus
   `remote_write`s to it; set `-retentionPeriod=5y`; far better compression; PromQL-ish
   query endpoint; simple ops. Lowest effort for multi-year.
2. **Thanos** (sidecar + store + compactor + object storage) — gives downsampling and
   cheap object storage, more moving parts. Worth it if S3-compatible storage exists.
3. **Grafana Mimir** — more scale than needed here.
4. **Pure Prometheus, accepted risks:** dedicated instance, `retention.time=5y` **and**
   `retention.size` as a hard cap, a large dedicated `/Dev_Data` volume, a longer scrape
   interval (60–120 s), and ideally only a curated subset of series (via federation or
   recording rules from the 15-day instance). Monitor disk aggressively.

Recommendation: ship the 15-day Prometheus first, add VictoriaMetrics as the long-term
tier in the same phase, point Grafana at both.

---

## 5. Container image cleanup (prune)

Current state: a monthly Jenkins job runs purge commands on each server.

**Preferred: a per-host systemd timer, managed by Ansible.** For rootless Podman under
`jenkins`, a **user** timer (`systemctl --user`, linger enabled) running e.g.:

```
podman system prune -f --filter until=720h
podman volume prune -f
```

Why this beats the Jenkins job:

- Runs locally even when the controller is down.
- Logged in `journalctl --user` on the host itself.
- Added automatically whenever the role is applied to a new node — nothing to remember.
- Tunable per host via `group_vars`.

Notes:

- Prefer `--filter until=` over blanket `-a`; keep recent layers and base images you pull
  every build (blowing them away wastes bandwidth on the next build).
- Optionally guard the timer against running mid-build (check for running containers /
  a lock file).
- Cron also works, but for a rootless account without an active session a `--user`
  systemd timer with linger is the native RHEL 10 fit.
- Keep this **local**. Don't centralise what can run on the host.

---

## 6. Scheduled Ansible runs (patching, drift checks)

For **fleet-wide** scheduled Ansible (patching, drift detection, any cross-host task):
run it from a **Jenkins pipeline job on a schedule**, with this repo as source and
credentials from Jenkins. Reasons: audit trail, retained logs, RBAC, notifications, and
no "which laptop is the control node" problem. Cron on a dedicated, itself-managed control
node is acceptable only as a fallback. If this grows, evaluate AWX / Ansible Automation
Platform as a proper control plane.

For **per-host recurring** tasks (podman prune, log trims): keep them **local** as systemd
timers laid down by the role.

---

## 7. Users, SSH and secrets

- **`ansible` user** on every host: in a dedicated group, scoped or `NOPASSWD` sudo,
  **SSH key-only**. The interactive password exists only in the secret server for humans /
  break-glass. Ansible never uses the password.
- **Bootstrap** (the one unavoidable manual step): the DC provisioning process seeds the
  `ansible` public key (ask for this in the service-desk template), *or* a one-time
  `ssh-copy-id` using the secret-server password, *or* cloud-init / kickstart.
- **`jenkins` user** on workers: system account, **password locked** (`!` in shadow),
  **SSH key-only**, no console access. Shell is `/bin/bash` because the SSH-launched agent
  needs to execute a shell and rootless Podman + `systemctl --user` are far smoother with
  a real login shell + linger. "No password, key-only, no console" is what the security
  requirement means in practice. If workers later move to **inbound/JNLP** agents running
  as a system service, switch the shell to `/sbin/nologin`. See
  [jenkins-worker-node.md](jenkins-worker-node.md).
- **sshd hardening** (a `common` role): `PasswordAuthentication no`, `PermitRootLogin no`,
  `AllowGroups`. Roll out carefully on one host first so you don't lock yourself out.
- **Secrets do not live in this repo.** Pull them from the secret server at run time, or
  pass them via `--extra-vars` from Jenkins credentials. Ansible Vault is a stopgap only —
  and in a public repo one weak passphrase = exposure. Prefer the secret server you
  already run; integrate, don't duplicate.
- **TLS everywhere.** Every web service behind TLS with the internal CA, renewal
  automated. Terminate either at the app (Jenkins `--httpsPort`, Nexus jetty-https,
  registry `tls:`) or at a shared reverse proxy (nginx / Caddy / HAProxy) — a proxy is
  usually easier for central cert management and adding auth headers. The registry
  **must** be TLS: Podman/Docker refuse plain-HTTP registries unless explicitly
  whitelisted.

---

## 8. Other server-setup topics to standardise

Covered by the `common` role or documented runbooks:

- **Time sync** (chrony) — mandatory for TLS, Jenkins, and metric timestamps.
- **SELinux** stays `enforcing`. Budget time for `semanage fcontext` on `/Dev_Data`
  service paths and rootless-podman storage; record every context you set.
- **firewalld** — explicit per-profile port lists, nothing implicit.
- **Logging** — persistent journald with size caps; logrotate for app logs under
  `/Dev_Data`.
- **Backups** — what is backed up, how, and a **restore test** cadence. A snapshot is not
  a backup.
- **Monitoring the monitors** — `node_exporter` on every host + a basic alert on disk
  usage, especially `/Dev_Data`.
- **Filesystem** — XFS (RHEL default) for large data volumes; note XFS cannot shrink, so
  size via LVM and grow as needed.
- **Outbound proxy / package-repo strategy** (Satellite? direct CDN? offline mirror?) —
  this materially affects section 2. Decide and document.
- **Locale / timezone** consistent fleet-wide.
- **`/tmp` on build nodes** — build temp can be large; consider a dedicated dir on
  `/Dev_Data` or a sized tmpfs.
- **Naming & inventory conventions.**
- **Drift detection** — run Ansible `--check --diff` on a schedule and alert on changes.
- **Node onboarding / offboarding runbooks.**
- **CIS / STIG baseline** — if the org requires it, layer it into `common` early rather
  than retrofitting.

---

## 9. When Ansible is *not* the right tool

- **VM provisioning and snapshots** — DC team / cloud API. Ansible can *call* a cloud
  module but doesn't own the lifecycle.
- **Jenkins plugin dependency resolution** — use Jenkins' plugin manager / the
  plugin-installation-manager-tool with a lockfile.
- **Nexus / Jenkins internal data migrations during major upgrades** — vendor tooling
  does the migration; Ansible only orchestrates stop/snapshot/swap/start.
- **Grafana dashboard authoring** — build in the UI, export JSON, commit it, provision it.
  Don't hand-write dashboards.
- **Prometheus long-term storage** — use a purpose-built store (section 4.4).
- **Secret management** — the secret server, not Vault-in-repo.
