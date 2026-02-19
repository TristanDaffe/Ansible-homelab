# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This is a HomeLab Ansible monorepo with three active projects, each a self-contained Ansible workspace (own `ansible.cfg`, `inventory`, `group_vars/`, and `roles/`):

| Directory | Purpose |
|-----------|---------|
| `NodeSetup/` | Provision VMs and LXC containers on a Proxmox host |
| `NewStart/` | Configure provisioned nodes (Docker, Tailscale, security, monitoring) |
| `Swarm/` | Docker Swarm + GlusterFS cluster (current production, being migrated to NewStart) |

`HomeLab/` is legacy and not in scope.

## Architecture

### NewStart (active project)

**Execution order** (defined in `run.yaml`):
1. `Security` — creates non-root user (`susu`), SSH hardening, UFW firewall, fail2ban
2. `Tailscale` — VPN mesh via `artis3n.tailscale` collection
3. `Monitoring` — node_exporter + promtail (only for hosts in the `Monitoring` group)
4. `Docker` — installs Docker Engine
5. `Deploy-composes` — renders Jinja2 compose templates and deploys them with `docker compose`
6. `Recap` — prints a summary of deployed services
7. `docker-maintenance` — prunes images/containers

**Variable hierarchy:**
- `group_vars/all/all.yaml` — shared defaults (username, timezone, Docker dir, monitoring versions)
- `group_vars/all/secret.all.yaml` — secrets (git-ignored via `secret.*` pattern)
- `group_vars/<GroupName>.yaml` — per-node overrides; the `docker.composes` list defines which compose stacks are deployed on that node

**Deploy-composes role pattern:**
The `docker.composes` list in group_vars drives deployment. Each entry maps to a Jinja2 template in `roles/deploy-composes/templates/` and is deployed to `{{ docker.docker_dir }}/compose/<name>/compose.yaml` on the target host. Config files (e.g. `traefik.yaml`, `prometheus.yaml.j2`) are templated separately into subdirectories under `docker_dir`.

**Inventory groups** (`NewStart/inventory`):
- `Vm` — Ubuntu cloud-init VMs (SSH as `susu`)
- `Lxc` — LXC containers (SSH as `root`)
- `Monitoring` — hosts that get monitoring stack; `MonitoringMaster` sub-group gets Prometheus+Loki
- `Tailscale` — all nodes joined to the VPN mesh

### NodeSetup

**Proxmox host is on version 9.1.5+.** Do not suggest workarounds for bugs fixed in that release (e.g. the `ip_unprivileged_port_start` sysctl issue with Docker in LXC was a known Proxmox bug fixed in 9.1.5).

`setup.yaml` targets the `proxmox` host group and:
1. Configures the Proxmox host (powertop auto-tune, directory structure)
2. Downloads Ubuntu LTS cloud image and LXC template if missing
3. Loops over `vms` variable → calls `provision_vm.yaml` (uses `community.proxmox.proxmox_kvm`)
4. Loops over `lxc_containers` variable → calls `provision_lxc.yaml` (uses `community.proxmox.proxmox`)

VMs and containers are defined in group_vars for the `proxmox` group. The Proxmox API credentials (`proxmox_api_host`, `proxmox_api_token_id`, `proxmox_api_token_secret`) must be defined — typically in a `secret.*` vars file.

LXC containers get Docker-friendly kernel settings written to `/etc/pve/lxc/<ctid>.conf` automatically based on the `ct.privileged` flag.

**LXC device passthrough pattern** (`provision_lxc.yaml`):
Each passthrough feature is a `block:` with the `when:` condition at block level (not repeated per task). Current features:
- `enable_tun` — TUN device for VPN (works on both privileged and unprivileged)
- `enable_igpu` — Intel iGPU passthrough, **unprivileged containers only**

**Intel iGPU passthrough for unprivileged LXC:**
- Host side (`setup.yaml`): detects `video` (GID 44) and `render` (GID 104) group GIDs dynamically, adds them to `/etc/subgid` so root can map those GIDs into the container
- LXC config: `lxc.cgroup2.devices.allow: c 226:* rwm`, bind-mount `/dev/dri`, and a full `lxc.idmap` block that punches holes for GID 44 and 104 so device ownership works inside the container
- Inside the container: install `intel-media-va-driver-non-free`, add app user to `render` group (GID 104), set `LIBVA_DRIVER_NAME=iHD`
- `intel_gpu_top` and `vainfo` are **not reliable test tools inside LXC** — verify by checking actual application behavior (e.g. Jellyfin dashboard showing VAAPI transcoding)

## Secrets

Files matching `secret.*` are git-ignored. Sensitive variables (API tokens, passwords, Tailscale auth keys) belong in `group_vars/<group>/secret.<group>.yaml` or `group_vars/all/secret.all.yaml`.
