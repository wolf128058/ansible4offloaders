# Agent Rules

- This repository is the source of truth for the offloader configuration.
- Do not treat ad-hoc remote changes as a fix. Remote commands are allowed for diagnosis, verification, and controlled tests only.
- Any real fix for the offloader must be implemented in the Ansible playbook, roles, templates, variables, or tasks, and must survive both a playbook run and a reboot.
- Before changing live remote state, make clear whether the action is diagnosis, verification, a temporary test, or a persistent fix. Persistent fixes belong in this repo first.
- For WireGuard, VXLAN, BATMAN, bridge, and respondd issues, prefer changes in the relevant roles over manual systemd, iproute2, batctl, or file edits on the remote host.
