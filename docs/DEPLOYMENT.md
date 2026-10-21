# Deployment record — 20 October 2026

Deployed on all three Debian 13.7 arm64 Pis using distribution Samba
4.22.11-Debian-4.22.11+dfsg-0+deb13u1. Forest/domain functional level is 2016;
all FSMO roles remain on pi16001. Approved IPv4 networks are mapped to AD site
Lab. The final full site deployment completed with no failed or unreachable hosts.

Checks performed:

- All five directory partitions replicated between every controller pair.
- DRS summaries reported `[ALL GOOD]` on all three controllers.
- Database and SYSVOL ACL checks passed; Chrony can access the signed-time socket.
- Trusted LDAPS and Kerberos-encrypted SMB SYSVOL access worked on each node.
- Cleartext LDAP password binds and SMB2-only clients were rejected.
- Every controller delivered encrypted backups to its ring destination; an
  archive from every receiver decrypted/listed successfully on the Ansible
  controller without copying recovery private keys to the Pis.
- SYSVOL source selection was exercised on pi16002 and pi16003 using the
  survivor-only playbook, then returned to pi16001. This tests source relocation,
  not physical outage or lost-database recovery.
- Current Ethernet IPv6 addresses are present in the firewall peer allowlist.
- No failed systemd units remained; update-availability logs were generated.
- External DNS resolution through the corrected 192.168.51.1 forwarder worked.

Outstanding integration/acceptance work:

- The existing client resolver **192.168.51.1 returns NXDOMAIN for AD LDAP and
  Kerberos SRV records**. It and DHCP remain unchanged. Clients retaining that
  resolver cannot discover AD until zone-wide forwarding is arranged. Individually
  selecting Samba DNS servers is an alternative; no client machines were changed.
- Existing non-controller records from the old lab.hmits DNS namespace have not
  been migrated automatically. Preserve/import any required records deliberately.
- Windows/macOS joins, GPMC and policy application require actual client tests.
- Microsoft AD CS templates/auto-enrollment are not implemented by Samba; only
  the private LDAPS CA and external-CA inputs are provided.
- FreeIPA trust is deferred until the partner domain/realm and version are known.
- Protocol encryption limitations are explicit in CERTIFICATES.md; an all-packet
  encryption requirement still needs a separately designed IPsec/VPN layer.
- Backups were decrypted/listed, but no isolated full domain restore was attempted.
  Arrange off-site/offline copies of backups and controller recovery/CA keys.

Two failed initial provisioning trees were retained on pi16001 as
`/var/lib/samba.failed-initial-provision-20261020` and
`/var/lib/samba.failed-functional-level-20261020`. They are not active domain
databases. Their private contents remain root-protected; remove/archive them
according to the eventual retention process.

Bootstrap convergence uses a temporary-scope KRB5_CONFIG selecting pi16001 to
avoid choosing a KDC before all joined controller SPNs have replicated. This
does not replace routine KDC discovery in /etc/krb5.conf. Cloud-init hostname
rewrites are disabled so the controllers' names continue resolving to Ethernet
addresses after reboot; IPv6 addressing is not managed by the role.
