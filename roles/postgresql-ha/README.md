# postgresql-ha

3-node PostgreSQL 17 cluster on Debian 13 LXCs, all from Debian packages:

- **PostgreSQL** — one leader (read/write), two replicas streaming from it
- **Patroni** — runs PostgreSQL on each node, elects the leader, fails over
- **etcd** — 3-member store where Patroni keeps the leader lock

What happens when a node goes down and comes back:

| Node that goes down | While it is down | When it comes back |
|---|---|---|
| A replica | nothing changes | Patroni restarts it as a replica; it catches up from the leader |
| The leader | after ~30 s a replica is promoted | `pg_rewind` turns it into a replica of the new leader; if that fails its data dir is wiped and re-cloned |

The cluster needs **2 of 3 nodes up** (etcd quorum). With only one node left,
PostgreSQL is stopped/read-only until a second member returns.

## Usage

Hosts come from the `postgres_ha` group in `configure/inventory-dev`
(exactly three; `ansible_host` is the node IP, the inventory name is the
member name). Passwords come from `configure/group_vars/all/secret.all.sops.yaml`:

- `postgresql_ha_superuser_password` — `postgres` user
- `postgresql_ha_replication_password` — `replicator` user

```bash
ansible-playbook -i configure/inventory-dev configure/postgresql-ha.yaml
```

The first run bootstraps the cluster; the play ends with `patronictl list`.
Check it any time from a member:

```bash
patronictl -c /etc/patroni/config.yml list
patronictl -c /etc/patroni/config.yml switchover   # move the leader on purpose
```

## Variables

| Variable | Default | Purpose |
|---|---|---|
| `postgresql_ha_version` | `"17"` | PostgreSQL major (must be in Debian's repo) |
| `postgresql_ha_scope` | `homelab-postgres` | Patroni cluster name |
| `postgresql_ha_client_networks` | `[192.168.128.0/23]` | Allowed to connect to 5432 |
| `postgresql_ha_replication_user` | `replicator` | Replication role |
| `postgresql_ha_data_dir` | `/var/lib/postgresql/17/patroni` | PostgreSQL data |
| `postgresql_ha_etcd_data_dir` | `/var/lib/etcd/postgres-ha` | etcd data |
| `postgresql_ha_etcd_initial_state` | `new` | `existing` when re-adding a member |

## Firewall (UFW)

- 22/tcp from anywhere
- 2379, 2380 (etcd), 8008 (Patroni API), 5432 from the three members only
- 5432 from `postgresql_ha_client_networks`
- everything else incoming denied

## Not included (yet)

- **A single endpoint for apps.** Clients must find the leader themselves:
  libpq / JDBC accept several hosts —
  `postgresql://.133,.134,.135/db?target_session_attrs=read-write`.
  HAProxy or a VIP can come later.
- **Backups.** Replicas are not backups — add `pg_dump` / WAL archiving to the NAS.
- **TLS** on etcd and the Patroni API (firewalled to the members instead).

## Replacing a dead member

Its etcd identity has to be swapped by hand. From a healthy member:

```bash
etcdctl member list
etcdctl member remove <id>
etcdctl member add <name> --peer-urls=http://<ip>:2380
```

Then recreate the LXC and run the playbook with
`-e postgresql_ha_etcd_initial_state=existing --limit <name>`. Patroni re-clones
PostgreSQL from the leader on its own.
