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
