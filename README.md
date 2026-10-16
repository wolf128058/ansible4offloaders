# Ansible for Offloaders
Freifunk-Offloaders installed by Ansible-Roles


## Preparations on target systems:
* You need to install ansible on your remote-machine.
* You have to become root for some roles. So get yourself automatically given sudoer-permissions. 
## Customizations ToDo:
* Clone **hosts.ini.example** and make your own **hosts.ini**

## System Updates
- `ansible-playbook offloader.yml --tags updates-check` — read-only report of pending apt updates (no reboot / needs reboot, security, held), pending reboot, services on outdated libraries
- `ansible-playbook offloader.yml --tags updates-apply` — same report, then upgrades everything that needs no reboot (kernel, firmware, libc, systemd … stay held back)
- neither tag runs in a plain playbook run; details: [roles/system-updates/README.md](roles/system-updates/README.md)

## Automatic Updates (role `autoupdate`)
- `ansible-playbook offloader.yml --tags autoupdate` — configures Ubuntu's `unattended-upgrades` to install all regular updates (`-security`, `-updates`) nightly via `apt-daily-upgrade.timer` (`autoupdate_schedule`, default 04:00 + up to 30 min)
- never automatic: everything in `autoupdate_hold_patterns` (kernel, firmware, DKMS, libc, systemd, dbus — same as `system_updates_reboot_patterns`) and `autoupdate_extra_hold_patterns` (dkms, wireguard, iproute2, netplan/ifupdown, dnsmasq, docker/containerd), nor packages whose new version needs one of them → these stay for `--tags updates-apply`
- no automatic reboot, needrestart only lists (`autoupdate_needrestart_mode: l`)
- history: `/var/log/unattended-upgrades/`, `journalctl -u apt-daily-upgrade`; dry run on the host: `sudo unattended-upgrade --dry-run -v`
