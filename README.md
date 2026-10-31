# WireGuard-Endpoint-Log

Logs every new source endpoint observed for each WireGuard peer public key on OpenWrt.

## Configuration

`/etc/config/wireguard-endpoint-log`:

| Option | Default | Meaning |
|--------|---------|---------|
| `enabled` | `1` | Run the service |
| `interval` | `2` | Poll interval in seconds |
| `ip_only` | `0` | `1` ignores port changes and logs IP only |
| `log_file` | unset | Optional copy of the log on persistent storage |
| `ignore_cidr` | unset | IPv4 range(s) whose endpoints are never logged, repeatable |

`ignore_cidr` takes `a.b.c.d/n` or a bare address (same as `/32`). Each wildcard octet subtracts 8 from the prefix:

| Matches | `ignore_cidr` |
|---------|---------------|
| `10.9.10.5` only | `10.9.10.5/32` or `10.9.10.5` |
| `10.9.10.*` | `10.9.10.0/24` |
| `10.9.*.*` | `10.9.0.0/16` |
| `10.*.*.*` | `10.0.0.0/8` |

Use it to drop LAN endpoints, e.g. an iOS on-demand tunnel that still connects over local Wi-Fi:
```
	list ignore_cidr '10.9.10.0/24'
	list ignore_cidr '192.168.1.0/24'
```
Host bits are ignored, so `10.9.10.5/24` is the same as `10.9.10.0/24`. Octets and prefixes with leading zeros (`010`) are invalid. Invalid entries are logged to syslog and skipped, and a `/0` logs a warning since it ignores every IPv4 endpoint. IPv6 endpoints are never ignored.

## Log format

Written to syslog (`logread -e wireguard-endpoint-log`). `log_file` format:

```
20261031-05:06:02 started interval=2s ip_only=0
20261031-05:06:25 iface=wg0 peer=myphone endpoint=70.172.116.173:57891 pubkey=xr4l...3TQ=
20261031-05:08:15 iface=wg0 peer=wifephone endpoint=207.119.148.124:61500 pubkey=gDTT...C2U=
20261031-05:09:33 iface=wg1 peer=basecamp endpoint=103.102.36.53:51110 pubkey=8Nf/...m0=
```

## Persistent logs

Syslog is RAM and is cleared on reboot. Optionally point `log_file` at persistent storage. If it is under an `/etc/config/fstab` mount, lines are only written while that mount is present.

## Pro tip: get an aligned log

Install the `column` package and query the log with it for a nicer aligned display:
```
% column -t /mnt/spare/wireguard-endpoint.log
20261031-05:06:25  iface=wg0  peer=myphone    endpoint=70.172.116.173:57891   pubkey=xr4l...3TQ=
20261031-05:08:15  iface=wg0  peer=wifephone  endpoint=207.119.148.124:61500  pubkey=gDTT...C2U=
20261031-05:09:33  iface=wg1  peer=basecamp   endpoint=103.102.36.53:51110    pubkey=8Nf/...m0=
```

## Installation

See https://github.com/openwrt/packages/pull/31066

## Notes

- Changes faster than the poll interval can be missed (A to B to A within one interval).
- Mobile carriers commonly use CGNAT, so these endpoints change often.
- Rapid alternation between two endpoints for the same key suggests the key is in use on two devices at once.
- Site-to-site peers with a configured `endpoint_host` show that address before first contact; those lines do not represent an incoming connection.
