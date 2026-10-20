# Manual controller updates

Available package updates are reported to `/var/log/samba-ad/updates.log` daily.
Unattended installation is disabled; repository metadata refresh is permitted.
There is no email/desktop notification. To check explicitly:

```bash
ansible samba4 -m command -a 'systemctl start samba-ad-updates.service'
ansible samba4 -m command -a 'tail -60 /var/log/samba-ad/updates.log'
```

Review package changes, backups, replication and recovery access first. To
initiate installation and sequential reboots yourself:

```bash
ansible-playbook playbooks/maintenance.yml
```

The playbook checks every peer, creates a fresh backup, upgrades one controller,
reboots it, and waits for health checks before proceeding. Failures stop the
rollout. Keep healthy controllers online and correct the failure before retrying.
Never reboot all three together. Major OS upgrades and schema/functional-level
changes require separate planning. Do not blindly downgrade Samba databases.

After maintenance verify DRS replication, SYSVOL, DNS service records, Chrony,
LDAPS trust, backup freshness, and actual Windows login/Group Policy application.
