# Pi 5 Single-Drive Backup NAS — Ansible Playbook

Automates the software side of a single-drive Raspberry Pi 5 NAS (built for
use with a PCIe SATA HAT, e.g. Radxa's Penta SATA HAT, but works with any
single block device — USB or PCIe). Handles boot/PCIe config, storage
(ZFS or ext4), and Samba/NFS sharing.

## Before you run this

1. **Know your disk device.** SSH into the target Pi and run `lsblk`.
   Set `nas_disk_device` in `group_vars/nas.yml` accordingly
   (e.g. `/dev/sda`). **This playbook will format that disk — double,
   triple check it's the right one.** Don't point it at your SD card / boot
   drive.
2. **Edit `inventory/hosts.ini`** with your Pi's IP/hostname and SSH user.
3. **Review `group_vars/nas.yml`** — in particular:
   - `storage_backend`: `zfs` (default, recommended) or `ext4` (simpler)
   - `samba_users` / `samba_shares`: adjust names and paths as you like
   - `enable_nfs`: set `true` if you also want NFS exports

## First-time setup (on your control machine, i.e. your laptop, not the Pi)

```bash
# Install ansible if you don't have it
pip install --user ansible

# Install required collections (needed for ext4 filesystem/mount modules)
ansible-galaxy collection install -r requirements.yml

# Test connectivity
ansible nas -m ping
```

## Running it

```bash
ansible-playbook site.yml
```

You'll be prompted (once, hidden input) to set a Samba password for each
user the first time it runs — after that it's idempotent and won't
re-prompt for users who already have a password set.

If `pcie_force_gen3` is enabled, the Pi will reboot partway through to
apply the boot config change, then Ansible will reconnect automatically.

## Testing on your existing Pi first

Since you don't have the SATA HAT/drive yet, you can safely test everything
except the storage role against your current Pi:

```bash
ansible-playbook site.yml --tags "boot_config,samba" --skip-tags storage
```

Or just point `nas_disk_device` at a spare USB stick to dry-run the full
storage flow safely before your real drive arrives.

## Notes / things to sanity-check yourself

- **Single disk = no redundancy.** ZFS mode gives you checksums and weekly
  scrubs to catch bitrot, but a drive failure still means data loss. Treat
  this NAS as *a* backup target, not your only copy of anything irreplaceable.
- `pcie_force_gen3` is a per-HAT gamble — Jeff Geerling's testing found some
  HAT/cable combos are flaky at Gen 3. If you get PCIe errors in `dmesg`
  after the reboot, set it back to `false`.
- Samba write speeds will bottleneck around 90-120 MB/sec over 1 Gbps
  Ethernet regardless of backend — that's a network ceiling, not something
  this playbook can fix.
- This playbook doesn't touch networking (static IP, 2.5G HAT config, etc.)
  or monitoring/SMART alerts — happy to add roles for either once you've
  got hardware and know what you want there.

## Structure

```
site.yml                    # main playbook
group_vars/nas.yml          # all your config knobs live here
inventory/hosts.ini          # target host(s)
roles/
  pi_boot_config/           # config.txt / PCIe tweaks
  storage/                  # ZFS or ext4 setup on the single drive
  samba/                    # Samba install, users, shares
  nfs/                      # optional NFS exports
```
