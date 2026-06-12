# Roles Review — 2026-06-11

> **Status update (same day):** all "Fix first" items, the documentation gaps, and
> the dead-weight removals below have been applied across the 11 roles (both
> playbooks pass `--syntax-check`; compose templates render-validated). Still open,
> deliberately left for a human decision:
> the `NewStart/` deletion (contains a git-ignored secrets file), the
> `NodeSetup/`→`nodesetup/` casing rename (git currently tracks BOTH casings),
> the missing UrBackup client on master (deployment choice), the role/var
> renames (`role-name` rule, `prune_*` prefixes — repo-wide convention change),
> `meta/argument_specs.yml` adoption, and the UFW `limit` on 80/443 (possibly
> deliberate hardening).

Review of the 11 active roles in `roles/` against the standard role template
(geerlingguy-style: `defaults/` with every variable declared, `meta/`, `README.md`
documenting vars + defaults, FQCN, idempotent tasks), done in two passes: one
reviewer per role plus cross-role checks, then claim-by-claim verification against
the actual files and group_vars. Claims that didn't survive verification were
dropped (e.g. "alloy user-add fails because the docker group doesn't exist" — the
security role creates that group first). Legacy roles in `roles/legacy/` were not
reviewed.

**Overall:** every active role has `defaults/main.yaml`, `meta/main.yaml` and a
`README.md`; modules are FQCN with few exceptions; structure is solid. The real
findings: one play-aborting bug in `tailscale`, a few deploy-composes template
bugs, missing-prerequisite and idempotence issues, and documentation gaps
concentrated in the two Proxmox guest roles.

## Fix first (verified issues)

| Where | Issue |
|---|---|
| **tailscale** | `meta/main.yaml` declares `artis3n.tailscale.machine` under `dependencies:`. That run executes *before* `tasks/main.yaml` with the collection defaults (`tailscale_authkey: ""`, `tailscale_up_skip: false`), so the collection's "Tailscale Auth Key Required" `fail` task fires — **`configure.yaml` aborts at the Tailscale role on every host**. The `include_role` in tasks (which passes the authkey) never runs. Set `dependencies: []`; the collection requirement belongs in `requirements.yml`. |
| **deploy-composes** | `templates/media.yaml` marks the `immich` network `internal: true`, but `immich-machine-learning` is *only* on that network — model downloads will fail. CLAUDE.md's own Immich section says this network must NOT be internal. |
| **deploy-composes** | `clean-up-containers.yaml` is imported unconditionally and force-removes every container not belonging to a deployed compose project — one-off/debug containers die on every run. The `docker.cleanup: true` flag in `group_vars/all/all.yaml:48` looks like the toggle but is consumed nowhere. Wire the flag up (or gate with `tags: [never, ...]` like docker-maintenance). |
| **security** | `nas.yaml` mounts a CIFS share but nothing installs `cifs-utils` on the target node (`extra_packages` doesn't include it; only proxmox-host installs it, on the host). Fresh provision of a `mount_nas: true` node fails at the mount. |
| **proxmox-vm** | `qm resize` runs with `changed_when: true` and no size check — Proxmox only grows disks, so re-runs with `vm.disk_gb` set fail. Guard with a current-size check or tolerate the already-sized error. |
| **proxmox-lxc** | `Create LXC container` passes `api_password` without `default(omit)` alongside the token vars (mutually exclusive auth; the `Start` task does it right). Also the inline default password `'ChangeMe123!'` — should fail loudly or come from secrets. |
| **recap** | Both package counters are wrong: "Pkgs Installed" always shows `✗ None` (the `ir.get('results') is defined` branch — `.get()` returns `None`, which *is* defined, and a non-looped register has no `results`, so the count is always 0), and "Pkgs Upgraded (N)" counts `Inst` lines that apt only emits in check mode, so real runs show `(0)`. The Yes/No flags are correct; only the counts lie. |
| **opkssh-server** | The role errors under `--check`: the version-probe `command` is skipped, so `opkssh_version_check.rc` is undefined in the `when:` of the next two tasks. Add `check_mode: false` to the probe. |
| **proxmox-host** | The `ubuntu_cloudimg_filename` `set_fact` is untagged while all its consumers are tagged — `--tags setup` (advertised in the README) skips it and setup.yaml hits an undefined variable. Latent today only because `nodesetup.yaml` defines the same expression as a play var. Simplest fix: move it to `defaults/main.yaml` as `{{ ubuntu_lts_image_url | basename }}` and delete the set_fact. |
| **security** | `ansible.builtin.authorized_key` → `ansible.posix.authorized_key` (collection already required). Also `user.yaml` creates literal groups `admin, docker, sudo` but adds the user to `{{ ssh_group }}`/`{{ sudo_group }}` — overriding those vars breaks user creation. |
| **monitoring** | The node_exporter smoke test targets `node_exporter_host`, but the bind address is `node_exporter_listen_address` — works on MonitoringMaster only because the Tailscale IP happens to be reachable. Test the listen address or `localhost`. |
| **docker** | Old-repo removal hardcodes `bionic` and masks failures with `failed_when: false` — silent no-op on any other release. Either fix or delete the migration tasks (see dead weight). |
| **deploy-composes** | Volume-dir creation uses `stat` + conditional `file` (and the `stat` lacks `become`), so an existing dir with wrong ownership is never corrected. One unconditional `ansible.builtin.file` with `state: directory` + owner/group is idempotent and simpler. |

## Documentation gaps (minor)

- **proxmox-vm / proxmox-lxc** — `defaults/main.yaml` in both is comments only; vars with sensible defaults (`proxmox_bridge`, `proxmox_api_user`, `validate_certs`, `clear_pass`, `lxc_features`…) live as inline `default(...)` in tasks. Declare them and list them in the README.
- **proxmox-host** — `igpu_video_gid`/`igpu_render_gid` facts are an undocumented contract with proxmox-lxc (and running `--tags proxmox-lxc` alone leaves them unset for `enable_igpu` containers). `lxc_features` is declared here but consumed only by proxmox-lxc — move it there. `cloud_init_keyboard_layout` and `lxc_containers` are consumed but undeclared.
- **deploy-composes** — README documents only `deploy_composes_pull_policy`; undocumented: `username`, `docker.docker_dir`, `docker.docker_networks`, and the `volumes[].uid/gid` / `config[].force` sub-keys of the composes contract.
- **monitoring** — `node_exporter_uid`/`gid` consumed but undeclared; ~half the defaults missing from the README table; `services_ip` (and the implied tailscale-first ordering) not in Requirements. `alloy_version` in defaults doesn't actually pin the included grafana role (its own default wins) — it only works because group_vars repeats it; pass it explicitly in the `include_role` vars.
- **security** — `reboot_*` vars and the `nas` dict shape missing from the README. The "security pocket only" comment on `security_updates_only` is inaccurate — `upgrade: safe` applies all upgrades, just without removals.
- **docker** — external dep on `username` and the exported `docker_version` fact (consumed by recap) not mentioned in the README.
- **opkssh-server** — `opkssh_providers` (drives `providers.j2`, lives in `secret.all.yaml`) undeclared and absent from the README. Also: bumping `opkssh_install_ref` never upgrades an existing install (existence-only gate) — document it.
- **tailscale** — `tailscale_args: "--ssh"` hardcoded and undocumented; README says the role "sets `services_ip`" but that mapping happens in group_vars.

## Unused / dead — removal candidates

- **docker** — never-notified `restart docker` handler; the GPG-key/bionic migration tasks; the `/etc/docker` dir task (package creates it); redundant `failed_when` on the smoke test.
- **deploy-composes** — `templates/config/calibre-database.db` (committed SQLite binary, referenced by nothing); two git-tracked `.DS_Store` files (`.gitignore` doesn't exclude them); `allowed_containers` fact computed but never used; the lone `when: docker.composes is defined` guard on the last task (three earlier tasks loop unguarded); `slave-containers.yaml` declares a `proxy_network` no service joins; prometheus mounts `docker.sock` it never uses (static_configs only).
- **security** — dead `install_result` set_fact stub (a skipped register is still defined, and recap defends anyway); redundant inline `default()`s duplicating declared defaults (also in tailscale, opkssh-server, deploy-composes).
- **proxmox-host** — both `stat` + `when` guards in setup.yaml duplicate `creates:`/`get_url` semantics; the inner `cifs_mounts | length` guard duplicates the import-level gate (and lacks `default([])`); the `default()` in the cloudimg expression can never fire (same flawed expression duplicated in nodesetup.yaml).
- **monitoring** — dead `or node_exporter_version is not defined` branch; promtail stop/disable migration shim (all nodes migrated).
- **proxmox-vm** — `created_vm` register never read.

## Cross-role / repo-level

- **`NewStart/` leftover** — contains only `group_vars/all/secret.all.yaml`; `configure/` has its own copy. Confirm identical, then delete (manual — it's git-ignored content).
- **`NodeSetup/` casing** — docs say `nodesetup/`; the directory is `NodeSetup/`. Fine on APFS, breaks on Linux/CI. Two-step `git mv` via a temp name.
- **`NodeSetup/group_vars/all/prod.yml`** — duplicates `all.yml`; and anything under `group_vars/all/` applies to every host, so a prod-specific file is misplaced there.
- **Monitoring defaults duplicated** — `configure/group_vars/all/all.yaml:75-95` repeats every monitoring default verbatim; group_vars outrank defaults, so editing the role's defaults silently does nothing in this playbook. Trim group_vars to actual overrides.
- **docker-maintenance framing vs deployment** — README/defaults stress "all toggles default false", but group_vars sets `prune_images: true`, so a plain `--tags docker-maintenance` run *does* prune. Also the README example omits `-i configure/inventory` — as written it matches zero hosts.
- **SSH config ownership** — `security` (drop-in hardening + `PasswordAuthentication` in `user.yaml`) and `opkssh-server` (lineinfile into main `sshd_config`, Include line removed) both own SSH directives; both define a handler named `Restart SSH` that actually *reloads*. Works today; consolidate ownership and rename the handlers `Reload SSH` when convenient.
- **media-management.yaml** — bazarr routers consistently misspelled `barzarr-api` (harmless, but diverges from the naming pattern).
- **Vestigial master backup client** — `backup.yaml`'s dedicated `backup` network has only the server in it, and master's composes list has no `slave-containers.yaml`, so master runs no UrBackup client while `master.yaml` still carries client settings (`urbackup.server_host`, `authkey`) consumed by nothing. Either add the client back or drop the dead config.
- **other.yaml (mealie)** — secrets render unquoted (`POSTGRES_PASSWORD: {{ mealie.db_pass }}`); a password containing `:`/`#`/`{` produces a broken compose file. Quote them like every other template.

## Template-standard notes (project-wide conventions, not per-role bugs)

Checked against current ansible-lint rules and Geerling's conventions:

- Role directory names with hyphens (`deploy-composes`, `docker-maintenance`, `opkssh-server`, `proxmox-*`) violate ansible-lint's `role-name` rule (`^[a-z][a-z0-9_]*$`). A rename is invasive; if keeping them, skip that rule deliberately in `.ansible-lint`.
- Variables are mostly unprefixed (`prune_*`, `allow_reboot`, `extra_packages`…) rather than role-prefixed — a repo-wide convention decision; the docker-maintenance `prune_*` names are already overridden from a group_vars file that never mentions the role, which is the exact risk the prefix rule prevents.
- `meta/argument_specs.yml` (role input validation) is the modern complement to defaults+README; worth adopting for the roles with required external vars (deploy-composes, proxmox-*).
- CI gate: the repo has yamllint config, but `ansible-lint --profile basic` (tightening over time) would catch most of the mechanical findings in this report automatically.

## Clean

**docker-maintenance** remains the best-shaped role (fully documented, properly
`never`-tag gated) — only the framing/README nits above. **recap** is structurally
clean but its two counter bugs are listed under Fix first.
