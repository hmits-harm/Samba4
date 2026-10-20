# Optional objects and future FreeIPA

`samba_ad_objects` is separate and defaults to no additions. Samba provisioning
creates mandatory built-ins and default policies. Set these lists explicitly:

```yaml
samba_ad_objects_ous:
  - OU=People,DC=lab,DC=hmits
samba_ad_objects_groups: [LabUsers]
samba_ad_objects_users:
  - name: example
    password: '{{ vaulted_example_password }}'
samba_ad_objects_gpos: [Lab workstation baseline]
```

Run `ansible-playbook playbooks/objects.yml`. OUs must be ordered parents first.
Groups/users use standard containers. Existing objects are preserved; the role
does not delete, move, reset passwords or reconcile GPO content. GPOs are empty
objects; use RSAT/GPMC against the SYSVOL source to edit content and links.
Vault initial passwords. Use delegated accounts for routine administration.

FreeIPA configuration/trust is deferred until its EL 10 version and separate
domain/realm are known. Intended direction: LAB AD users access FreeIPA clients,
with authorization through IPA HBAC/sudo rules. Before trust setup validate exact
versions, bidirectional DNS forwarding, time, firewall networks and ID ranges.
No trust credentials or partner namespace are fabricated. Trust does not grant
every AD user access automatically. See [FreeIPA documentation](https://www.freeipa.org/page/Active_Directory_trust_setup.html).
