# docker-maintenance

**Destructive.** Optional Docker prune tasks. The role is gated behind `tags: [never, docker-maintenance]`, so it only runs when explicitly requested with `--tags docker-maintenance`. All prune toggles default to `false` in the role — but note this repo's `configure/group_vars/all/all.yaml` sets `prune_images: true`, so a plain `--tags docker-maintenance` run **will** prune dangling images.

## Requirements

- `community.docker` collection
- Docker SDK for Python on the target hosts (`python3-docker`, installed by the `docker` role)

## Role variables

| Variable                       | Default | Purpose                                                       |
| ------------------------------ | ------- | ------------------------------------------------------------- |
| `prune_containers`             | `false` | Remove stopped containers older than `prune_containers_older_than` |
| `prune_images`                 | `false` | Remove **dangling** (untagged) images. Never `--all`.         |
| `prune_volumes`                | `false` | Remove unused volumes. **Wipes any unattached named volume.** |
| `prune_containers_older_than`  | `24h`   | `until` filter for container prune                            |
| `prune_images_older_than`      | `168h`  | `until` filter for image prune (volumes have no `until` — module limitation) |

## Example

```bash
ansible-playbook -i configure/inventory configure/configure.yaml \
  --tags docker-maintenance -e prune_containers=true -e prune_images=true
```

## Dependencies

None.

## License

MIT
