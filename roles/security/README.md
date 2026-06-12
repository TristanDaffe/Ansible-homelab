# security

Bootstraps a hardened Linux host: package upgrades, non-root sudoer, SSH hardening, UFW firewall, fail2ban, optional NAS CIFS mount.

## Requirements

- Ansible ≥ 2.15
- Debian 12+ / Ubuntu 22.04+
- `community.general` and `ansible.posix` collections

## Role variables

See `defaults/main.yaml`. Common overrides:

| Variable                | Default               | Purpose                                                |
| ----------------------- | --------------------- | ------------------------------------------------------ |
| `security_updates_only` | `true`                | `apt upgrade=safe` instead of `dist`                   |
| `allow_reboot`          | `false`               | Reboot when `/var/run/reboot-required` is set          |
| `reboot_delay`          | `30`                  | Seconds before the controlled reboot                   |
| `post_reboot_delay`     | `60`                  | Seconds to wait after reboot before reconnecting       |
| `reboot_timeout`        | `600`                 | Max seconds to wait for the host to come back          |
| `extra_packages`        | `[]`                  | Extra apt packages to install on every host            |
| `mount_nas`             | `false`               | Whether to mount the `nas` CIFS share                  |
| `tailscale_network`     | `100.64.0.0/10`       | Source CIDR allowed to reach monitoring ports via UFW  |
| `docker_bridge_network` | `172.16.0.0/12`       | Source CIDR for containerized Prometheus → node_exporter (master) |

When `mount_nas: true`, a `nas` dict is required (see the commented example in `defaults/main.yaml`):

```yaml
nas:
  share: //192.168.0.1/shared   # CIFS share
  mount: /mnt/nas               # mountpoint
  puid: 1000                    # uid for the mount
  pgid: 1000                    # gid for the mount
```

The username, sudo group, and SSH key path are read from play-level vars (`username`, `sudo_group`, `ssh_group`, `ssh_public_key_file`) since they are shared with other roles.

The role also pre-creates the `docker` group (so the user's membership exists before the docker role installs Docker later in the play) and the firewall rules key off the `Monitoring`/`Master` inventory group names.

## Facts set

`upgrade_result`, `install_result`, and `reboot_required_file` registers are consumed by the `recap` role.

## Example

```yaml
- hosts: all
  roles:
    - role: security
      vars:
        extra_packages: [vim, htop]
        allow_reboot: true
```

## Dependencies

None.

## License

MIT
