# opkssh-server

Installs OpenPubkey [opkssh](https://github.com/openpubkey/opkssh) on a host and wires it into `sshd_config` as the `AuthorizedKeysCommand` backend. Authorized OIDC users are listed in `opkssh_users`.

## Requirements

- Ansible ≥ 2.15
- The `security` role must have run first (creates the SSH-allowed group; opkssh requires public-key auth)

## Role variables

| Variable                   | Default  | Purpose                                                          |
| -------------------------- | -------- | ---------------------------------------------------------------- |
| `opkssh_install_ref`       | `main`   | Git ref of the install script. **Pin to a tag.**                 |
| `opkssh_install_checksum`  | unset    | `sha256:<hash>` to verify the downloaded install script          |
| `opkssh_users`             | `[]`     | List of `{ username, email, provider_url }` triples              |
| `opkssh_providers`         | —        | **Required.** List of `{ url, client_id, token_duration }` for `/etc/opk/providers`. Supply via `secret.all.yaml`. |

## Behavior notes

- The install is gated on opkssh being **absent**: changing `opkssh_install_ref` does not upgrade an existing install. Remove `/usr/local/bin/opkssh` to force a reinstall.
- `PasswordAuthentication no` is also enforced by the `security` role; both manage the same line in `/etc/ssh/sshd_config` with identical values.

## Example

```yaml
- hosts: all
  roles:
    - role: security
    - role: opkssh-server
      vars:
        opkssh_users:
          - { username: alice, email: alice@example.com, provider_url: https://accounts.google.com }
```

## Dependencies

None.

## License

MIT
