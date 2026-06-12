# proxmox-host

Configures a Proxmox VE host: powertop tuning, snippet/template directories, cloud-init `user-data` snippet, Ubuntu cloud image + LXC template downloads, CIFS mounts, and iGPU subgid mapping for unprivileged LXC passthrough.

## Requirements

- Ansible ≥ 2.15
- `ansible.posix` collection (CIFS mounts)
- Target host is a Proxmox VE node (Debian-based)

## Role variables

| Variable                     | Default                                                                       | Purpose                                            |
| ---------------------------- | ----------------------------------------------------------------------------- | -------------------------------------------------- |
| `ubuntu_lts_image_url`       | https://cloud-images.ubuntu.com/noble/current/noble-server-cloudimg-amd64.img | Ubuntu LTS cloud image URL                         |
| `ubuntu_cloudimg_filename`   | basename of the URL                                                            | Image filename (also used by `proxmox-vm`)         |
| `ubuntu_lxc_template`        | `ubuntu-24.04-standard_24.04-2_amd64.tar.zst`                                  | LXC template (must exist in `pveam available`)     |
| `cloud_init_keyboard_layout` | `be`                                                                           | XKB layout for the cloud-init snippet              |
| `cifs_mounts`                | `[]`                                                                           | List of `{ src, path, opts }` host-level mounts    |

External: the `igpu` tasks read `lxc_containers` (nodesetup group_vars) to detect unprivileged containers with `enable_igpu: true`.

## Facts exported

`igpu_video_gid` and `igpu_render_gid` (host `video`/`render` group GIDs) — consumed by the **proxmox-lxc** role's idmap block. Running `--tags proxmox-lxc` alone skips these; include the `igpu` tag when provisioning iGPU containers.

## Tags

- `setup`  — host setup only (powertop, dirs, downloads)
- `cifs`   — CIFS mounts only
- `igpu`   — iGPU subgid mapping only

## Example

```yaml
- hosts: proxmox
  roles:
    - role: proxmox-host
```

## Dependencies

None.

## License

MIT
