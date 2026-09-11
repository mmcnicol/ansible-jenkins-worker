# ansible-jenkins-worker

Ansible automation for platform dev servers, starting with Jenkins worker nodes on
RHEL 10.

> This repository is **public**. It contains no hostnames, credentials, internal URLs or
> employer-identifying detail. Environment-specific values live in a private inventory and
> the secret server.

## Status

Phase 1 prototype: `common` + `jenkins_worker` roles pass `ansible-lint`
(production profile) and `--syntax-check`, and have been run end-to-end
against a real throwaway RHEL-10-family VM (Rocky Linux 10.2) with real
JDK/Maven/Flyway/Node/docker-compose downloads — idempotent on a repeat run,
verified via a genuine `ssh jenkins@host` login. SQLcl wasn't exercised (no
public download URL — see the role defaults). Five real bugs were found and
fixed this way; see [docs/jenkins-worker-node.md §12](docs/jenkins-worker-node.md#12-testing)
for what they were. Every version/URL/checksum in
`roles/jenkins_worker/defaults/main.yml` is still a placeholder; override
them in a private `group_vars`/`host_vars` file before running for real (see
`.gitignore`).

## Usage

```
ansible-galaxy collection install -r collections/requirements.yml

# One-time per new host, using whatever account the DC provisioned. Use a
# --extra-vars FILE, not inline -e "key=$(cat ...)" — Ansible's inline -e
# parser splits on whitespace and will truncate the key (see the comment at
# the top of bootstrap.yml).
echo "ansible_control_pubkey: $(cat ~/.ssh/id.pub)" > /tmp/bootstrap-vars.yml
ansible-playbook playbooks/bootstrap.yml -l <host> -u <initial-admin> \
  --ask-pass --ask-become-pass --extra-vars @/tmp/bootstrap-vars.yml

# Then, as the ansible user:
ansible-playbook playbooks/common.yml -l <host>
ansible-playbook playbooks/jenkins_worker.yml -l <host>
ansible-playbook playbooks/smoke_test.yml -l <host>
```

## Testing

`yamllint`, `ansible-playbook --syntax-check` and `ansible-lint` all pass
locally and in CI (`.github/workflows/lint.yml`). Beyond that, the role has
been run end-to-end on a real Rocky Linux 10.2 VM — see
[docs/jenkins-worker-node.md §12](docs/jenkins-worker-node.md#12-testing) for
the results and the bugs that live testing caught (lint alone would not
have). It has **not** been run against the actual work fleet — do that with
real (not placeholder) versions/checksums before trusting it there.

## Docs

| Doc | What it covers |
|-----|----------------|
| [docs/devops-policy.md](docs/devops-policy.md) | When to use dnf vs vendor repo vs tarball vs container; automatic-update policy; per-service upgrade guidance; long-term metrics; prune strategy; users/SSH/secrets/TLS; other server-setup topics; where Ansible is the wrong tool. |
| [docs/server-profiles.md](docs/server-profiles.md) | CPU / RAM / disk sizing per server role and which paths map to external `/Dev_Data` storage. |
| [docs/roadmap.md](docs/roadmap.md) | Monorepo layout and phased backlog: foundations → Jenkins worker → Nexus → Jenkins controller → metrics → registry → ongoing ops. |
| [docs/jenkins-worker-node.md](docs/jenkins-worker-node.md) | Design of the `jenkins_worker` role, the `jenkins` user, JDK/Maven install approach, rootless Podman, prune timer, and the manual controller-side steps. |
