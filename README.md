# 🏠 Homelab

Documentation, configurations, and lessons learned from my self-hosted homelab. A three-node hybrid setup built around privacy, automation, and understanding systems from the ground up.

This repo serves three purposes: a personal reference so nothing lives only in my head, a resource for the self-hosting community, and a record of the architecture and operational decisions behind the lab.

> **Status:** Repository under construction. Folder structure and per-service configs are being migrated in. See [Roadmap](#roadmap) for where the lab itself is heading.

---

## Architecture at a Glance

The lab spans **three machines**, each with a distinct role:

```
┌─────────────────────────── Home Network ───────────────────────────┐
│                                                                    │
│   Router (DDNS + WireGuard port forward — only inbound path)       │
│      │                                                             │
│   Switch                                                           │
│      │                                                             │
│   ┌──┴──────────────────────┐     ┌───────────────────────────┐    │
│   │  Raspberry Pi (arm64)   │     │  Proxmox Host             │    │
│   │  SATA HAT + disks       │     │  Debian VMs, Kali Linux   │    │
│   │  Cosmos Cloud + Docker  │     │                           │    │
│   │  DHCP / DNS / domains   │     │                           │    │
│   └──────────┬──────────────┘     └───────────────────────────┘    │
│              │                                                     │
└──────────────┼─────────────────────────────────────────────────────┘
               │  netconsole (kernel logs, UDP)
               ▼
   ┌───────────────────────────┐
   │  Oracle Cloud VPS         │
   │  netconsole receiver      │
   │  NPM, Portainer,          │
   │  Cloudflare DNS, domain   │
   └───────────────────────────┘
```

### 🍓 Raspberry Pi — Primary self-hosting node

The workhorse. An arm64 Pi with a SATA HAT and attached disks (with redundancy), running **Cosmos Cloud** as the container management layer. It also handles internal networking: DHCP, DNS, and local domains.

**Services:**

| Service | Purpose |
|---|---|
| [Immich](https://immich.app/) | Photo & video management (self-hosted Google Photos alternative) |
| [Home Assistant](https://www.home-assistant.io/) | Home automation, HACS integrations, custom Lovelace dashboards |
| [Pi-hole](https://pi-hole.net/) | Network-wide DNS & ad blocking |
| [Uptime Kuma](https://github.com/louislam/uptime-kuma) | Service monitoring & status |
| [Crafty Controller](https://craftycontrol.com/) | Minecraft server management (Forge 1.20.1 + lazymc sleep/wake) |
| [WireGuard](https://www.wireguard.com/) | VPN — the only remote access path into the network |
| [ntfy](https://ntfy.sh/) | Push notifications |
| [OpenCloud](https://opencloud.eu/) | File sync & share |
| [Snapdrop](https://github.com/SnapDrop/snapdrop) | Local file sharing |
| Stremio| Media |
| Thunderbird | Mail |

### 🖥️ Proxmox — Virtualization host

Runs the VMs: Debian machines for general workloads and Kali Linux as a security/CTF environment.

### ☁️ Oracle Cloud VPS — Off-site node

Currently a lightweight satellite, **not** an ingress point for home services:

- **netconsole receiver** — the Pi ships kernel `printk` output here over UDP at boot, giving out-of-band crash visibility even if local disk logging fails
- Nginx Proxy Manager, Portainer
- Domain (DigitalPlat) + Cloudflare DNS

### 🔑 Remote Access — Current Model

Remote access is **WireGuard only**: DDNS on the home router plus a single forwarded WireGuard port. No home service is directly exposed to the internet.

---

## Roadmap

Planned changes, clearly separated from what runs today:

- [ ] **Headscale** on the Oracle VPS — self-hosted Tailscale control server to replace the port-forwarded WireGuard setup
- [ ] **Tang server** migration from Proxmox to Oracle — network-bound disk encryption (NBDE) so encrypted disks unlock only when the trusted network is reachable
- [ ] **Edge routing** — evaluate fronting selected services through Cloudflare → Oracle → NPM instead of VPN-only access
- [ ] Harden the netconsole path (restrict UDP ingress on OCI to known source ranges)

---

## Planned Repository Structure

```
homelab/
├── docs/            # Architecture, network design, decisions (ADRs), lessons learned
├── pi/              # Hardware, networking, and Cosmos container stacks
├── proxmox/         # VM notes and templates
├── oracle-cloud/    # VPS services & netconsole receiver
└── scripts/         # Backup & maintenance automation
```

Each service folder will contains `docker-compose.yml`, a `.env.example` with placeholders wherever aplicable, and a README covering instructions and quirks.
