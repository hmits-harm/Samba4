# Samba AD on Raspberry Pi 5 / Debian 13

Three writable controllers, new forest `lab.hmits`, realm `LAB.HMITS`, short
domain `LAB`, site `Lab`, functional level 2016. Initial FSMO and SYSVOL source:
`pi16001`. The other controllers are `pi16002` and `pi16003`.

Ethernet addresses are DHCP reservations `192.168.51.60`, `.61`, `.62`; gateway
is `192.168.51.1`. Samba binds Ethernet and loopback, excluding Wi-Fi. The role
does not manage IPv6 addressing; current Ethernet IPv6 addresses are included
in certificates and peer firewall access. Additional IPv6 client prefixes require
`samba_ad_ipv6_allowed_subnets`. Rerun after service address changes.

```bash
ansible-galaxy collection install -r requirements.yml
ansible-playbook site.yml --syntax-check
ansible-playbook site.yml
```

Run from this directory; `ansible.cfg` selects SSH user, inventory and Vault file.
Existing vaulted `ansible_password` supplies SSH, sudo and the initial domain
Administrator password. Password tasks suppress output. Provisioning briefly
passes its initial password to samba-tool as an argument, visible to privileged
local process inspection. Root-only `/etc/samba-ad/admin.auth` supports domain
administration; update it after password rotation. No disk formatting occurs.
Existing domain databases are never automatically reprovisioned. Initial runs
must include all three hosts in inventory order. Check mode cannot fully simulate
domain provisioning.

Configuration: `inventory/group_vars/samba4.yml`, per-host files, and
`roles/samba_ad/defaults/main.yml`. AD and SSH access starts with
`192.168.51.0/24` and `192.168.7.0/24`. nftables uses its own table without
flushing other tables. ICMPv6/DHCPv6 remain available. RPC dynamic range is
49152–49252. Existing unrelated firewall rules can impose further restrictions.

Controllers use their own/peer DNS; NetworkManager DNS updates are disabled
without changing addresses. External names forward to `192.168.51.1`. DHCP and
that device remain untouched. Client DNS must resolve the entire `lab.hmits`
zone, including `_msdcs` service records; forwarding just hostnames is insufficient.
Chrony uses `nl.pool.ntp.org` and Samba signed-time support.

The existing client resolver remains `192.168.51.1`; it currently returns NXDOMAIN
for AD service discovery. Client rollout requires either zone-wide forwarding on
that device or individually configured Samba DNS servers (`192.168.51.60`, `.61`,
`.62`). The role changes neither DHCP nor that device. This limitation must be
resolved before clients can discover AD through the existing resolver.

Password policy: minimum 14 characters, complexity, 24 remembered passwords,
no periodic expiry, 15-minute lockout after 10 failures in 15 minutes.

Root-only operational logs: `/var/log/samba-ad/`, daily rotation, 14 rotations.
Health checks run every 15 minutes. Update notices run daily at 09:00 Amsterdam
time with randomized delay; no external notifications yet. No automatic update
installation or reboot occurs. Inspect `systemctl --failed` and
`systemctl list-timers 'samba-ad-*'`.

Read [SYSVOL failover](docs/SYSVOL-FAILOVER.md), [manual updates](docs/UPDATES.md),
[certificates/security](docs/CERTIFICATES.md), [backups](docs/BACKUPS.md), and
[optional objects/FreeIPA](docs/OBJECTS-AND-TRUST.md).
The [deployment record](docs/DEPLOYMENT.md) lists checks and remaining integration
work, including the existing resolver's missing AD service records.

Run `ansible-playbook playbooks/verify.yml` for server health, Kerberos-encrypted
SYSVOL access, rejection of SMB2-only clients and cleartext LDAP binds, trusted
LDAPS, and encrypted archive decryption/listing on the Ansible controller. This
does not perform a full domain restore or certify Windows/macOS client joins.
