# 🔐 Network Bound Disk Encryption — Tang/Clevis Auto-Unlock for Encrypted Data Disks

Network-Bound Disk Encryption: LUKS2 disks that unlock automatically **only when the Tang server is reachable**.

## 1. Prepare the Drive

```bash
# 1) Wipe filesystem signatures
sudo wipefs -af /dev/sdX

# 2) Encrypt (LUKS2 — required for Clevis token metadata)
sudo cryptsetup luksFormat --type luks2 /dev/sdX

# 3) Unlock
sudo cryptsetup open /dev/sdX <label>

# 4) Format (ext4)
sudo mkfs.ext4 -L <label> /dev/mapper/<label>

# 5) Lock again
sudo cryptsetup close <label>
```

## 2. Bind to the Tang Server

```bash
# 1) Install Clevis
sudo apt install -y clevis clevis-luks clevis-systemd

# 2) On the TANG SERVER, get the signing-key thumbprint
sudo tang-show-keys 80        # (port tangd listens on)

# 3) Bind, pinning the expected key
sudo clevis luks bind -d /dev/sdX tang '{"url":"http://<TANG_IP>","thp":"<THUMBPRINT>"}'

# 4) Verify the binding
sudo clevis luks list -d /dev/sdX
sudo cryptsetup luksDump /dev/sdX

# 5) Test unlock via Tang, then close again
sudo clevis luks unlock -d /dev/sdX -n <label>
sudo cryptsetup close <label>

# 6) Back up the LUKS header
sudo cryptsetup luksHeaderBackup /dev/sdX --header-backup-file <label>-luks-header.img
```

## 3. crypttab & fstab

```bash
# 1) Find the UUID — use the OUTER crypto_LUKS UUID, not the inner ext4 one
lsblk -o NAME,FSTYPE,UUID,SIZE,MOUNTPOINT,LABEL
```

```bash
# 2) /etc/crypttab entry
#    NOTE: add "discard" to the options if the device is an SSD
<label>  UUID=<LUKS-UUID>  none  luks,_netdev,x-systemd.device-timeout=60s

# 3) /etc/fstab entry
#    NOTE: add "discard" to the options if the device is an SSD
/dev/mapper/<label>  /mnt/<label>  ext4  defaults,_netdev,x-systemd.mount-timeout=60s  0  2
```

```bash
# 4) Reload so the generator creates the units
sudo systemctl daemon-reload

# 5) Confirm the generated units exist
systemctl list-units "systemd-cryptsetup@*"

# 6) Belt-and-suspenders: make sure the Clevis askpass responder is up before this device's unlock prompt appears
sudo systemctl edit systemd-cryptsetup@<label>.service
```

```ini
[Unit]
Requires=clevis-luks-askpass.path
After=clevis-luks-askpass.path
```

## 4. Boot Ordering

```bash
# 1) Network must be genuinely up before unlock
sudo systemctl enable NetworkManager-wait-online.service      # NetworkManager
# OR
sudo systemctl enable systemd-networkd-wait-online.service    # systemd-networkd

# 2) Enable the Clevis askpass responder
sudo systemctl enable clevis-luks-askpass.path

# 3) Enable the target that governs _netdev encrypted devices.
sudo systemctl enable remote-cryptsetup.target

# 4) Make Docker WAIT for the data mounts, but start even if they fail
#     (Pi-hole & Wireguard live on the boot SSD. The LAN's DNS and remote access must come up regardless.
#     Wants= not Requires=: waits for mounts to settle, tolerates their failure.
#     The disk-safe named volumes then fail closed for any affected container — see docker.md.)
sudo systemctl edit docker.service
```

```ini
[Unit]
Wants=mnt-<label1>.mount mnt-<label2>.mount
After=mnt-<label1>.mount mnt-<label2>.mount
```

## 5. Reboot & Verify

```bash
sudo systemctl daemon-reload
sudo systemctl daemon-reexec
sudo reboot

# After boot:
systemctl list-units "systemd-cryptsetup@*"     # all active?
systemd-analyze blame | grep cryptsetup         # unlock cost
systemd-analyze critical-chain                  # what waited on what
lsblk                                           # mappers open, mounts present
```

## Maintenance

**Tang key rotation** (deliberate, never automatic — Tang's keys are files in `/var/db/tang` and survive reboots forever). The correct sequence never breaks unlocks:

```bash
# On the TANG SERVER:
# 1) Generate new keys
sudo /usr/libexec/tangd-keygen /var/db/tang        # path varies by distro
# 2) HIDE the old keys (dot-prefix): no longer advertised, still answer recovery for existing bindings
sudo mv /var/db/tang/<oldkey>.jwk /var/db/tang/.<oldkey>.jwk   # each old key

# On EVERY CLIENT, per bound device (slot from `clevis luks list`):
sudo clevis luks regen -d /dev/sdX -s <slot>

# Only after ALL clients are regenerated: delete the hidden old keys.
```

> [!WARNING]
> Deleting old Tang keys **before** regenerating every client drops every bound disk to passphrase-at-boot simultaneously.

**After any keyslot change** (bind, regen, passphrase change): refresh the header backup.

**Disaster matrix:**

| Situation | What happens | What to do |
|---|---|---|
| Tang unreachable at boot | Unlock waits, then boot continues without the mounts (timeouts are 60s) | Fix Tang, then reboot host |
| Tang permanently dead | Auto-unlock gone | Unlock with the keyslot-0 passphrase, stand up a new Tang, re-bind |
| Header corrupted | Disk won't open at all | `cryptsetup luksHeaderRestore` from the backup image |
| Disk/Pi stolen | Attacker is off the trusted network or create automations to deliberately shut down Tang | Data is passphrase-only |

## Design Note (roadmap)

NBDE's security property is "unlocks only on the trusted network." Moving Tang to a **publicly reachable** VPS would dissolve that property. A stolen Pi would unlock from any internet connection. If Tang migrates to Oracle, it must be reachable **only by the home external IP**. To be decided in an ADR before any migration.
