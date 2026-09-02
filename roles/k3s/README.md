# k3s

Installs a [k3s](https://k3s.io) cluster on provisioned nodes using the official
`get.k3s.io` install script.

- Hosts in the **`k3s_server`** group become control-plane nodes. The first one
  is installed as the server, and its join token is read and exposed as a fact.
- Hosts in the **`k3s_agent`** group join that server as workers.

Run the two plays in order via `configure/kubernetes.yaml` (servers first, then
agents) so the agents can read the token from the server.

## Key variables

See `defaults/main.yaml`. Most-used:

| Variable | Default | Purpose |
|---|---|---|
| `k3s_version` | `""` (latest stable) | Pin, e.g. `v1.30.5+k3s1` |
| `k3s_server_extra_args` | `""` | e.g. `--disable traefik --disable servicelb` |
| `k3s_agent_extra_args` | `""` | Extra agent flags |
| `k3s_kubeconfig_mode` | `"600"` | Node kubeconfig perms (owner-only; holds admin creds) |
| `k3s_fetch_kubeconfig` | `false` | Copy kubeconfig to the controller (IP rewritten) |

## Usage

```bash
ansible-playbook -i configure/inventory configure/kubernetes.yaml
# fetch a ready-to-use kubeconfig to the repo root:
ansible-playbook -i configure/inventory configure/kubernetes.yaml -e k3s_fetch_kubeconfig=true
```

## Notes

- The install is guarded by `creates: /usr/local/bin/k3s`, so re-runs are no-ops
  once k3s is present — changing `*_extra_args` later won't re-apply on its own.
- Single control-plane (sqlite) by default. For embedded-etcd HA, add
  `--cluster-init` to the first server and `--server https://<first>:6443` to the
  others via `k3s_server_extra_args`.
- If you later apply the `security` role (UFW), open k3s ports: 6443/tcp (API),
  10250/tcp (kubelet), 8472/udp (flannel VXLAN), and 51820/udp if using wireguard.
