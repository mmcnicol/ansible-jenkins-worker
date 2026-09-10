# Dev Server Profiles

Status: **draft for team review**

Starting-point sizing for each server role. Tune against real usage once `node_exporter`
data exists. The **data centre team sizes the root partition (`/`)**; the platform team is
responsible for everything under `/Dev_Data`.

## Core principle

`/` is sized by the DC. Everything that grows lives under **`/Dev_Data`** on
independently-expandable storage:

- container / image stores
- build workspaces
- artifact and blob stores
- metric TSDBs
- large application logs

A full `/Dev_Data` must never prevent the host from booting or logging in. Where one
consumer could starve another, give it its **own volume**, not just its own folder.

## Profiles

| Profile | Role | vCPU | RAM | `/Dev_Data` (start) | Externally-mapped paths | Notes |
|---------|------|-----:|----:|--------------------:|-------------------------|-------|
| `worker-small` | Jenkins worker, light jobs | 4 | 8 GB | 100–200 GB | `/Dev_Data/jenkins` (home + workspace), `/Dev_Data/jenkins/containers` (rootless podman graphroot) | swap 2–4 GB |
| `worker-standard` | Jenkins worker, Maven/Java builds | 8 | 16–32 GB | 250–500 GB | as above | RAM for parallel Maven + Surefire; give build temp its own dir on `/Dev_Data` |
| `jenkins-controller` | Jenkins controller | 4–8 | 8–16 GB (heap 2–4 GB) | 100–250 GB | `/Dev_Data/jenkins_home` | prune build logs/artifacts; back up `jenkins_home` |
| `nexus` | Sonatype Nexus | 4–8 | 8–16 GB (heap 2–4 GB) | 500 GB – 2 TB+ | `/Dev_Data/nexus/sonatype-work` (own volume) | blob stores grow; keep ~20–30 % free for compaction |
| `registry` | OCI / Docker registry | 2–4 | 4–8 GB | 200 GB – 1 TB | `/Dev_Data/registry` | enable GC / retention policy |
| `prometheus-15d` | Prometheus, 15-day retention | 4 | 8–16 GB | 100–300 GB | `/Dev_Data/prometheus` | size ≈ bytes/sample × samples/s × 1.3 × retention_seconds |
| `metrics-longterm` | VictoriaMetrics (or Prometheus LT) | 4–8 | 16–32 GB | 1–4 TB (own volume) | `/Dev_Data/vm-data` | the one to give generous, separately-expandable storage |
| `grafana` | Grafana | 2 | 2–4 GB | 20–50 GB | `/Dev_Data/grafana` (only if SQLite) | prefer external PostgreSQL |

## Per-host checklist for the service-desk ticket

Ask the DC team to provision with:

- OS: RHEL 10.x
- `/` sized per their standard
- `/Dev_Data` on expandable storage at the size above
- For worker and Nexus: the large-growth path (`/Dev_Data/jenkins/containers`,
  `/Dev_Data/nexus/sonatype-work`) as its **own** volume
- The `ansible` user's SSH public key pre-seeded (or agree a bootstrap method)
- Subscription attached, base repos reachable
- Forward and reverse DNS registered
- NTP reachable
- Outbound proxy details (if applicable)
