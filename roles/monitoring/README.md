# monitoring

Installs Prometheus `node_exporter` and Grafana `Alloy` on a node. Alloy ships logs (journal, /var/log, Docker containers) to Loki on the master.

## Requirements

- Ansible ≥ 2.15
- `grafana.grafana` collection (Alloy)
- `services_ip` must be defined for every host — in this repo it maps to `tailscale_ip`, so the **tailscale role must run first**
- The Loki endpoint default reads `hostvars[groups['Master'][0]].services_ip` — an inventory group named `Master` must exist (override `alloy_loki_url` otherwise)

## Role variables

| Variable                       | Default                         | Purpose                                            |
| ------------------------------ | ------------------------------- | -------------------------------------------------- |
| `node_exporter_enabled`        | `true`                          | Install/run node_exporter (also the systemd enable flag) |
| `node_exporter_state`          | `started`                       | systemd service state                              |
| `node_exporter_restart`        | `on-failure`                    | systemd `Restart=` policy                          |
| `node_exporter_version`        | `latest`                        | Pin, or resolve the GitHub "latest" release        |
| `node_exporter_arch`           | `amd64`                         | Release artifact architecture                      |
| `node_exporter_bin_path`       | `/usr/local/bin/node_exporter`  | Install path                                       |
| `node_exporter_host`           | `{{ services_ip }}`             | Address Prometheus scrapes (kept in group_vars too — read cross-host via `hostvars`) |
| `node_exporter_listen_address` | `{{ node_exporter_host }}`      | Bind address; set to `""` on the master so the containerized Prometheus can reach it |
| `node_exporter_port`           | `9100`                          | Listen port                                        |
| `node_exporter_options`        | `''`                            | Extra CLI flags                                    |
| `node_exporter_uid` / `_gid`   | unset                           | Optional fixed UID/GID for the service user/group  |
| `node_alloy_enabled`           | `true`                          | Install/run Alloy                                  |
| `monitoring_alloy_version`     | `1.15.1`                        | Passed to `grafana.grafana.alloy` as `alloy_version` (pin avoids pre-release 404s) |
| `alloy_loki_url`               | master `services_ip`, port 3100 | Loki push endpoint                                 |

## Example

```yaml
- hosts: Monitoring
  roles:
    - role: monitoring
```

## Dependencies

None (the `grafana.grafana.alloy` role is invoked via `include_role`).

## License

MIT
