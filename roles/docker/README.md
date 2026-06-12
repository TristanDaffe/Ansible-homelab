# docker

Installs Docker Engine and the Compose plugin from the official Docker apt repository. Optionally installs the standalone `docker-compose` binary as a fallback.

## Requirements

- Ansible ≥ 2.15
- Ubuntu 22.04+ (Jammy/Noble)

## Role variables

| Variable                             | Default   | Purpose                                                     |
| ------------------------------------ | --------- | ----------------------------------------------------------- |
| `docker_install_compose_standalone`  | `false`   | Also drop `/usr/local/bin/docker-compose` (legacy fallback) |
| `docker_compose_version`             | `v2.32.1` | Standalone binary version (first install only — see defaults) |
| `docker_logrotate_rotate`            | `7`       | Container log rotations kept                                |
| `docker_logrotate_size`              | `10M`     | Rotate container logs beyond this size                      |

External (play-level) variables:

| Variable   | Purpose                                                                   |
| ---------- | ------------------------------------------------------------------------- |
| `username` | User added to the `docker` group. Skipped when `root` (LXC containers).   |

## Facts set

| Register         | Purpose                                              |
| ---------------- | ---------------------------------------------------- |
| `docker_version` | `docker --version` output, consumed by the `recap` role. |

## Example

```yaml
- hosts: docker_hosts
  roles:
    - role: docker
```

## Dependencies

None.

## License

MIT
