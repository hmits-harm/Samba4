# FreeIPA on AlmaLinux 10 Raspberry Pi hosts

Entrypoint: `ansible-playbook freeipa.yml`. Install collection dependencies with
`ansible-galaxy collection install -r requirements.yml`. The wrapper role
`roles/freeipa_environment` prepares networking/firewall/time, invokes collection
roles via the entrypoint, and configures DNS and trust with collection modules.
Do not run its preparation-only main.yml as a substitute for the full entrypoint.

| Inventory/SSH identity | Permanent IPA identity | Ethernet address | Role |
|---|---|---|---|
| pi8003.lab.hmits | pi8003.ipa.hmits | 192.168.51.82 | Initial server |
| pi8004.lab.hmits | pi8004.ipa.hmits | 192.168.51.83 | Replica |

Both provide DNS, CA, KRA and AD trust-controller services for IPA.HMITS. This is
one replicated IPA deployment, not two independent CAs. The first installation
creates a self-signed IPA CA; certificate keys/KRA data replicate to the second
server. Directory Manager and admin use vaulted `ansible_password`.

The replica explicitly selects `ipareplica_servers: [pi8003.ipa.hmits]`; it does
not infer the replication source from the inventory's lab.hmits SSH alias.
Verification requires domain and CA topology segments joining the two permanent
IPA names, enabled CA/KRA/DNS/AD trust-controller roles on each, and cross-realm
Kerberos and AD identity resolution from each node.
The IPA CA does not automatically replace the existing Samba LDAPS private root CA, nor
implement Microsoft AD CS templates and Windows auto-enrollment.

Inventory retains the lab.hmits aliases for bootstrap. Connections use fixed
Ethernet addresses, pinned SSH keys under `.artifacts/ipa_known_hosts`, and the
permanent ipa.hmits names inside the IPA deployment. DHCP/router-provided names
are not used as IPA identities. Host mappings and cloud-init settings preserve
the selected identities. IP addressing, including IPv6 addressing, is unchanged.
After reinstallation, independently verify replacement host keys before updating
the pin file. The current pin file is Git-ignored; keep a protected copy with
the Ansible controller recovery material.

## DNS

IPA is authoritative for ipa.hmits. General forwarding goes to 192.168.51.1;
controllers use their own DNS with their peer as fallback. NetworkManager DNS
rewrites are disabled. Router and DHCP configuration remain unchanged.

IPA forwards lab.hmits and the shared 51.168.192.in-addr.arpa reverse zone to all
three Samba controllers. It does not create a conflicting reverse zone.
These private unsigned zones have scoped BIND `validate-except` entries, since
they have no public DNSSEC chain. Public-zone DNSSEC validation stays enabled.
Samba internal DNS lacks the per-zone forwarding needed here. Each AD controller
therefore uses a loopback-only Unbound service on 127.0.0.1:5354: ipa.hmits queries
go directly to the two IPA servers, all other external queries to 192.168.51.1.
Unbound is a forwarding proxy; it does not host another copy of either zone.
Its iterator-only configuration does not claim DNSSEC validation.

AD/IPA clients must use a DNS path that can resolve both realms' service records.
The existing router resolver previously returned NXDOMAIN for AD records, and
no change is made to it. Hosts using only that resolver may not discover either
realm. Existing non-controller host records are not implicitly migrated.

## Firewall and time

The firewalld ipa zone accepts identity/DNS/trust/SSH traffic from configurable
`ipa_allowed_subnets`, initially 192.168.51.0/24 and 192.168.7.0/24, plus the
current IPv6 addresses of the two IPA peers. The collection's broad automatic
firewall setup is disabled in favor of this scoped zone. Existing unrelated
firewall configuration is preserved. Trust RPC services use the firewalld
freeipa-trust definition, including dynamic RPC ports.
The AD firewall also admits the IPA peers' current IPv6 service addresses from
their host variables, since AD DNS publishes IPv6 controller addresses. Keep
those inventory addresses current when the network's IPv6 prefix changes; the
role does not configure IPv6 addressing itself.

Chrony uses nl.pool.ntp.org and serves only the configured `ipa_ntp_subnets`,
initially 192.168.51.0/24. Installation waits for clock synchronization. The
collection does not overwrite this time configuration.

## Authentication and trust

`allow_all` is never installed and is removed if encountered on a rerun. No
replacement broad access rule or user/group grant is fabricated. Configure HBAC
after setup. Local Ansible SSH accounts remain local; IPA/AD users may be denied
SSH until explicit HBAC grants exist. Trust does not imply access permission.

OpenSSH is configured with VerifyHostKeyDNS through the collection. Automatic
trust in SSHFP records depends on authenticated DNS/DNSSEC validation; setting
this option alone does not make unsigned DNS trustworthy.

The requested forest trust uses collection module ipatrust with two_way=true and
SID-based `ipa-ad-trust` ID mapping. Existing AD security settings must not be
silently weakened to establish it. Two-way trust permits relevant cross-realm
Kerberos workflows; it does not make IPA users supported Windows interactive
logon identities. See [FreeIPA limitations](https://www.freeipa.org/page/Windows_authentication_against_FreeIPA)
and the [EL 10 trust guide](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html-single/installing_trust_between_idm_and_ad/index).

## Checks and operations

Run `ansible-playbook playbooks/verify-freeipa.yml` for the automated checks.
Kerberos verification explicitly selects each node's own IPA KDC, so the replica
cannot pass its service-ticket test merely by using the primary's KDC. Temporary
verification credentials/configuration are removed afterward.

Check `ipactl status`, `ipa server-role-find`, `ipa topologysegment-find domain`,
`ipa topologysegment-find ca`, `ipa trust-show lab.hmits --all`,
`ipa trustdomain-find lab.hmits`, `getent passwd 'administrator@lab.hmits'`, DNS
SRV records in both directions and `chronyc tracking`. IPA CLI administration
requires `kinit admin` first. Never put passwords directly in shell arguments.

Both CA and KRA service roles must be enabled on both nodes before declaring
replication complete. Use the Web UI at https://pi8003.ipa.hmits/ipa/ui/ or its
peer; distribute the IPA CA certificate to administrators' trust stores first.
An identity lookup test does not prove HBAC authorization or Windows logon.

Deployment verification on both AlmaLinux 10.2 nodes passed: shared CA certificate,
healthy domain and CA replication agreements, all four server roles, bidirectional
forest trust, AD Administrator identity lookup, and AD-to-IPA HTTP service tickets
using each node's own IPA KDC. No HBAC rules were present. Samba-side trust inspection
reported `Direction: 0x3 (BOTH)` and `FOREST_TRANSITIVE`; the existing Samba verification
playbook passed on all three AD controllers after service recovery.

Configure independent encrypted/off-site IPA backups and test isolated restores;
replication is not a backup. This role does not install unattended updates or
erase an existing IPA deployment on failure. Preserve installer logs and recover
partial installations deliberately; do not wipe CA databases to bypass errors.
