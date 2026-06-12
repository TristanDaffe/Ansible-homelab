# deploy-composes

Renders Jinja2 Docker Compose templates from `templates/` and brings them up via `community.docker.docker_compose_v2`. The list of stacks to deploy is driven by the consumer-supplied `docker.composes` group_var.

## Requirements

- Ansible ≥ 2.15
- `community.docker` collection
- The `docker` role (or equivalent) must have run first

## Role variables

| Variable                       | Default     | Purpose                                            |
| ------------------------------ | ----------- | -------------------------------------------------- |
| `deploy_composes_pull_policy`  | `missing`   | `docker_compose_v2 pull` mode (`missing` / `always`) |

External (consumer-supplied) variables:

| Variable                 | Purpose                                                              |
| ------------------------ | --------------------------------------------------------------------- |
| `username`               | Owner of the created dirs and rendered files                          |
| `docker.docker_dir`      | Root dir for `compose/` and `volumes/` trees                          |
| `docker.docker_networks` | External Docker networks to create (default `[]`)                     |
| `docker.composes`        | List of stacks to deploy (shape below)                                |
| `docker.cleanup`         | `true` to **remove every container** not belonging to a deployed stack (default `false`) |

Shape of a `docker.composes` entry:

```yaml
- name: traefik                      # project name; template defaults to <name>.yaml
  compose: network.yaml              # optional: template filename override
  volumes:                           # optional: dirs created under docker_dir/volumes/
    - letsencrypt                    # plain string, owned by username
    - path: pocket-id                # or a dict with fixed ownership
      uid: 65532
      gid: 65532
  config:                            # optional: files templated under docker_dir/volumes/
    - file: traefik.yaml             # source under templates/config/
      dst: letsencrypt/traefik.yaml  # destination relative to docker_dir/volumes/
      force: false                   # optional: don't overwrite an existing file (default true)
```

Volume dirs are **create-only**: ownership is set on first creation and never corrected afterwards, because many containers chown their own data dirs at runtime.

The compose templates themselves consume many deployment-specific vars (`timezone`, `services_ip`, `nas.*`, `traefik_peers`, app secrets, …) — all defined in `configure/group_vars/all/all.yaml` and `secret.all.yaml`.

## Example

```yaml
docker:
  docker_dir: /docker
  docker_networks: [proxy_network]
  cleanup: true
  composes:
    - name: traefik
      compose: network.yaml
      volumes: [letsencrypt]
      config:
        - { file: traefik.yaml, dst: letsencrypt/traefik.yaml }
```

## Dependencies

None.

## License

MIT
