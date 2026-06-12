# recap

Prints a per-host deployment summary table at the end of a play. Reads facts registered by earlier roles (`tailscale_ip`, `docker_version`, `upgrade_result`, `install_result`, `reboot_required_file`).

## Requirements

- Run after `security` (registers `upgrade_result`, `install_result`, `reboot_required_file`)
- Optional after `docker` (`docker_version`) and `tailscale` (`tailscale_ip`)

## Role variables

None overridable.

## Example

```yaml
- hosts: all
  roles:
    - role: security
    - role: tailscale
    - role: docker
    - role: recap
```
