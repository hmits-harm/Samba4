# Temporary backup ring

pi16001 uploads to pi16002, pi16002 to pi16003, pi16003 to pi16001. Incoming
archives live at `/var/backups/samba-ad/incoming/<source-FQDN>/*.cms`.
Nightly jobs run at 02:00 Amsterdam time plus up to 30 minutes randomized delay.
Incoming files older than 14 days are pruned during successful local jobs.
This is age retention, not exactly 14 files. Monitor disk space and job failures.

Samba's `domain backup offline` locks local databases correctly while backing
up. Configuration/TLS material is included separately in the bundle. Archives
are CMS-encrypted with AES-256 for the separate RSA recovery certificate, then
sent over SSH with pinned host keys and source-restricted upload-only keys.
Plaintext staging is root-only and removed on ordinary completion/failure;
power loss may leave abandoned staging directories requiring cleanup.

Recovery private keys stay on the Ansible controller in `.artifacts/`. Make a
separate protected offline copy of `backup-recovery.key`, `backup-recovery.crt`
and the LDAPS CA material. Losing the recovery key makes archives unusable.
Never commit them. The ring shares site/power/admin risks and is a temporary
solution, not off-site protection. Nightly frequency permits up to 24 hours loss.

On an isolated trusted recovery host:

```bash
umask 077
mkdir recovery-test
openssl cms -decrypt -binary -inform DER -in source-backup.cms \
  -recip .artifacts/backup-recovery.crt \
  -inkey .artifacts/backup-recovery.key -out recovery-test/bundle.tar
tar -tf recovery-test/bundle.tar
tar -xf recovery-test/bundle.tar -C recovery-test
```

Use the inner Samba-generated backup with `samba-tool domain backup restore`
on a clean isolated machine running a compatible Samba version; consult command
help for required target/server/address arguments. Never unpack databases over
a live DC. Restore configuration deliberately and verify DNS, Kerberos, LDAP,
SYSVOL and controller metadata before reconnecting. A successful decryption
and archive listing is not a complete disaster recovery rehearsal.

Reference: [samba-tool](https://www.samba.org/samba/docs/current/man-html/samba-tool.8.html).
