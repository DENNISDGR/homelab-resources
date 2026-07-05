# 🐳 Docker & Cosmos

Conventions and how-tos for how containers are run in this lab.

This file is the **single reference** for recurring Docker procedures.

## The Config Format

The JSON files in `pi/cosmos-containers/` are **Cosmos' compose-like JSON dialect** (from its "Edit as JSON" view), not standard Docker Compose. YAML 1.2 is a superset of JSON, but these files won't run under `docker compose` as-is. The giveaways:

- Volume mounts use capitalized `Type`/`Source`/`Target` keys (Docker Engine API style; Compose long syntax is lowercase)
- The `routes` field is pure Cosmos — its reverse-proxy routing lives there
- `networks` and `network_mode` both appear, a combination vanilla Compose rejects

**To reuse these configs:** paste into Cosmos's JSON editor, or translate to standard `docker-compose.yml` for plain Docker.

## How-To: Disk-Safe Bind Volumes

**Create bind-backed *named* volumes directly with Docker, not as plain bind mounts in Cosmos.**

```bash
sudo docker volume create --driver=local --opt type=none --opt o=bind --opt device=/<path> <volume_name>
```

Then reference the volume **by name** in the container config, like any named volume.

**Why this instead of a plain bind mount:**

- **If the backing disk is missing, the container refuses to start (or stops) instead of writing garbage.** A classic bind mount to an unmounted path can be silently auto-created on the root filesystem — the container then happily fills the boot disk with data that lands in the wrong place and shadows the real dataset when the disk comes back. With a named volume wrapping the bind, Docker fails the mount outright when the path doesn't exist.
- **Cosmos treats it as a normal named volume.** The data lives exactly where you want it on the data disks, while the config stays clean and portable.

## How-To: LAN DNS for All Containers

Make every container resolve through the local DNS server (e.g. Pi-hole) by configuring the Docker daemon itself, instead of per-container `dns:` entries.

Create or edit `/etc/docker/daemon.json`:

```json
{
  "dns": ["<your_lan_dns_ip>", "<public_fallback_dns>"]
}
```

Then restart the daemon — **note this restarts all running containers**:

```bash
sudo systemctl restart docker
```

Notes:

- Containers on user-defined networks resolve through Docker's embedded DNS (`127.0.0.11`); this setting controls where the embedded resolver forwards *external* lookups — so it covers every container, both default-bridge and custom networks.
- The public fallback (e.g. `1.1.1.1`) is a resilience/consistency tradeoff: containers keep resolving if the LAN DNS is down, at the cost of those queries bypassing its filtering. Drop the fallback for strict sinkhole enforcement.
- Per-container `dns:` entries in a config override this daemon-wide setting.
