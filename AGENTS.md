# Agent Rules

- This repository is the source of truth for the offloader configuration.
- Do not treat ad-hoc remote changes as a fix. Remote commands are allowed for diagnosis, verification, and controlled tests only.
- Any real fix for the offloader must be implemented in the Ansible playbook, roles, templates, variables, or tasks, and must survive both a playbook run and a reboot.
- Before changing live remote state, make clear whether the action is diagnosis, verification, a temporary test, or a persistent fix. Persistent fixes belong in this repo first.
- For WireGuard, VXLAN, BATMAN, bridge, and respondd issues, prefer changes in the relevant roles over manual systemd, iproute2, batctl, or file edits on the remote host.

# Repository Structure

- **Playbook**: `offloader.yml` — single playbook with multiple tagged plays, each mapping to one role.
- **Inventory**: `hosts.ini` — single host group `[offloader]`.
- **Host vars**: `host_vars/<hostname>.yml` — per-host variable overrides (secrets, MAC addresses, site-specific settings).
- **No group_vars** directory exists.

## Role Conventions

- Roles live in `roles/<role-name>/` and are **not** git submodules unless they have an entry in `.gitmodules`.
- Standard role layout: `defaults/main.yml`, `tasks/main.yml`, `handlers/main.yml`, `templates/`.
- All role variables are prefixed with the role name using underscores (e.g. `bandwidth_monitor_*`, `unifi_respondd_*`, `dnsmasq_*`). Cross-role shared variables use the `ff_` prefix (e.g. `ff_community_segment`, `ff_bridge_interface`).
- `become: yes` is applied per-task, not at the play level.
- Config files are deployed via Jinja2 templates with `notify: Restart <service>` handlers.
- Handlers follow the pattern: name `Restart <service>`, use `systemd` module with `daemon_reload: yes`, `enabled: yes`, `state: restarted`.
- Software is installed under `/opt/<service>/` for standalone installs, or via system packages (`.deb`) where available.

## Target Host

- The offloader (apu3faltigkeit) runs **Debian on amd64**. When adding `.deb` packages or kernel modules, always use `amd64` architecture.
- Network interfaces: `enp1s0` (WAN), `enp2s0` (LAN), `lo` (loopback). Bridge and VPN interfaces are created dynamically by the batman/wg/vxlan roles.

## GitHub Releases Pattern

The `unifi-respondd` role established the canonical pattern for downloading from GitHub Releases:

1. **Version strategy**: `*_version_strategy` with values `latest`/`release` (query API) or `fixed` (pin a tag).
2. **API lookup**: `uri` module with `delegate_to: localhost`, `become: no`, `run_once: true` against `https://api.github.com/repos/{org}/{repo}/releases/latest`.
3. **Fallback**: `*_release_fallback_tag` hardcoded tag used when the API is unreachable (rescue block).
4. **Fixed override**: `*_version` empty by default; used only when strategy is `fixed`.
5. **Resolved version**: `*_resolved_version` computed at runtime via `set_fact`, not a default variable.
6. **Assert task**: Validates strategy and resolved version are sane.

For `.deb` packages (e.g. `bandwidth-monitor`), the pattern is: download via `get_url`, install via `apt` module with `deb:` parameter, then remove the temp file. Config goes to `/etc/<service>/env` when using system packages.

## Adding a New Role — Checklist

1. Create `roles/<role-name>/` with `defaults/main.yml`, `tasks/main.yml`, `handlers/main.yml`, and `templates/` as needed.
2. Prefix all variables with `<role_name>_` (using underscores, not hyphens).
3. Add a play to `offloader.yml` with `hosts: offloader` and at least one tag matching the role name (plus an optional short alias tag like `bwm` for `bandwidth-monitor`).
4. If the role downloads from GitHub Releases, follow the **GitHub Releases Pattern** above. Use `delegate_to: localhost` + `run_once: true` for the API lookup, and always include a fallback tag in a `rescue` block. See `roles/bandwidth-monitor/` as the `.deb` example or `roles/unifi-respondd/` as the git-clone example.
5. If the role installs a `.deb` package: download to `/tmp/`, install with `apt: deb:`, then clean up. Config goes to `/etc/<service>/env` (system package convention). If it's a standalone `/opt` install, config goes to `/opt/<service>/.env`.
6. If per-host overrides are needed, add variables to `host_vars/<hostname>.yml`.
7. Run `ansible-playbook --syntax-check offloader.yml` to validate before committing.
8. Do **not** add the new role to `.gitmodules` unless it has its own upstream git repo — local roles are fine.

## ansible.cfg Highlights

- Inventory: `./hosts.ini`
- Fact caching: `jsonfile` in `./facts/` with 2h timeout
- SSH pipelining enabled, 3 retries on SSH failure
- Output callback: `default` with `callback_result_format: yaml`

## Current Roles

| Role | Source | Install method |
|------|--------|---------------|
| `ssh-authorized-keys` | git submodule | ssh key deployment |
| `cli-tools` | git submodule | apt packages |
| `dnsmasq` | git submodule | apt + templates |
| `docker-unifi` | git submodule | docker container |
| `batman` | git submodule | DKMS build + templates |
| `wg-ffmuc` | git submodule | wireguard + templates |
| `vxlan-ffmuc` | git submodule | vxlan + templates |
| `respondd` | git submodule | git clone + templates |
| `unifi-respondd` | git submodule | git clone + templates |
| `bandwidth-monitor` | local (not submodule) | `.deb` from GitHub Releases + template |

## Running the Playbook

```bash
# Full playbook
ansible-playbook offloader.yml

# Single role by tag
ansible-playbook offloader.yml -t bandwidth-monitor
ansible-playbook offloader.yml -t bwm

# Syntax check
ansible-playbook --syntax-check offloader.yml

# Dry run (note: --check will skip downloads, so .deb installs will fail — this is expected)
ansible-playbook --check --diff offloader.yml -t <tag>
```