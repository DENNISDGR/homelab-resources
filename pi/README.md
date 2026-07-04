# 🍓 Raspberry Pi - Primary Self-Hosting Node

The workhorse of the lab. A Raspberry Pi 5 (16 GB) with a Radxa Penta SATA HAT and four disks, running **Cosmos Cloud** as the container management and reverse-proxy layer. Besides hosting all self-hosted services, it is also the network's infrastructure node: DHCP, DNS, and local domain resolution all live here.

> If this machine is down, the LAN loses DHCP and DNS. See [../docs/networking.md](../docs/networking.md#failure-modes) for what that means in practice.

All container stacks live under [cosmos/](cosmos/), one folder per service with its `docker-compose.yml`, an `.env.example` where applicable, and notes.

---

## Services

All containers are managed through Cosmos Cloud.

| Service | Purpose | Stack |
|---|---|---|
| Immich | Photo & video management | [cosmos/immich/](cosmos/immich/) |
| Home Assistant | Home automation, HACS, Lovelace dashboards | [cosmos/home-assistant/](cosmos/home-assistant/) |
| Pi-hole | Network-wide DNS & ad blocking | [cosmos/pihole/](cosmos/pihole/) |
| Uptime Kuma | Service monitoring & status | [cosmos/uptime-kuma/](cosmos/uptime-kuma/) |
| Crafty Controller | Minecraft server management (Forge 1.20.1 + lazymc) | [cosmos/crafty/](cosmos/crafty/) |
| WireGuard | VPN, the only remote access path | [cosmos/wireguard/](cosmos/wireguard/) |
| ntfy | Push notifications | [cosmos/ntfy/](cosmos/ntfy/) |
| OpenCloud | File sync & share | [cosmos/opencloud/](cosmos/opencloud/) |
| Snapdrop | Local file sharing | [cosmos/snapdrop/](cosmos/snapdrop/) |
| Stremio | Media | [cosmos/stremio/](cosmos/stremio/) |
| Thunderbird | Mail | [cosmos/thunderbird/](cosmos/thunderbird/) |

---

## Hardware

### Platform

| Component | Detail |
|---|---|
| Board | Raspberry Pi 5, 16 GB RAM (arm64) |
| Storage HAT | [Radxa Penta SATA HAT](https://radxa.com/products/accessories/penta-sata-hat/), attaches via the Pi 5's PCIe FFC connector |
| Boot device | Samsung 500 GB SSD over a SATA→USB 3.0 adapter (Even though the HAT has 5 SATA ports Raspberry Pis don't support booting from SATA) |
| PSU | 12 V / 8 A (96 W) barrel connector into the HAT that powers the Pi and all peripherals below |
| Cooling | be quiet! Pure Wings 2 80 mm (3-pin), fed from the HAT via a Digitus 4-pin Molex→Molex adapter (AK-430302-002-M) |
| Switch | 5-port gigabit unmanaged desktop switch, shell removed, power leads soldered directly to the HAT's power pinout <!-- TODO: which pins the switch is fed from --> |
| Wi-Fi (dormant) | TP-Link TL-WN8200ND, shell removed, connected via USB 2.0 |
| USB hub | Generic 4 port USB 3.0 hub. Extends the Pi's USB ports to the case exterior, so ports stay accessible without opening the enclosure |
| Case | Custom 3D-printed enclosure housing the whole stack, Pi + HAT + disks + switch + antenna <!-- TODO: add STL files --> |

> Design note: this is deliberately a **single-box network appliance**, compute, storage, switching, and (eventually) Wi-Fi all live in one 3D-printed enclosure powered from a single 96 W supply through the HAT.

### Disks & Redundancy

| Disk | Capacity | Role |
|---|---|---|
| Western Digital Red | 4 TB | Data |
| Western Digital Blue | 1 TB | Data |
| Seagate Barracuda | 500 GB | Data |
| Western Digital Red | 4 TB | **Parity** |

**Redundancy scheme:** managed through Cosmos Cloud's **Storage → Parity** tab ([Cosmos storage docs](https://cosmos-cloud.io/docs/storage/)). Under the hood this is snapshot-based parity over independent disks (SnapRAID-style), optionally pooled under one mount point with MergerFS — not striped RAID. Key properties of this design:

- **Parity is snapshot-based, not real-time.** Parity is recalculated on a scheduled *sync*, so data written between syncs is unprotected until the next sync runs. Fine for mostly-static data (media, photos); the sync cadence is the recovery-point objective. <!-- TODO: document the sync schedule -->
- **No striping.** Every disk keeps its own filesystem. If a disk dies beyond parity's ability to recover, only that disk's data is lost.
- **Parity disk must be ≥ the largest data disk.** Note this is the *minimum*, with parity and largest-data equal-sized, keep some free-space margin on the big data disk or parity syncs can run out of room.
- **Recovery procedure:** replace the failed disk, then Cosmos **Storage → Parity → select the array → Fix** and wait for the rebuild.
- **Parity is not backup.** It protects against disk failure, not deletion, corruption of the array host, fire, or theft. <!-- TODO: link backup strategy once documented in scripts/backups/ -->
