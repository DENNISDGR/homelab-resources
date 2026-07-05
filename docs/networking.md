# Networking

How the lab's network fits together. The [Pi](../pi/README.md) is the network's infrastructure node. **DHCPv4**, **DNS** (Pi-hole), and **local domain resolution** for services behind Cosmos. The ISP router keeps the jobs it refuses to share: gateway, **IPv6 (SLAAC/RA + DHCPv6)**, DDNS, and the single WireGuard port forward.

## Addressing Plan

| Thing | Value |
|---|---|
| LAN subnet | `192.168.X.0/24` |
| Router / gateway | `192.168.X.1` |
| Pi (static) | `192.168.X.2` |
| Proxmox host | `192.168.X.3` |
| DHCPv4 pool | `192.168.X.150 – .254` — `.2 – .149` deliberately reserved for static assignments |
| DHCPv4 lease time | 24 h (default) |

## DHCP — Split Stack

DHCP duties are deliberately split between two devices:

| Protocol | Served by | Why |
|---|---|---|
| **DHCPv4** | Pi-hole (built-in DHCP server; router's v4 DHCP disabled) | Puts lease management and per-client DNS assignment in one place, next to the resolver |
| **IPv6 (SLAAC/RA + DHCPv6)** | ISP router | The ISP router misbehaved when other devices participated in RA/SLAAC. It insists on owning IPv6 address configuration, so it keeps it |

**The classic caveat of this split, IPv6 DNS bypass, is handled.** Because the router owns IPv6 configuration, it also advertises the IPv6-side DNS, which by default points at itself and bypasses Pi-hole. In my case the router's DHCPv6 DNS delegation can be set to **Manual**. Then I set it so Pi's IPv6 address is advertised first, with Cloudflare's v6 resolvers (`2606:4700:4700::1111` / `::1001`) as fallback.

## DNS Flow

```
LAN clients
   │  (DHCPv4 hands out the Pi as sole v4 DNS server)
   ▼
Pi-hole ── blocklists + local records ──▶ answers local + filtered queries
   │
   ▼
Upstream: Cloudflare (1.1.1.1)
```

### Local domains

Services are reached as **`<service>.cosmos.local`** (e.g. `drive.cosmos.local`, `dns.cosmos.local`).

| Question | Answer |
|---|---|
| Domain scheme | `<service>.cosmos.local` |
| Where records live | Pi-hole local/wildcard DNS records → all `*.cosmos.local` names resolve to the Pi |
| Who routes by hostname | Cosmos' buit in reverse proxy, no NginX or Trafeik shenanigans. One IP, many services, routed on the requested name |
| TLS for local services | Self-managed **Root CA**. Service certificates are issued from it and supplied to Cosmos's config. Clients trust the Root CA once and every `*.cosmos.local` service gets valid HTTPS |

## Remote Access

**WireGuard only.** No home service is directly exposed to the internet.

```
Internet ──▶ DDNS name ──▶ Router (single forwarded WG port) ──▶ WireGuard on the Pi ──▶ LAN
```

- DDNS: **No-IP** free tier, updated natively by the router's built-in DDNS client. Operational caveat: free No-IP hostnames must be **manually confirmed every 30 days** or they expire.
- The router forwards exactly one UDP port to the Pi's WireGuard container
- VPN clients are handed the LAN DNS server, so filtering and `*.cosmos.local` work on the road

> Roadmap: replace this with Headscale on the Oracle VPS — see the [docs README](../README.md#roadmap).

## netconsole

The Pi ships kernel `printk` output over UDP to the Oracle Cloud VPS at boot. This gives out-of-band visibility into kernel panics, crashes and early logs in general.

Loaded at boot as a module:

```bash
# /etc/modules — append the module name
netconsole

# /etc/modprobe.d/netconsole.conf
options netconsole netconsole=6665@<PI_LAN_IP>/eth0,6666@<ORACLE_PUBLIC_IP>/<GATEWAY_MAC>
```

Syntax: `netconsole=[src-port]@[src-ip]/[dev],[tgt-port]@<tgt-ip>/[tgt-macaddr]`. Because the receiver is off-subnet, the MAC field must be the **default gateway's MAC** (find it with `ip neigh` after pinging the gateway), not the receiver's.

On the source fields: `6665` is the documented default source port (explicit here for clarity). The explicit **source IP** is load-bearing, netconsole builds its packets via netpoll and netpoll refuses to start if it can't determine a local IP. Hardcoding it means the module comes up cleanly even if it loads before the interface has been assigned its address.

**Limits to remember**:

- **Capture starts at module load, not at power-on.** Netconsole depends on the NIC driver, so it cannot see anything earlier than network init.
- netpoll constructs raw UDP frames at the driver level (IRQ-safe, no routing, no ARP) which is why it survives panics, and why only IP-over-UDP on ethernet devices is supported (no tunneling over WireGuard)
- Plain UDP, unencrypted and unauthenticated. Kernel logs cross the internet in cleartext. Mitigation is source-IP restriction on the Oracle side
- Verify it works with a harmless test: `echo "<0>netconsole test $(date +%s)" | sudo tee /dev/kmsg` and watch the receiver

## Failure Modes

Because DHCPv4 and all DNS live on one machine (the Pi):

| Pi state | Effect on LAN |
|---|---|
| Down briefly | Existing v4 clients keep working on cached leases. New joins and lease renewals fail. **All DNS resolution fails immediately** (IPv6 addressing continues via the router, but names still don't resolve) |
| Down for long | v4 leases expire → clients drop off the network entirely |

Mitigations in place / to document:

- [x] DHCPv4 lease time: 24h — comfortably rides out reboots and short maintenance windows
- [ ] Manual fallback: static IP + public DNS on a client to reach the router/Proxmox when the Pi is down
- [ ] TODO: secondary Pi-hole DNS server e.g. on proxmox 

## Segmentation

At this point you'd expect I also went into VLANs but I have no managed switch and my ISP router doesn't support LAN Segmentation :(

On paper though, untrusted devices with weak or absent authentication shouldn't share a segment with trusted infrastructure. Until that's enforceable, the honest description of the current mitigation is perimeter-only: the LAN's entry points are WireGuard and physical presence, both trusted.
