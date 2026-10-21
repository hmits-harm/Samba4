# Certificates and security scope

Samba cannot host Microsoft AD CS enterprise certificate templates and Windows
auto-enrollment. That requires a Windows member server running AD CS or a
separately evaluated compatible enrollment service. None is currently available.
Samba's enrollment-client features do not make it a Microsoft enterprise CA.
See the [Samba advisory](https://www.samba.org/samba/security/CVE-2026-3012.html).

This setup issues LDAPS certificates from a private root on the Ansible controller.
`.artifacts/` is mode 0700; private keys are 0600 and Git-ignored. The root signing
key never goes to the Pis. Distribute `.artifacts/ca.crt` to client root trust
stores, never the signing key. Back up the complete artifact directory securely
outside these Pis. Protect the Ansible controller with encrypted storage.

Server certificates last one year; rerun site.yml to renew near expiry and deploy
changed certificates. Health checks warn 30 days before expiry. Root lifetime is
ten years; plan overlapping trust before replacement.

For an external CA, set `samba_ad_external_ca: true` and per-host controller-local
PEM paths `samba_ad_external_cert`, `samba_ad_external_key`, and
`samba_ad_external_chain`. Use serverAuth and SANs for controller FQDN/service
addresses. Include intermediate certificates in the server certificate file and
trust anchors in the chain file. Distribute new trust first, rerun deployment,
then verify every LDAPS endpoint. No external CA private key is required.

SMB3 encryption is required; signing is mandatory and NTLM disabled. LDAP rejects
cleartext password binds and enforces TLS channel binding; outbound SASL uses
sealing. **Samba strong-auth LDAP still permits signed-only SASL** on port 389.
DNS, discovery and initial NTP are not universally encrypted. The role therefore
does not guarantee encryption of every packet. A mandatory all-packet encryption
policy needs a separate IPsec/VPN design. See [Samba settings](https://www.samba.org/samba/docs/current/man-html/smb.conf.5.html).

Actual Windows/macOS joins, GPMC and future FreeIPA trust need client-side
acceptance tests. None of those client machines have been supplied. Windows 11
requires a domain-capable edition such as Pro; future OS compatibility cannot be
guaranteed merely by selecting functional level 2016.
