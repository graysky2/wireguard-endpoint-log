# WireGuard-Endpoint-Log

Logs every new source endpoint observed for each WireGuard peer public key on OpenWrt.

## Why

WireGuard does not log connections. If a peer's private key is copied off a lost phone, a backup or a shared config, whoever has it can connect as that peer and nothing on the router says so.

A typical home setup has a handful of peers: family phones, a laptop, maybe a remote site. Each one tends to connect from a small, predictable set of networks: the home ISP, a mobile carrier, work, in the same region. This log is an audit trail of every place each key has connected from, so you can look back and check that the pattern holds.

A peer that suddenly connects from an unfamiliar ISP, a hosting provider or VPN, or a city or country its owner has never been to is a red flag that the key may be compromised. The default `asn` and `geoip` fields show that directly; with them off, look up the addresses by hand. If a key looks compromised, remove the peer and issue a new key pair.

## Configuration

`/etc/config/wireguard-endpoint-log`:

| Option | Default | Meaning |
|--------|---------|---------|
| `enabled` | `1` | Run the service |
| `interval` | `2` | Poll interval in seconds |
| `ip_only` | `1` | Log IP only and ignore port changes; `0` logs `ip:port` and every port change |
| `asn` | `1` | Add the endpoint's `ASN=`; `0` disables (see [Lookups](#lookups)) |
| `geoip` | `1` | Add the endpoint's `ISP=`, `Country=` and `City=`; `0` disables (see [Lookups](#lookups)) |
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
20261031-05:06:02 started interval=2s ip_only=1 asn=1 geoip=1
20261031-05:09:33 iface=wg1 peer=basecamp City=Kathmandu,BagmatiProvince Country=NP ISP=InternetExchangeNepal ASN=AS17412 endpoint=103.102.36.53 pubkey=8Nf/...m0=
20261031-07:12:48 iface=wg0 peer=wifephone City=Cottonwood,Arizona Country=US ISP=CABLEONE ASN=AS11492 endpoint=66.164.137.1 pubkey=gDTT...C2U=
20261031-08:05:19 iface=wg0 peer=daughterphone City=Cottonwood,Arizona Country=US ISP=CABLEONE ASN=AS11492 endpoint=66.164.137.1 pubkey=Hs4k...p8Q=
20261031-09:31:07 iface=wg0 peer=myphone City=Flagstaff,Arizona Country=US ISP=NorthernArizonaUniversity ASN=AS21582 endpoint=134.114.0.1 pubkey=xr4l...3TQ=
20261031-18:04:26 iface=wg0 peer=myphone City=Cottonwood,Arizona Country=US ISP=CABLEONE ASN=AS11492 endpoint=66.164.137.1 pubkey=xr4l...3TQ=
20261031-18:22:10 iface=wg0 peer=laptop City= Country= ISP= ASN= endpoint=10.9.10.5 pubkey=Lp2c...9wA=
```

With `asn` and `geoip` set to `0`:

```
20261031-05:06:02 started interval=2s ip_only=1 asn=0 geoip=0
20261031-05:09:33 iface=wg1 peer=basecamp endpoint=103.102.36.53 pubkey=8Nf/...m0=
20261031-07:12:48 iface=wg0 peer=wifephone endpoint=66.164.137.1 pubkey=gDTT...C2U=
20261031-08:05:19 iface=wg0 peer=daughterphone endpoint=66.164.137.1 pubkey=Hs4k...p8Q=
20261031-09:31:07 iface=wg0 peer=myphone endpoint=134.114.0.1 pubkey=xr4l...3TQ=
20261031-18:04:26 iface=wg0 peer=myphone endpoint=66.164.137.1 pubkey=xr4l...3TQ=
```

## Lookups

`asn` and `geoip` look up each new endpoint at [ipinfo.io](https://ipinfo.io) over HTTPS. Both are on by default, so every public peer address is sent to a third party, which may matter for GDPR or similar rules. Set both to `0` to keep addresses off the network.

- Spaces and `=` are removed from values, and common company suffixes (e.g. Inc, LLC, Ltd, Corp) are dropped from `ISP=`.
- Fields go between `peer=` and `endpoint=` in the order `City=`, `Country=`, `ISP=`, `ASN=`. `City=` is `city,region`.
- An enabled field is always printed, with a blank value when it is unknown (ipinfo has no data, private address, failed lookup), so `column -t` keeps the columns aligned.
- Private, CGNAT, loopback, link-local, multicast and ULA addresses, and any IPv6 outside 2000::/3, are never looked up.
- Results are cached in `/tmp` for 24 hours, IPv4 per address and IPv6 per /64, up to 500 entries.
- A failed lookup (no network, no HTTPS support, rate limit) logs one warning to syslog and pauses lookups for 60 seconds; affected lines get blank fields.
- Lookups run inline in the poll loop, so a slow or failing lookup (e.g. dead upstream DNS, ~10s) delays polling until it returns.
- The free ipinfo tier allows roughly 1000 lookups per day without a key.

Requires `uclient-fetch`, `jsonfilter`, `ca-bundle` and a `libustream-*` SSL library, all in default OpenWrt images. If `uclient-fetch` or `jsonfilter` is missing, both options are disabled with an error in syslog.

## Persistent logs

Syslog is RAM and is cleared on reboot. Optionally point `log_file` at persistent storage. If it is under an `/etc/config/fstab` mount, lines are only written while that mount is present.

## Pro tip: get an aligned log

Install the `column` package and query the log with it for a nicer aligned display:
```
% column -t /mnt/spare/wireguard-endpoint.log
20261031-05:09:33  iface=wg1  peer=basecamp       City=Kathmandu,BagmatiProvince  Country=NP  ISP=InternetExchangeNepal      ASN=AS17412  endpoint=103.102.36.53  pubkey=8Nf/...m0=
20261031-07:12:48  iface=wg0  peer=wifephone      City=Cottonwood,Arizona         Country=US  ISP=CABLEONE                   ASN=AS11492  endpoint=66.164.137.1   pubkey=gDTT...C2U=
20261031-08:05:19  iface=wg0  peer=daughterphone  City=Cottonwood,Arizona         Country=US  ISP=CABLEONE                   ASN=AS11492  endpoint=66.164.137.1   pubkey=Hs4k...p8Q=
20261031-09:31:07  iface=wg0  peer=myphone        City=Flagstaff,Arizona          Country=US  ISP=NorthernArizonaUniversity  ASN=AS21582  endpoint=134.114.0.1    pubkey=xr4l...3TQ=
20261031-18:04:26  iface=wg0  peer=myphone        City=Cottonwood,Arizona         Country=US  ISP=CABLEONE                   ASN=AS11492  endpoint=66.164.137.1   pubkey=xr4l...3TQ=
20261031-18:22:10  iface=wg0  peer=laptop         City=                           Country=    ISP=                           ASN=         endpoint=10.9.10.5      pubkey=Lp2c...9wA=
```

## Installation

See https://github.com/openwrt/packages/pull/31066

## Notes

- Changes faster than the poll interval can be missed (A to B to A within one interval).
- Mobile carriers commonly use CGNAT, so these endpoints change often.
- Rapid alternation between two endpoints for the same key suggests the key is in use on two devices at once.
- Site-to-site peers with a configured `endpoint_host` show that address before first contact; those lines do not represent an incoming connection.
