# ansible-jenkins-worker

Ansible automation for platform dev servers, starting with Jenkins worker nodes on
RHEL 10.

> This repository is **public**. It contains no hostnames, credentials, internal URLs or
> employer-identifying detail. Environment-specific values live in a private inventory and
> the secret server.

## Status

Phase 1 prototype: `common` + `jenkins_worker` roles exist and pass
`ansible-lint` (production profile) and `--syntax-check`, but have **not yet
been run against a real host** — see [Testing](#testing) below. Every
version/URL/checksum in `roles/jenkins_worker/defaults/main.yml` is a
placeholder; override them in a private `group_vars`/`host_vars` file before
running for real (see `.gitignore`).

## Usage

```
ansible-galaxy collection install -r collections/requirements.yml

# One-time per new host, using whatever account the DC provisioned:
ansible-playbook playbooks/bootstrap.yml -l <host> -u <initial-admin> \
  --ask-pass --ask-become-pass -e "ansible_control_pubkey=$(cat ~/.ssh/id.pub)"

# Then, as the ansible user:
ansible-playbook playbooks/common.yml -l <host>
ansible-playbook playbooks/jenkins_worker.yml -l <host>
ansible-playbook playbooks/smoke_test.yml -l <host>
```

## Testing

Not yet run against a real RHEL 10 (or AlmaLinux/Rocky 10) host. `yamllint`,
`ansible-playbook --syntax-check` and `ansible-lint` all pass locally and in
CI (`.github/workflows/lint.yml`), but that doesn't catch runtime issues —
wrong module arguments, template logic, ordering. Validate on a real host
before pointing this at the actual fleet.

## Docs

| Doc | What it covers |
|-----|----------------|
| [docs/devops-policy.md](docs/devops-policy.md) | When to use dnf vs vendor repo vs tarball vs container; automatic-update policy; per-service upgrade guidance; long-term metrics; prune strategy; users/SSH/secrets/TLS; other server-setup topics; where Ansible is the wrong tool. |
| [docs/server-profiles.md](docs/server-profiles.md) | CPU / RAM / disk sizing per server role and which paths map to external `/Dev_Data` storage. |
| [docs/roadmap.md](docs/roadmap.md) | Monorepo layout and phased backlog: foundations → Jenkins worker → Nexus → Jenkins controller → metrics → registry → ongoing ops. |
| [docs/jenkins-worker-node.md](docs/jenkins-worker-node.md) | Design of the `jenkins_worker` role, the `jenkins` user, JDK/Maven install approach, rootless Podman, prune timer, and the manual controller-side steps. |
