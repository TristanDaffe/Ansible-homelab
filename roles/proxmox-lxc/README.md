# proxmox-lxc

Provisions a single Proxmox LXC container with optional Docker-friendly settings, TUN passthrough, Intel iGPU passthrough, and NAS bind mounts. Designed to be invoked from a loop in the playbook (one call per container definition).

## Requirements

- Ansible ≥ 2.15
- `community.proxmox` collection
- Proxmox API credentials in scope (`proxmox_api_host` + token or password)
- The `proxmox-host` role should run first: it downloads the LXC template, adds the subgid entries, **and exports the `igpu_video_gid`/`igpu_render_gid` facts the iGPU idmap block needs**. Running `--tags proxmox-lxc` alone breaks `enable_igpu` containers — include the host's `igpu` tag.

## Role variables

| Variable                 | Default                  | Purpose                                |
| ------------------------ | ------------------------ | -------------------------------------- |
| `proxmox_api_user`       | `root@pam`               | API user                               |
| `proxmox_bridge`         | `vmbr0`                  | Bridge for `net0`                      |
| `proxmox_validate_certs` | `false`                  | Verify the Proxmox API TLS certificate |
| `lxc_features`           | `[keyctl=1, nesting=1]`  | Features for containers without their own |
| `clear_pass`             | unset                    | Container root password; **omitted entirely when unset** (SSH-key-only access) |

Per-container config comes from the loop variable `ct` — full key list in `defaults/main.yaml`. Notable booleans:

- `privileged` — defaults to false (unprivileged container)
- `enable_tun` — bind-mount `/dev/net/tun`
- `enable_igpu` — Intel iGPU passthrough (unprivileged only)

## Behavior notes

- `state: present` only **creates**: changing `cores`/`memory`/`rootfs_gb` does not update an existing container (module limitation).
- `ct.ipconfig` must NOT carry an `ip=` prefix — the role adds it (`"192.168.1.10/24,gw=192.168.1.1"` or `"dhcp"`).

## Example

```yaml
tasks:
  - name: Provision each LXC
    ansible.builtin.include_role:
      name: proxmox-lxc
    loop: "{{ lxc_containers }}"
    loop_control:
      loop_var: ct
      label: "{{ ct.hostname }}"
```

## Dependencies

None.

## License

MIT
