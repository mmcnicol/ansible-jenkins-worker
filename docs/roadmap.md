# Roadmap / Backlog

Status: **draft for team review**

Single monorepo, one inventory, roles per service, playbooks per service. Deliver in
phases; each phase is independently useful.

## Proposed repo layout

```
inventories/
  dev/
    hosts.yml
    group_vars/
    host_vars/
collections/requirements.yml      # community.general, ansible.posix,
                                  # community.crypto, prometheus.prometheus
roles/
  common/                         # baseline every host gets
  jenkins_worker/
  jenkins_controller/
  nexus/
  docker_registry/
  prometheus/
  metrics_longterm/
  grafana/
playbooks/
  site.yml
  common.yml
  jenkins_worker.yml
  ...
  patching.yml
docs/
```

---

## Phase 0 — Foundations

- Repo skeleton, `dev` inventory, `collections/requirements.yml`.
- **`common` role:** `ansible` + service users, scoped sudo, sshd hardening, chrony,
  firewalld baseline, disable unattended `dnf` updates, install `versionlock`, persistent
  journald, base packages, assert `/Dev_Data` layout, `node_exporter` (via
  `prometheus.prometheus`).
- CI: `yamllint`, `ansible-lint`, `--syntax-check`; Molecule optional.
- **Bootstrap runbook** (the one manual step to get `ansible` key access).

**Exit:** `ansible-playbook playbooks/common.yml` brings any fresh RHEL 10 host to a
known baseline.

---

## Phase 1 — Jenkins worker  *(immediate need — 6 nodes)*

- **`jenkins_worker` role:**
  - `jenkins` system user (locked password, `/bin/bash`, linger, subuid/subgid).
  - Controller's SSH public key in `~jenkins/.ssh/authorized_keys`.
  - **2 × OpenJDK** (Temurin tarballs, pinned + checksum) at `/Dev_Data/tools/jdk/<ver>`.
  - **2 × Maven** (Apache tarballs, pinned + checksum + PGP) at
    `/Dev_Data/tools/maven/<ver>`.
  - Managed Maven `settings.xml` (no credentials; Nexus URL via a variable).
  - **Rootless Podman**, graphroot on `/Dev_Data/jenkins/containers`, SELinux contexts.
  - **Prune** user systemd timer.
  - Example `dnf` package install (placeholder: `git` / `jq` — swap for the real one).
  - Host facts / labels for the controller.
- Test on AlmaLinux / Rocky 10 cloud VM(s).
- Document controller-side node creation (see
  [jenkins-worker-node.md](jenkins-worker-node.md)); JCasC/API automation as a
  fast-follow once > 2 nodes exist.
- Roll to the 6 nodes.

**Exit:** a new worker is inventory entry + one playbook run + one node definition on the
controller; smoke job green (`java -version` ×2, `mvn -v` ×2, `podman run` as `jenkins`).

---

## Phase 2 — Nexus  *(unblocks the internal artifact mirror for every later phase)*

- **`nexus` role:** tarball + systemd, `nexus` user, data on `/Dev_Data/sonatype-work`
  (own volume), TLS, **PostgreSQL** backend.
- **Config as code:** blob stores; repos (maven hosted/proxy/group, a `raw` repo for
  tool tarballs, docker repos if the registry is consolidated here); **two roles** — one
  for humans, one for CI; **scheduled tasks** — purge Maven snapshots after N days,
  compact the blob store.
- Repoint worker `settings.xml` and `artifact_base_url` at Nexus.

**Exit:** tool tarballs and Maven deps served from Nexus; scheduled maintenance running.

---

## Phase 3 — Jenkins controller

- **`jenkins_controller` role:** install (WAR + systemd, or pinned RPM), `JENKINS_HOME`
  on `/Dev_Data`, TLS, plugins pinned via `plugins.txt`, **JCasC** (security realm,
  agents, credential references, global tools matching the workers), `JENKINS_HOME`
  backup.
- Migrate worker node definitions into JCasC.

**Exit:** controller reproducible from code; nodes defined as code.

---

## Phase 4 — Metrics stack

- **`prometheus` role** (15-day) via `prometheus.prometheus`; scrape targets generated
  from inventory; alert rules (node down, disk %, especially `/Dev_Data`).
- **`grafana` role:** install, provisioned datasources, CPU/RAM dashboards as code.
- **Long-term tier:** VictoriaMetrics single-node; Prometheus `remote_write`; Grafana
  datasource for it. (Thanos instead only if object storage is available.)

**Exit:** one Grafana, two datasources (15-day + multi-year), dashboards from code.

---

## Phase 5 — Registry / consolidation

- Either a **`docker_registry` role** (`registry:2` via Quadlet, TLS, GC policy) **or**
  fold the registry into Nexus docker repos / evaluate Harbor.
- Decommission the container-hosted Nexus/registry; move to the managed installs above.

---

## Phase 6 — Ongoing operations

- **`patching` playbook** + maintenance-window process (batched, health-checked).
- Scheduled **drift detection** (`--check --diff`).
- Node onboarding / offboarding runbooks.
- Backup / restore **test** cadence.

---

## Notes on scope

- Every service above is a reasonable fit for Ansible **except** the items in
  [devops-policy.md §9](devops-policy.md#9-when-ansible-is-not-the-right-tool).
- Phases 2–5 are independent once Phase 0–1 land; reorder by business priority.
  Nexus-before-Jenkins-controller is recommended because the internal mirror de-risks
  every later download.
