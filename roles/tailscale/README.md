# tailscale

Thin wrapper around `artis3n.tailscale.machine` that registers the joined node's Tailscale IPv4 as the host fact `tailscale_ip`. The playbook's group_vars map `services_ip` from that fact (the role itself does not set `services_ip`).

## Requirements

- Ansible ≥ 2.15
- `artis3n.tailscale` collection (install via `requirements.yml` / `ansible-galaxy collection install artis3n.tailscale`)

## Role variables

| Variable            | Default   | Purpose                                              |
| ------------------- | --------- | ---------------------------------------------------- |
| `tailscale.authkey` | —         | Required. Pass via `secret.all.yaml`.                |
| `tailscale_up_args` | `"--ssh"` | Arguments passed to `tailscale up`.                  |

## Facts set

| Fact           | Purpose                                            |
| -------------- | -------------------------------------------------- |
| `tailscale_ip` | The node's Tailscale IPv4 (from the artis3n role). |

## Example

```yaml
- hosts: all
  roles:
    - role: tailscale
```

`tailscale.authkey` must be defined in group_vars (typically `group_vars/all/secret.all.yaml`).

## Dependencies

None (`dependencies: []` — the artis3n role is invoked via `include_role`, not as a meta dependency, so the authkey can be passed explicitly).

## License

MIT
