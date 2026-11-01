# WireGuard-Endpoint-Log

Small daemon that logs each change in the source address every WireGuard peer key pair connects from on OpenWrt, making it possible to spot a key pair being used by someone other than its owner.

## Rationale

WireGuard does not log connections. If a peer's private key is copied off a lost phone, a backup or a shared config, whoever has it can connect as that peer and nothing on the router easily shows that record. How do you know someone didn't connect overnight using a compromised key? **You don't. Until now.**

A typical home setup has a handful of peers: phones for family members, a laptop or tablet, maybe a remote site. Each one tends to connect from a small, predictable set of networks: the home ISP, a mobile carrier, work/school etc. all of which are in the same region. This log is an audit trail of every place each key pair has connected from, so the administrator can look back and check that the pattern holds.

A peer that suddenly connects from an unfamiliar ISP, a hosting provider or VPN, or a city/country its owner has never been to is a $\color{red}\textbf{huge red flag}$ that the key pair may be compromised. The log this generates allows for easy review of where the device's connection originated.

## Log format

Example log:
```
20261031-05:09:33 iface=wg1 peer=basecamp City=Kathmandu,BagmatiProvince Country=NP ISP=InternetExchangeNepal ASN=AS17412 endpoint=103.102.36.53 pubkey=8Nf/...m0=
20261031-07:12:48 iface=wg0 peer=wifephone City=Cottonwood,Arizona Country=US ISP=CABLEONE ASN=AS11492 endpoint=66.164.137.1 pubkey=gDTT...C2U=
20261031-08:05:19 iface=wg0 peer=daughterphone City=Cottonwood,Arizona Country=US ISP=CABLEONE ASN=AS11492 endpoint=66.164.137.1 pubkey=Hs4k...p8Q=
20261031-09:31:07 iface=wg0 peer=myphone City=Flagstaff,Arizona Country=US ISP=NorthernArizonaUniversity ASN=AS21582 endpoint=134.114.0.1 pubkey=xr4l...3TQ=
20261031-18:04:26 iface=wg0 peer=myphone City=Cottonwood,Arizona Country=US ISP=CABLEONE ASN=AS11492 endpoint=66.164.137.1 pubkey=xr4l...3TQ=
20261031-18:22:10 iface=wg0 peer=laptop City= Country= ISP= ASN= endpoint=10.9.10.5 pubkey=Lp2c...9wA=
```

`peer=` is the peer's `description` in `/etc/config/network`, or `unnamed`; public keys are shortened here. `myphone` at the university in the morning and home in the evening is a normal pattern. The same key pair on a hosting provider, or in another country, while its owner is home would not be.

With `asn` and `geoip` set to `0`:

```
20261031-05:09:33 iface=wg1 peer=basecamp endpoint=103.102.36.53 pubkey=8Nf/...m0=
20261031-07:12:48 iface=wg0 peer=wifephone endpoint=66.164.137.1 pubkey=gDTT...C2U=
20261031-08:05:19 iface=wg0 peer=daughterphone endpoint=66.164.137.1 pubkey=Hs4k...p8Q=
20261031-09:31:07 iface=wg0 peer=myphone endpoint=134.114.0.1 pubkey=xr4l...3TQ=
20261031-18:04:26 iface=wg0 peer=myphone endpoint=66.164.137.1 pubkey=xr4l...3TQ=
```
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


## Limits

- Changes faster than the poll interval can be missed (A to B to A within one interval).
- Address changes are logged, not connections: there is no line for connect, disconnect or duration, and returning to an earlier address is logged again. State is reset whenever the service starts or stops, so after a reboot, restart or config change every current endpoint is logged once more.
- A copied key pair used from the same address as its owner (same home Wi-Fi, same CGNAT gateway), or from an `ignore_cidr` range, produces no line.
- Mobile carriers use CGNAT, so phone endpoints change often and are shared with many other customers; judge them by `ISP=` and `ASN=`, not the address. Cellular addresses usually geolocate to the carrier's gateway city, which can be hundreds of km away. City data is approximate in general; country and ASN are more reliable.
- WireGuard only updates a peer's endpoint after an authenticated packet, so a logged address came from someone holding that key pair and cannot be spoofed.
- Site-to-site peers with a configured `endpoint_host` show that address before first contact; those lines do not represent an incoming connection.
- It only covers WireGuard, not other ways into the network.

## Lookups

`asn` and `geoip` look up each new endpoint at [ipinfo.io](https://ipinfo.io) over HTTPS, so an unfamiliar network stands out without manual lookups. Both are on by default, so every public peer address is sent to a third party, which may matter for GDPR or similar rules. Set both to `0` to keep addresses off the network.

- Spaces and `=` are removed from values, and common company suffixes (e.g. Inc, LLC, Ltd, Corp) are dropped from `ISP=`.
- `City=` is `city,region` (just the city if the region is unknown or the same).
- An enabled field is always printed, with a blank value when it is unknown (private address, no ipinfo data, or a failed lookup with nothing cached), so `column -t` keeps the columns aligned.
- Private, CGNAT, loopback, link-local, multicast and ULA addresses, and any IPv6 outside 2000::/3, are never looked up.
- Results are cached in `/tmp` for 24 hours.
- A failed lookup (no network, no HTTPS support, rate limit) logs one warning to syslog and pauses lookups for 60 seconds; affected lines reuse an expired cache entry if there is one.
- Lookups run inline in the poll loop, so a slow or failing lookup (e.g. dead upstream DNS, ~10s) delays polling until it returns.
- The free ipinfo tier allows roughly 1000 lookups per day without an API key.

Requires `uclient-fetch`, `jsonfilter`, `ca-bundle` and a `libustream-*` SSL library, all in default OpenWrt images. If `uclient-fetch` or `jsonfilter` is missing, both options are disabled with an error in syslog.

## Persistent logs

Syslog is RAM and is cleared on reboot, so the audit trail is lost unless `log_file` is set. Optionally point `log_file` at persistent storage. It only gets endpoint lines; startup, warnings and errors go to syslog only. If it is under an `/etc/config/fstab` mount, lines are only written while that mount is present.

## Pro tip: aligned output for easier reading

Install the `column` package and query the log with it for an aligned display:
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
