# proxmox-vm

Provisions a single Proxmox VM from an Ubuntu cloud image. Designed to be invoked from a loop in the playbook (one call per VM definition).

## Requirements

- Ansible ≥ 2.15
- `community.proxmox` collection
- Proxmox API credentials (`proxmox_api_host` + `proxmox_api_token_id`/`proxmox_api_token_secret`, or `proxmox_api_password`) in scope
- The `proxmox-host` role should run first (snippets dir, cloud image, `ubuntu_cloudimg_filename` default)

## Role variables

| Variable                 | Default    | Purpose                                  |
| ------------------------ | ---------- | ---------------------------------------- |
| `proxmox_api_user`       | `root@pam` | API user                                 |
| `proxmox_bridge`         | `vmbr0`    | Bridge for `net0`                        |
| `proxmox_validate_certs` | `false`    | Verify the Proxmox API TLS certificate   |

Per-VM config comes from the loop variable `vm`; required play vars (`proxmox_node`, `proxmox_storage`, `proxmox_snippets_storage`, API credentials) and the full `vm` key list are documented in `defaults/main.yaml`.

## Behavior notes

- `disk_gb` only **grows** the disk (Proxmox limitation); the resize is skipped when scsi0 is already at or above the requested size.
- The `qm` tasks run on the Proxmox host itself; the API tasks are delegated to localhost.

## Example

```yaml
tasks:
  - name: Provision each VM
    ansible.builtin.include_role:
      name: proxmox-vm
    loop: "{{ vms }}"
    loop_control:
      loop_var: vm
      label: "{{ vm.name }}"
```

## Dependencies

None.

## License

MIT
