# AD controller listeners and startup recovery

Use `sudo ss -lntup` to inspect local listening sockets. Firewall rules control
remote access; they do not hide listeners from this command.

| Service | TCP | UDP |
|---|---|---|
| DNS | 53 | 53 |
| Kerberos | 88 | 88 |
| Kerberos password changes | 464 | 464 |
| LDAP / CLDAP | 389 | 389 |
| LDAPS | 636 | |
| SMB | 445 | |
| Global Catalog / TLS | 3268, 3269 | |
| RPC endpoint mapper | 135 | |
| Dynamic RPC | Configured `samba_ad_rpc_ports` range | |
| NetBIOS compatibility | 139 | 137, 138 |
| Time (Chrony) | | 123 |

Samba binds loopback and the configured Ethernet interface. Allowed client
subnets are controlled separately by the firewall. Dynamic RPC listeners use
ports within the configured range; every port in the range need not be listening.

If several AD services disappear together, check:

```sh
systemctl status samba-ad-dc --no-pager -l
journalctl -u samba-ad-dc -b --no-pager
getent ahosts "$(hostname -f)"
ip -br address
```

On pi16001, a boot failure reported `NT_STATUS_ADDRESS_NOT_ASSOCIATED` while
binding its IPv6 address. Restarting Samba after the address became available
restored its AD listeners. The role now installs a systemd override with
`Restart=on-failure` and a 15-second retry interval, after `network-online.target`.
This changes service recovery only; it does not configure IPv6 addressing.

The role also restores Ethernet hostname mappings in `/etc/hosts`; mapping a
controller name to `127.0.1.1` breaks local administration when Samba listens on
`127.0.0.1` and Ethernet. Verify replication after recovery with:

```sh
sudo samba-tool drs showrepl --summary -A /etc/samba-ad/admin.auth --use-kerberos=required
```
