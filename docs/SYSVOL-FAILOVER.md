# Moving SYSVOL synchronization to another controller

All three controllers replicate directory data normally. Separately, followers
pull policy files once per minute from `samba_ad_sysvol_source`, initially
`pi16001.lab.hmits`. rsync preserves ACLs and xattrs over pinned SSH, using
source-IP-restricted read-only forced commands. All three potential sources are
prepared. Always target the current source explicitly in Windows GPMC for edits.

1. Stop policy editing. Isolate the failed source so it cannot accept new edits.
2. Check `samba-tool drs showrepl`, `samba-tool ntacl sysvolcheck`, and
   `/var/log/samba-ad/sysvol-last-success` on survivors. Compare GPO versions
   and `GPT.INI` files. Choose the latest consistent copy. Last-minute changes
   may not have replicated; reconcile with backups before selecting a source.
   For explicit Kerberos administration, append
   `-A /etc/samba-ad/admin.auth --use-kerberos=required` to `drs showrepl`.
3. Stop `samba-ad-sysvol-sync.timer` on survivors. Preserve each SYSVOL tree
   with ACLs/xattrs, e.g. `cp -a /var/lib/samba/sysvol /var/backups/samba-ad/sysvol-before-failover`.
   Preserve earlier snapshots before repeating this command.
4. Set `samba_ad_sysvol_source` in `inventory/group_vars/samba4.yml` to either
   `pi16002.lab.hmits` or `pi16003.lab.hmits`.
5. Apply to both reachable survivors, excluding the failed host:

   ```bash
   ansible-playbook playbooks/sysvol-source.yml --limit 'pi16002.lab.hmits,pi16003.lab.hmits'
   ```

6. Verify policy contents, successful sync timestamps and SYSVOL ACL checks.
   Resume editing through the new source only. Synchronization mirrors deletions;
   do not select an empty tree. No automatic source election is attempted.

On recovery of the old source, keep it isolated from policy editing, stop its
sync timer, preserve its old tree, and run this playbook including all reachable
controllers. It becomes a follower of the configured source before returning to
service. Do not copy its stale policies onto the new source. A lost domain
database requires a separate supported controller recovery/rejoin procedure.

SYSVOL source changes do not move FSMO roles. For planned transfer run
`samba-tool fsmo transfer --role=all` on the intended destination. Permanent loss
may require `samba-tool fsmo seize --role=all`, only after validating survivors,
backups and permanent isolation of the old holder. Never reconnect an old role
holder after seizure without proper demotion/rebuild. A temporary outage alone
does not justify seizure. If a ring member is permanently lost, also change
backup destinations. Do not run the complete site play against an absent node.
