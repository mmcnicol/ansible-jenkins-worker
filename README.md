# ansible-jenkins-worker

Ansible automation for platform dev servers, starting with Jenkins worker nodes on
RHEL 10.

> This repository is **public**. It contains no hostnames, credentials, internal URLs or
> employer-identifying detail. Environment-specific values live in a private inventory and
> the secret server.

## Status

Advice / design phase. No playbooks yet — see the docs below.

## Docs

| Doc | What it covers |
|-----|----------------|
| [docs/devops-policy.md](docs/devops-policy.md) | When to use dnf vs vendor repo vs tarball vs container; automatic-update policy; per-service upgrade guidance; long-term metrics; prune strategy; users/SSH/secrets/TLS; other server-setup topics; where Ansible is the wrong tool. |
| [docs/server-profiles.md](docs/server-profiles.md) | CPU / RAM / disk sizing per server role and which paths map to external `/Dev_Data` storage. |
| [docs/roadmap.md](docs/roadmap.md) | Monorepo layout and phased backlog: foundations → Jenkins worker → Nexus → Jenkins controller → metrics → registry → ongoing ops. |
| [docs/jenkins-worker-node.md](docs/jenkins-worker-node.md) | Design of the `jenkins_worker` role, the `jenkins` user, JDK/Maven install approach, rootless Podman, prune timer, and the manual controller-side steps. |
