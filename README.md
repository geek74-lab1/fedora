
## 0. Read First: Btrfs, Snapper and the Subvolume Map

This guide is written for one machine: the HP ZBook 15 G2 described above, running Fedora 44 KDE on UEFI with Intel graphics only. The commands assume Secure Boot, AHCI, Intel VT-x, a 120 GB SSD and ZRAM. If your hardware differs, stop at the section that differs instead of forcing the command.

Snapper is the only snapshot system in this guide. It snapshots the root subvolume `@`. The separate subvolumes below are deliberately outside `@`, so root snapshots stay small and a rollback does not overwrite logs, caches, temporary files or user data.

### 0.1 Subvolume map

| Mount point | Subvolume | Included in root snapshots? | Why |
|---|---|---:|---|
| `/` | `@` | Yes | System state: `/usr`, `/etc`, package database, most of `/var` |
| `/home` | `@home` | No | User files; back them up separately |
| `/.snapshots` | `@snapshots` | No | Snapshot storage; must survive a rollback |
| `/var/cache` | `@var_cache` | No | Rebuildable package and application cache |
| `/var/log` | `@var_log` | No | Logs should persist across rollbacks, and they are not system state you need to restore |
| `/var/spool` | `@var_spool` | No | Mail, cron and print queues; runtime data |
| `/var/tmp` | `@var_tmp` | No | Persistent temporary files; can be large and is not worth snapshotting |
| `/boot` | ext4, not Btrfs | No | Kernel and initramfs; back up separately |
| `/boot/efi` | vfat, not Btrfs | No | UEFI System Partition |

`/tmp` is already a tmpfs on Fedora, so it is not part of the Btrfs root and does not need a subvolume.

The root Snapper configuration only snapshots `@`. Btrfs snapshots do not recurse into nested or separately mounted subvolumes, so the four `/var` subvolumes and `@home` are excluded automatically. This is the intended design, not an accident.

If you later decide to isolate more data, create the subvolume and add its `/etc/fstab` entry **before** you rely on snapshots. Do not move live data into a new subvolume casually. The four `/var` subvolumes above are the only extra ones this guide requires.

If you use Docker, Podman or Flatpak heavily, consider additional subvolumes for `/var/lib/docker`, `/var/lib/containers` and `/var/lib/flatpak`; create them before the first snapshot and add their fstab entries. They are optional here because the required four already remove the fastest-growing cache and log data.

### 0.2 Rules that prevent most failures

- **Back up first.** The old data is worth more than the installation. Section 1.1 lists what to copy.
- **Use the exact names in this guide.** The root subvolume is `@`, home is `@home`, snapshots are `@snapshots`, and the four excluded subvolumes are `@var_cache`, `@var_log`, `@var_spool` and `@var_tmp`. The rollback script expects these names.
- **Never rename a live root subvolume.** If the installer produced `root`/`home` instead, stop and reinstall with this layout. Do not rename it in place and do not try to repair it from a running system.
- **Use Snapper only.** Do not install a second snapshot manager or run two snapshot schedules. Duplicate snapshots waste the 120 GB SSD and make rollback ambiguous.
- **`/boot` is not inside the root snapshot.** Keep a separate `/boot` backup and at least two kernels installed. A root rollback restores the system files, not the kernel in the ext4 `/boot`.
- **ZRAM only.** Do not create a swap partition or swap file, do not add `resume=`, and do not configure hibernation. Suspend-to-RAM works; hibernation needs a disk swap area and is outside this guide.
- **Intel only.** Use the `i965` VA-API driver. Do not install `intel-media-driver`, NVIDIA, CUDA, ROCm or AMDGPU packages.
- **Compression is optional, but verify it.** `compress=zstd:1,noatime` is appropriate for `@` and `@home`. Edit `/etc/fstab` explicitly; a blind `sed` can miss tabs or duplicate options. Run `findmnt --verify`, remount, and inspect `findmnt -no TARGET,OPTIONS /` before rebooting.
- **Use the guide's account name or substitute consistently.** The commands use the username `user` and UID/GID `1000`. If your account is different, replace `user` and adjust the UID/GID in the NTFS and Samba steps.
- **Take a snapshot before kernel, driver, Plasma or large package changes.** Use the manual command in Section 2.5 or enable the optional DNF5 hook in Section 2.3.
- **The native `snapper rollback` command is not used with this layout.** It changes the Btrfs default subvolume, but `/etc/fstab` mounts `subvol=@` explicitly, so the change is ignored at boot. Use the `snapper-rollback` script in Section 2.6.

### 0.3 Follow this order

1. **Section 1:** back up, update firmware, make the installer, install Fedora with the Snapper subvolume layout, then verify the mounts on first boot.
2. **Section 2:** install Snapper, create the root configuration and baseline snapshot, set retention, and optionally enable the DNF5 hook.
3. **Section 3:** configure DNF5, update Fedora, update firmware, set hostname/timezone, mount the NTFS data disk, reboot.
4. **Section 4:** enable RPM Fusion, Flathub, OpenH264, Chrome, Brave, VS Code, Docker and Temurin.
5. **Sections 5-17:** graphics, multimedia, security, shell, development, languages, pentest, wireless, networking, virtualization, desktop apps, fonts, vendor installs.
6. **Section 18:** tune the kernel, enable only the services you use, verify.
7. **Sections 19-22:** KDE settings, migrate data and dotfiles, install desktop launchers.
8. **Sections 23-25:** run the verification checklist, use troubleshooting, read the FAQ.
9. **Appendix D:** 100 essential Fedora commands for daily use.

If a **Verify** block fails, stop and fix it before continuing. Do not continue with a broken boot layout, a missing `/.snapshots` mount or a snapshot configuration that lives inside `@`.

## 1. Prepare and Install Fedora 44

This is the installation path for the Snapper layout. Complete it in order. The backup comes before any partitioning; the subvolume map is verified before Snapper is configured.

### 1.1 Back up the old machine

Copy these from the old disk. They contain work, credentials or multi-gigabyte downloads that are expensive to recreate:

```bash
# Run on the old machine, with the backup disk mounted at /media/BACKUP.
rsync -a --info=progress2 \
  /home/user/THE_DATABASE \
  /home/user/github-tools \
  /home/user/Android \
  /home/user/.android \
  /home/user/.ssh \
  /home/user/.gnupg \
  /home/user/.gitconfig \
  /home/user/.mozilla \
  /home/user/.thunderbird \
  /home/user/.BurpSuite \
  /home/user/.ollama \
  /media/BACKUP/
```

Also back up browser profiles, `~/.config/Code/User`, any `~/.var/app` data you need, and your own scripts. Do **not** copy the whole of `~/.config`, `~/.cache`, old kernels, old VirtualBox images or MX/Debian system files.

Before repartitioning, confirm Windows is fully shut down, not hibernated and not using Fast Startup. A dirty NTFS volume will refuse to mount or can be damaged.

### 1.2 Firmware and installer media

1. Set **UEFI native mode** (no CSM/Legacy), **Secure Boot enabled**, **Intel Virtualization Technology (VT-x) enabled**, and **SATA mode AHCI**. Keep the internal drive as the first boot device.
2. Download the Fedora 44 KDE x86_64 Live image from <https://fedoraproject.org/kde/download/>. Verify its checksum against Fedora's official checksum file.
3. Write the image with Fedora Media Writer or Ventoy. Boot it in UEFI mode with Secure Boot enabled; Fedora's kernel and drivers are signed. Do not use a legacy/BIOS boot for this layout.

### 1.3 Partition layout for Snapper

Use **Custom** partitioning, not Automatic. The required result is:

| Mount | Size | Filesystem | Subvolume name | Notes |
|---|---:|---|---|---|
| `/boot/efi` | 1 GiB | EFI System Partition (FAT32) | — | Shared ESP is fine |
| `/boot` | 1 GiB | ext4 | — | Kernel and initramfs; not part of the root snapshot |
| `/` | remaining SSD | Btrfs | `@` | Root; the only subvolume Snapper snapshots |
| `/home` | shared with `/` | Btrfs | `@home` | User data; excluded from root snapshots |
| `/.snapshots` | shared with `/` | Btrfs | `@snapshots` | Snapshot storage; must be outside `@` |
| `/var/cache` | shared with `/` | Btrfs | `@var_cache` | Excluded from snapshots |
| `/var/log` | shared with `/` | Btrfs | `@var_log` | Excluded from snapshots |
| `/var/spool` | shared with `/` | Btrfs | `@var_spool` | Excluded from snapshots |
| `/var/tmp` | shared with `/` | Btrfs | `@var_tmp` | Excluded from snapshots |
| NTFS data disk | unchanged | NTFS | — | Do not format; it is the 466 GB disk |

There is no swap partition, swap file or `@swap` subvolume. Fedora 44's `zram-generator-defaults` provides compressed RAM swap; Section 7.4 verifies it. Hibernation is not configured because ZRAM cannot hold a hibernation image.

Important details:

- **`@snapshots` must be outside `@`.** Mount it at `/.snapshots`. If Snapper writes snapshots inside `@`, the snapshots become part of the root subvolume and the rollback script cannot find them at `@snapshots`.
- **The four `/var` subvolumes are the point of this layout.** `/var/cache`, `/var/log`, `/var/spool` and `/var/tmp` change constantly, can grow without limit and do not need to be rolled back. Keeping them in separate subvolumes makes root snapshots smaller and more useful.
- **Keep `/boot` as a separate ext4 partition.** Fedora 44's installer and GRUB are supported with this layout, and it avoids the UEFI `/boot`-on-Btrfs boot problems that still affect custom Workstation installs. Snapper does not snapshot `/boot`; keep a separate backup.
- **Use `compress=zstd:1` and `noatime` on `@` and `@home`.** On the excluded subvolumes, `noatime` is enough. Optionally add `compress=zstd:1` to `@var_log` because logs compress well. If the installer already set them, do not duplicate the options.
- **Edit `/etc/fstab` explicitly.** Do not use a blind `sed`: tab-separated lines and existing options are easy to break. Run `sudo findmnt --verify --verbose`, remount the affected mounts, and confirm with `findmnt -no TARGET,OPTIONS MOUNTPOINT` before rebooting.
- **Do not use `subvolid=` entries.** Mount by subvolume name so the layout stays readable and the rollback script can find `@`.
- **The exact names matter.** This guide fixes `@` as the root name because the rollback script and the `/etc/fstab` examples use it. Do not rename the live root later.

### 1.4 Installer steps

1. In **Installation Destination**, choose **Custom** (not Automatic). Select the 120 GB SSD.
2. Create the 1 GiB EFI System Partition mounted at `/boot/efi`.
3. Create a 1 GiB ext4 partition mounted at `/boot`.
4. Create one Btrfs partition using the rest of the SSD. Leave its mount point empty for the moment.
5. Create a subvolume named `@` mounted at `/`.
6. Create `@home` mounted at `/home`.
7. Create `@snapshots` mounted at `/.snapshots`.
8. Create `@var_cache` mounted at `/var/cache`.
9. Create `@var_log` mounted at `/var/log`.
10. Create `@var_spool` mounted at `/var/spool`.
11. Create `@var_tmp` mounted at `/var/tmp`.
12. Do not add a swap partition or a swap file. Fedora will use ZRAM after boot.
13. Set the bootloader to the internal SSD. Use UEFI, not BIOS.
14. Create the user `user`, set the hostname to `fedora44`, and select the Africa/Cairo timezone. Set a root password or use the default sudo setup, but keep the `user` account as the daily account.
15. Apply the changes, confirm the summary, and install. Reboot only after the installer finishes.

The exact button labels vary slightly between Anaconda versions. The required result is what matters: `@` on `/`, `@home` on `/home`, `@snapshots` on `/.snapshots`, the four `/var` subvolumes, a separate ext4 `/boot`, a separate ESP, and no swap partition or swap file. If the live installer's storage editor does not expose these subvolume or mount-point fields, use the Fedora 44 Everything/netinstall image with Blivet; do not let automatic partitioning choose `root`/`home` and then rename it later.

### 1.5 First boot: verify the layout before installing anything

```bash
findmnt -no TARGET,SOURCE,FSTYPE,OPTIONS /
findmnt -no TARGET,SOURCE,FSTYPE,OPTIONS /home
findmnt -no TARGET,SOURCE,FSTYPE,OPTIONS /.snapshots
findmnt -no TARGET,SOURCE,FSTYPE,OPTIONS /var/cache
findmnt -no TARGET,SOURCE,FSTYPE,OPTIONS /var/log
findmnt -no TARGET,SOURCE,FSTYPE,OPTIONS /var/spool
findmnt -no TARGET,SOURCE,FSTYPE,OPTIONS /var/tmp
findmnt -no TARGET,SOURCE,FSTYPE,OPTIONS /boot
findmnt -no TARGET,SOURCE,FSTYPE,OPTIONS /boot/efi
sudo btrfs subvolume list -a /
sudo btrfs subvolume get-default /
grep -E 'subvol=' /etc/fstab
cat /proc/cmdline
```

Expected results:

- `/` is Btrfs with `subvol=@`.
- `/home` is Btrfs with `subvol=@home`.
- `/.snapshots` is Btrfs with `subvol=@snapshots`.
- `/var/cache`, `/var/log`, `/var/spool` and `/var/tmp` are Btrfs with their matching `@var_*` subvolumes.
- `/boot` is a separate `ext4` mount and `/boot/efi` is `vfat`.
- The subvolume list contains all seven `@` subvolumes.
- `btrfs subvolume get-default /` prints `path @`.
- `/proc/cmdline` contains `rootflags=subvol=@`.
- There is no swap partition, swap file or `@swap` subvolume.

A newly created Btrfs subvolume starts with mode `755`. Set the expected ownership and modes explicitly, especially the sticky bit on `/var/tmp`, then restore the SELinux labels:

```bash
sudo chown root:root /var/cache /var/log /var/spool /var/tmp
sudo chmod 755 /var/cache /var/log /var/spool
sudo chmod 1777 /var/tmp
sudo restorecon -Rv /var/cache /var/log /var/spool /var/tmp
```

If the default subvolume is not `@`, fix it before installing packages:

```bash
sudo btrfs subvolume set-default \
  "$(sudo btrfs subvolume list / | awk '/ path @$/{print $2}')" /
sudo grub2-mkconfig -o /boot/grub2/grub.cfg
sudo dracut --force --regenerate-all
```

If any expected subvolume is missing from `/etc/fstab`, add it explicitly and verify with `findmnt`. Do not remove `subvol=@` from the root entry; without it, the system would boot the Btrfs top-level instead of `@`.

If the installer produced `root` and `home` instead of `@` and `@home`, stop. Do not rename a live root subvolume. Reinstall with the layout above.

## 2. Snapshots with Snapper

Snapper is the only snapshot system in this guide. The root configuration snapshots `@`; `@home`, `@snapshots` and the four `/var` subvolumes are separate and therefore excluded. `/boot` is ext4 and is never part of a root snapshot, so keep an independent `/boot` backup.

### 2.1 Install Snapper and create the root configuration

```bash
sudo dnf install -y snapper
sudo umount /.snapshots
sudo rmdir /.snapshots
sudo snapper -c root create-config /
sudo btrfs subvolume delete /.snapshots
sudo mkdir /.snapshots
sudo mount -a
findmnt /.snapshots
sudo chmod 750 /.snapshots
sudo restorecon -Rv /.snapshots
sudo snapper -c root list
```

Snapper's `create-config` insists on creating the `.snapshots` subvolume itself. This layout already has `@snapshots` for that purpose, so the sequence above temporarily unmounts it, removes the empty mount point, lets Snapper create its own nested `.snapshots`, deletes that nested subvolume, recreates the mount point and mounts `@snapshots` again. The Snapper configuration remains, but snapshots are stored in `@snapshots` outside `@`.

`findmnt /.snapshots` must show the Btrfs `@snapshots` subvolume. If `rmdir /.snapshots` fails because the directory is not empty, stop and inspect it; do not delete data.

Create the baseline snapshot:

```bash
sudo snapper -c root create -d "clean Fedora 44"
sudo snapper -c root list
```

Confirm that snapshots are not being written inside `@`:

```bash
sudo btrfs subvolume list / | grep '@/.snapshots' || echo "OK: snapshots are outside @"
```

The command must print `OK`. If it prints a path, snapshots are inside `@`; stop and fix the `/.snapshots` mount before continuing.

### 2.2 Set retention and enable timers

```bash
sudo snapper -c root set-config \
  TIMELINE_CREATE=yes \
  TIMELINE_LIMIT_HOURLY=0 \
  TIMELINE_LIMIT_DAILY=5 \
  TIMELINE_LIMIT_WEEKLY=3 \
  TIMELINE_LIMIT_MONTHLY=2 \
  TIMELINE_LIMIT_YEARLY=0 \
  NUMBER_LIMIT=10 \
  NUMBER_LIMIT_IMPORTANT=10

sudo systemctl enable --now snapper-timeline.timer snapper-cleanup.timer
sudo snapper -c root get-config | grep -E 'TIMELINE|NUMBER'
```

This keeps five daily, three weekly and two monthly roots, plus ten numbered and ten important snapshots. It does not create hourly snapshots. Adjust the limits only after checking `sudo btrfs filesystem usage /`.

### 2.3 Optional: automatic snapshots around DNF5 transactions

Fedora 44 uses DNF5. The old DNF4 Snapper plugin does not work with it. The native hook is `libdnf5-plugin-actions`, which calls helper scripts before and after a transaction.

```bash
sudo dnf install -y libdnf5-plugin-actions
```

Create the pre-transaction helper:

```bash
sudo tee /usr/local/bin/snapper-dnf5-pre >/dev/null <<'EOF'
#!/usr/bin/env bash
set -euo pipefail
state=/run/snapper-dnf5
mkdir -p "$state"
desc=$(tr '\0' ' ' <"/proc/$PPID/cmdline" 2>/dev/null | sed 's/[[:space:]]*$//')
[[ -n "$desc" ]] || desc="dnf5 transaction"
printf '%s\n' "$desc" >"$state/desc"
snapper -c root create --type pre --cleanup-algorithm number \
  --print-number --description "$desc" >"$state/pre"
EOF
sudo chmod 0755 /usr/local/bin/snapper-dnf5-pre
```

Create the post-transaction helper. It flushes the RPM SQLite WAL before the POST snapshot, then pairs the POST with the PRE snapshot:

```bash
sudo tee /usr/local/bin/snapper-dnf5-post >/dev/null <<'EOF'
#!/usr/bin/env bash
set -euo pipefail
state=/run/snapper-dnf5
[[ -s "$state/pre" ]] || exit 0
pre=$(cat "$state/pre")
desc=$(cat "$state/desc" 2>/dev/null || echo "dnf5 transaction")

python3 - <<'PY' || true
import sqlite3, sys, time
db = "/usr/lib/sysimage/rpm/rpmdb.sqlite"
for _ in range(10):
    try:
        con = sqlite3.connect(db, timeout=3)
        con.execute("PRAGMA busy_timeout=3000")
        result = con.execute("PRAGMA wal_checkpoint(TRUNCATE)").fetchone()
        con.close()
        if result and result[1] == 0:
            sys.exit(0)
    except sqlite3.Error:
        pass
    time.sleep(0.5)
sys.exit(1)
PY

snapper -c root create --type post --pre-number "$pre" \
  --cleanup-algorithm number --description "$desc"
rm -f "$state/pre" "$state/desc"
EOF
sudo chmod 0755 /usr/local/bin/snapper-dnf5-post
```

Register the hooks:

```bash
sudo mkdir -p /etc/dnf/libdnf5-plugins/actions.d
sudo tee /etc/dnf/libdnf5-plugins/actions.d/snapper.actions >/dev/null <<'EOF'
pre_transaction::::/usr/local/bin/snapper-dnf5-pre
post_transaction::::/usr/local/bin/snapper-dnf5-post
EOF
```

Test the hook with a small package:

```bash
sudo dnf install -y sl
sudo snapper -c root list
```

You should see a PRE/POST pair. If it does not appear, remove `/etc/dnf/libdnf5-plugins/actions.d/snapper.actions` and use the manual snapshot command in Section 2.5 until the plugin is fixed.

### 2.4 Verify Snapper

```bash
sudo snapper -c root list
sudo snapper -c root get-config | grep -E 'SUBVOLUME|TIMELINE|NUMBER'
systemctl is-enabled snapper-timeline.timer snapper-cleanup.timer
systemctl is-active snapper-timeline.timer snapper-cleanup.timer
findmnt /.snapshots
sudo btrfs filesystem usage /
sudo btrfs subvolume list /
```

`SUBVOLUME` must be `/`, both timers must be enabled and active, `/.snapshots` must be the `@snapshots` mount, and `snapper list` must show the baseline snapshot.

### 2.5 Snapshot policy

Create a manual snapshot before kernel updates, driver changes, Plasma upgrades and large package installs:

```bash
sudo snapper -c root create -d "before change"
```

Useful inspection commands:

```bash
sudo snapper -c root list
sudo snapper -c root status 1..2
sudo snapper -c root diff 1..2
```

Keep at least one known-good snapshot. `snapper-cleanup.timer` enforces the limits, but you can run cleanup manually:

```bash
sudo snapper -c root cleanup number
```

Keep an independent `/boot` backup before kernel changes, because `/boot` is not inside `@`:

```bash
sudo tar -C / -czf "/home/user/boot-backup-$(date +%F-%H%M).tar.gz" boot
```

The DNF configuration in Section 3.1 keeps three kernels (`installonly_limit=3`). Do not reduce it to one while relying on root snapshots.

`/home` is not snapshotted. Back up `~/THE_DATABASE`, `~/github-tools`, `~/Android/Sdk` and your dotfiles separately.

### 2.6 Rollback to a snapshot

Do **not** run the native `snapper rollback` command on this layout. It changes the Btrfs default subvolume, but `/etc/fstab` mounts `subvol=@` explicitly, so the change is ignored at boot and the snapshot is only marked active in Snapper.

Use this script instead. It renames the current `@` to a timestamped backup, creates a new `@` from the chosen snapshot, and sets the new `@` as the default.

```bash
sudo tee /usr/local/bin/snapper-rollback >/dev/null <<'EOF'
#!/usr/bin/env bash
set -euo pipefail

[[ ${EUID} -eq 0 ]] || { echo "Run with sudo: sudo snapper-rollback SNAPSHOT_ID"; exit 1; }
snap=${1:?Usage: sudo snapper-rollback SNAPSHOT_ID}

findmnt -no OPTIONS / | grep -Eq 'subvol=(/)?@(,|$)' || { echo "ERROR: / is not mounted with subvol=@"; exit 1; }
systemctl stop snapper-timeline.timer snapper-cleanup.timer 2>/dev/null || true

dev=$(findmnt -n -o SOURCE / | sed 's/\[.*//')
top=$(mktemp -d)
mount -o subvolid=5 "$dev" "$top"
new=""
cleanup() {
  if [[ -n "$new" && -d "$top/$new" ]]; then
    btrfs subvolume delete "$top/$new" 2>/dev/null || true
  fi
  umount "$top" 2>/dev/null || true
  rmdir "$top" 2>/dev/null || true
}
trap cleanup EXIT

src="$top/@snapshots/$snap/snapshot"
[[ -d "$src" ]] || { echo "ERROR: snapshot $snap not found at $src"; exit 1; }

backup="@.old-$(date +%Y%m%d-%H%M%S)"
new="@.new-$(date +%Y%m%d-%H%M%S)"
[[ ! -e "$top/$backup" && ! -e "$top/$new" ]] || { echo "ERROR: temporary subvolume already exists"; exit 1; }

echo "Creating $new from snapshot $snap"
btrfs subvolume snapshot "$src" "$top/$new"
btrfs property set "$top/$new" ro false

echo "Saving current @ as $backup"
mv "$top/@" "$top/$backup"

if ! mv "$top/$new" "$top/@"; then
  echo "ERROR: could not install the new @; restoring $backup"
  mv "$top/$backup" "$top/@"
  exit 1
fi

newid=$(btrfs subvolume list "$top" | awk '/ path @$/{print $2}')
if ! btrfs subvolume set-default "$newid" "$top"; then
  echo "ERROR: could not set the new default; restoring $backup"
  mv "$top/@" "$top/$new"
  mv "$top/$backup" "$top/@"
  exit 1
fi
sync

echo
echo "Rollback prepared. Reboot with: sudo reboot"
echo "Old root kept as $backup; delete it only after the reboot is verified."
EOF
sudo chmod 0755 /usr/local/bin/snapper-rollback
```

List snapshots, choose an ID and roll back:

```bash
sudo snapper -c root list
sudo snapper-rollback SNAPSHOT_ID
sudo reboot
```

After reboot, verify the new root before deleting the old one:

```bash
findmnt -no TARGET,SOURCE,OPTIONS /
sudo snapper -c root list
sudo btrfs subvolume get-default /
```

Delete the saved `@.old-*` only after the system boots and works:

```bash
sudo mount -o subvolid=5 /dev/BTRFS_PART /mnt
sudo btrfs subvolume list /mnt | grep '@.old'
sudo btrfs subvolume delete /mnt/@.old-YYYYMMDD-HHMMSS
sudo umount /mnt
```

Replace `/dev/BTRFS_PART` with the actual Btrfs partition from `findmnt -no SOURCE /` (strip any `[...]` suffix). If the root is on LUKS, use the `/dev/mapper/...` device.

For a selective file restore, use `snapper undochange` instead of a full root swap. Review the diff first and reboot afterward:

```bash
sudo snapper -c root status 1..2
sudo snapper -c root diff 1..2
sudo snapper -c root undochange 1..2 /etc
sudo reboot
```

Because `/boot` is not snapshotted, a root rollback that crosses a kernel update can leave the current kernel without matching modules inside the restored `@`. Keep at least two kernels and a Fedora 44 live USB. If the system does not boot, boot the live USB, mount the Btrfs top level, restore the saved `@.old-*` by renaming it back to `@`, and boot again. Do not delete `@.old-*` until you have tested the rollback.

---

## 3. Base System

### 3.1 DNF5 configuration

```bash
sudo tee /etc/dnf/dnf.conf >/dev/null <<'EOF'
[main]
gpgcheck=True
installonly_limit=3
clean_requirements_on_remove=True
max_parallel_downloads=10
defaultyes=True
EOF

sudo dnf clean all
sudo dnf makecache
```

`max_parallel_downloads=10` is safe on this wired connection. Do not enable `keepcache`; the SSD is only 120 GB.

### 3.2 Update Fedora

Take a Snapper snapshot first, then update the new installation:

```bash
sudo snapper -c root create -d "before first dnf upgrade"
sudo dnf upgrade --refresh -y
sudo systemctl reboot
```

After the reboot, confirm the release and kernel:

```bash
cat /etc/fedora-release
rpm -E %fedora
uname -r
```

`rpm -E %fedora` must print `44`.

### 3.3 Firmware

```bash
sudo fwupdmgr refresh --force
sudo fwupdmgr get-devices
sudo fwupdmgr get-updates
sudo fwupdmgr update
```

HP does not publish every ZBook 15 G2 BIOS release to LVFS. If `get-updates` reports no devices or no updates, use HP's support page for the BIOS; do not force an update from an untrusted source.

### 3.4 Hostname, time and locale

```bash
sudo hostnamectl set-hostname fedora44
sudo timedatectl set-timezone Africa/Cairo
sudo timedatectl set-ntp true
sudo localectl set-locale LANG=en_US.UTF-8

hostnamectl
timedatectl
localectl status
```

`System clock synchronized: yes` is the line to check in `timedatectl`.

### 3.5 Mount the NTFS data disk

The old MX entry used the FUSE `ntfs-3g` driver. Fedora's kernel includes the faster `ntfs3` driver. Keep `ntfs-3g` and `ntfsprogs` installed as a fallback and for `ntfsfix`.

Before changing anything, shut Windows down completely and disable Fast Startup; do not leave the NTFS volume hibernated. A dirty or hibernated volume can fail to mount or be damaged.

Install the NTFS tools, create the mount point and back up `fstab`:

```bash
sudo dnf install -y ntfs-3g ntfsprogs
sudo mkdir -p /media/DATA
sudo cp -a /etc/fstab "/etc/fstab.bak-$(date +%F-%H%M%S)"
```

Identify the correct partition and check your UID/GID. Do not guess from the example UUID:

```bash
lsblk -f
sudo blkid -t TYPE=ntfs -o full
id -u user
id -g user
```

The UID/GID in this guide are `1000`. If `id` prints different numbers, replace both values in the `fstab` line.

Add one line to `/etc/fstab` with `sudoedit /etc/fstab`:

```fstab
UUID=E0F0BD6DF0BD4A94  /media/DATA  ntfs3  defaults,uid=1000,gid=1000,dmask=0002,fmask=0113,nofail,x-systemd.device-timeout=10  0 0
```

- Replace `E0F0BD6DF0BD4A94` with the UUID from `blkid`.
- `ntfs3` is the in-kernel driver; it is faster than the FUSE `ntfs-3g` fallback.
- `uid=1000,gid=1000` maps all files to `user`; NTFS does not store Linux ownership.
- `dmask=0002,fmask=0113` gives group-writable directories and removes execute bits from regular files.
- `nofail` keeps a missing disk from blocking boot, and `x-systemd.device-timeout=10` avoids a long boot delay.
- If you share the disk through Samba, add `context=system_u:object_r:samba_share_t:s0` before `nofail`, then verify with `ls -Zd /media/DATA` and `sudo ausearch -m AVC -ts recent`. That label is Samba-specific; do not use it blindly for NFS.

Test before rebooting. If the disk is already mounted, close anything using it and unmount it first so the new options take effect:

```bash
sudo systemctl daemon-reload
sudo findmnt --verify
sudo umount /media/DATA 2>/dev/null || true
sudo mount -a
findmnt /media/DATA
ls -la /media/DATA
```

`findmnt` should show `ntfs3` and the expected options. Reboot once and confirm the disk mounts without help:

```bash
sudo systemctl reboot
```

After the reboot:

```bash
findmnt /media/DATA
mount | grep ' /media/DATA '
```

If the mount fails:

```bash
dmesg | grep -i ntfs3
```

- A dirty or hibernated volume must be fixed from Windows with a clean shutdown and a filesystem check. `sudo ntfsfix /dev/sdXN` only clears the dirty flag; it is not a substitute for Windows `chkdsk`.
- If `ntfs3` does not work with this volume, change the type to `ntfs-3g` and use the FUSE options:

```fstab
UUID=E0F0BD6DF0BD4A94  /media/DATA  ntfs-3g  defaults,uid=1000,gid=1000,umask=022,nofail,x-systemd.device-timeout=10  0 0
```

A malformed `fstab` can stop the boot. If that happens, boot the Fedora live USB, mount the Btrfs root with `subvol=@`, restore the newest `fstab` backup, and reboot:

```bash
sudo mkdir -p /mnt/root
sudo mount -o subvol=@ /dev/BTRFS_PART /mnt/root
ls /mnt/root/etc/fstab.bak-*
sudo cp /mnt/root/etc/fstab.bak-YYYYMMDD-HHMMSS /mnt/root/etc/fstab
sudo umount /mnt/root
```

Replace `/dev/BTRFS_PART` with the Btrfs partition from `blkid` (or the LUKS mapper device).

### 3.6 Reboot checkpoint

```bash
sudo systemctl reboot
```

After logging in, `systemctl --failed` should be empty.

---

## 4. Repositories

### 4.1 RPM Fusion 44

```bash
sudo dnf install -y \
  https://mirrors.rpmfusion.org/free/fedora/rpmfusion-free-release-$(rpm -E %fedora).noarch.rpm \
  https://mirrors.rpmfusion.org/nonfree/fedora/rpmfusion-nonfree-release-$(rpm -E %fedora).noarch.rpm

sudo dnf install -y rpmfusion-free-release-tainted rpmfusion-nonfree-release-tainted
```

RPM Fusion free provides `VirtualBox`, `akmod-VirtualBox`, `ffmpeg`, `vlc-plugins-freeworld`, `HandBrake-gui` and the Intel VA-API driver. The tainted repositories provide `libdvdcss` for encrypted DVDs.

### 4.2 Flathub

```bash
sudo dnf install -y flatpak
flatpak remote-add --if-not-exists flathub https://flathub.org/repo/flathub.flatpakrepo
flatpak remote-ls flathub --columns=application | head
```

### 4.3 Cisco OpenH264

Fedora ships the Cisco repository definition disabled. Enable it, then install the codec packages:

```bash
sudo dnf config-manager setopt fedora-cisco-openh264.enabled=1
sudo dnf install -y gstreamer1-plugin-openh264 mozilla-openh264
```

### 4.4 Google Chrome

```bash
sudo dnf install -y https://dl.google.com/linux/direct/google-chrome-stable_current_x86_64.rpm
```

### 4.5 Brave

```bash
sudo dnf config-manager addrepo --from-repofile=https://brave-browser-rpm-release.s3.brave.com/brave-browser.repo
sudo dnf install -y brave-browser
```

The repository file already contains Brave's signing key; no separate `rpm --import` is needed.

### 4.6 Visual Studio Code

```bash
sudo rpm --import https://packages.microsoft.com/keys/microsoft.asc
sudo tee /etc/yum.repos.d/vscode.repo >/dev/null <<'EOF'
[code]
name=Visual Studio Code
baseurl=https://packages.microsoft.com/yumrepos/vscode
enabled=1
autorefresh=1
type=rpm-md
gpgcheck=1
gpgkey=https://packages.microsoft.com/keys/microsoft.asc
EOF

sudo dnf install -y code
```

### 4.7 Docker CE

```bash
sudo dnf config-manager addrepo --from-repofile=https://download.docker.com/linux/fedora/docker-ce.repo
sudo dnf install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo systemctl enable --now docker
sudo usermod -aG docker "$USER"
```

The group change takes effect after you log out and back in. Docker is optional; Podman is installed later and needs no daemon.

### 4.8 Adoptium Temurin 17

Adoptium does not publish a `.repo` file for Fedora, so create the repository directly:

```bash
sudo tee /etc/yum.repos.d/adoptium.repo >/dev/null <<'EOF'
[adoptium]
name=Adoptium
baseurl=https://packages.adoptium.net/artifactory/rpm/fedora/$releasever/$basearch
enabled=1
gpgcheck=1
gpgkey=https://packages.adoptium.net/artifactory/api/gpg/key/public
EOF

sudo dnf install -y temurin-17-jdk
```

Fedora's OpenJDK 25 is the system JDK. Temurin 17 is kept only for projects that require it.

### 4.9 Refresh and verify

```bash
sudo dnf makecache
dnf repolist --enabled
rpm -q google-chrome-stable brave-browser code docker-ce docker-ce-cli containerd.io temurin-17-jdk
flatpak remotes --columns=name,url
```

All RPM queries must print a version. A missing package means that repository block was skipped.

## 5. Multimedia and Intel Graphics

### 5.1 Full FFmpeg

Fedora ships `ffmpeg-free`, which omits some encoders. Replace it with RPM Fusion's full build:

```bash
sudo dnf install -y --allowerasing ffmpeg
```

Keep the RPM Fusion build. Do not install a second FFmpeg from source; mixing builds breaks `ffmpeg`, `ffplay` and applications that link to `libav*`.

### 5.2 GStreamer

```bash
sudo dnf install -y \
  gstreamer1-plugins-bad-freeworld \
  gstreamer1-plugins-ugly \
  gstreamer1-plugin-libav \
  gstreamer1-plugin-openh264
```

### 5.3 Encrypted DVDs

```bash
sudo dnf install -y libdvdcss vlc-plugins-freeworld
```

`libdvdcss` comes from the RPM Fusion tainted repository. It is required for commercial DVD playback; it is not a general-purpose codec.

### 5.4 Intel VA-API on Haswell

The HD 4600 is Gen 7.5. It uses the legacy `i965` VA-API driver. The newer `intel-media-driver` starts at Gen 8 (Broadwell) and does not support this GPU.

```bash
sudo dnf install -y \
  libva-intel-driver \
  libva-utils \
  mesa-dri-drivers \
  mesa-vulkan-drivers \
  vulkan-tools
```

Make the driver choice explicit for the session:

```bash
mkdir -p "$HOME/.config/environment.d"
cat > "$HOME/.config/environment.d/90-intel-vaapi.conf" <<'EOF'
LIBVA_DRIVER_NAME=i965
EOF
```

Log out and back in, then verify:

```bash
vainfo
vulkaninfo --summary
```

`vainfo` must report the **i965** driver and at least one H.264 profile. `vulkaninfo --summary` should show the Intel Haswell Vulkan device. Do not use `glxinfo` as the primary check; it is an X11/GLX tool and this guide does not install an Xorg session.

> **Do not install `intel-media-driver` or `libva-intel-media-driver` on this laptop.** They are for Gen 8 and newer Intel graphics and will not provide VA-API on Haswell.

### 5.5 Firefox hardware decoding

In Firefox, open `about:config` and set:

| Preference | Value |
|---|---|
| `media.ffmpeg.vaapi.enabled` | `true` |
| `media.hardware-video-decoding.enabled` | `true` |

On Wayland, Firefox uses the Wayland backend by default. Test with a local H.264 video and check `about:support` → **Media** for hardware decoding. If Firefox falls back to XWayland, fix the Wayland backend instead of treating XWayland as the target environment.

### 5.6 Bluetooth audio

```bash
sudo dnf install -y pipewire-codec-aptx libfreeaptx libldac fdk-aac-free
sudo systemctl enable --now bluetooth
```

PipeWire and WirePlumber use these codecs automatically. Verify with `bluetoothctl` after pairing headphones; the codec is shown in the device properties.

### 5.7 Verify

```bash
ffmpeg -version | head -n1
ffmpeg -codecs 2>/dev/null | grep -Ei 'h264|hevc'
vainfo | grep -E 'Driver version|i965|H264'
vulkaninfo --summary | grep -E 'deviceName|apiVersion'
```

---

## 6. Intel Graphics and Session Choice

### 6.1 Identify the GPU

```bash
lspci -nnk | grep -A3 -E 'VGA|3D'
lsmod | grep i915
```

The expected device is `Intel Corporation Haswell-ULT Integrated Graphics Controller` using the `i915` kernel driver.

### 6.2 Wayland-only session

Plasma 6.7.5 uses Wayland by default. This guide does **not** install the X11 session. Do not install `plasma-workspace-x11`, and do not select a Plasma X11 session at the login screen. Use Wayland for the desktop and for native applications.

XWayland may still start inside the Wayland compositor for an application that has not been ported. That is a per-application fallback, not an Xorg session; do not build a workflow around it. If an application only works under a real Xorg session, treat it as unsupported here.

Check the current session; it must report `wayland`:

```bash
echo "$XDG_SESSION_TYPE"
plasmashell --version
kwin_wayland --version
```

### 6.3 Verify

```bash
vainfo | grep -E 'i965|H264'
vulkaninfo --summary | grep -E 'deviceName|apiVersion'
```

---

## 7. Security and System Services

### 7.1 SELinux

Fedora enables SELinux in enforcing mode by default. Keep it enforcing.

```bash
getenforce
sudo dnf install -y policycoreutils-python-utils setools-console
sestatus
```

If `getenforce` prints `Permissive` only because you changed it temporarily, restore enforcement with `sudo setenforce 1`. Do not disable SELinux to work around a denial; read the AVC and fix the label or policy.

### 7.2 firewalld

```bash
sudo dnf install -y firewalld firewall-config
sudo systemctl enable --now firewalld
sudo firewall-cmd --state
sudo firewall-cmd --get-default-zone
sudo firewall-cmd --list-all
```

The `FedoraWorkstation` zone opens TCP and UDP ports **1025-65535** by default. That is convenient on a desktop but exposes every high-numbered test service (Burp, local web servers, lab listeners) to the LAN. For this machine, use the more restrictive `public` zone as the default:

```bash
sudo firewall-cmd --set-default-zone=public
sudo firewall-cmd --reload
sudo firewall-cmd --get-default-zone
sudo firewall-cmd --list-all
```

`--set-default-zone` affects new connections, not an interface that is already assigned to `FedoraWorkstation`. Check `nmcli device status`, then move the active interface explicitly if needed:

```bash
sudo firewall-cmd --change-interface=wlp3s0 --zone=public --permanent
sudo firewall-cmd --reload
sudo firewall-cmd --get-active-zones
```

Replace `wlp3s0` with the interface name from `nmcli`. Add only the services you actually use to `public`; do not reopen the full high-port range.

### 7.3 SSD TRIM

```bash
sudo systemctl enable --now fstrim.timer
systemctl list-timers fstrim.timer --no-pager
```

### 7.4 ZRAM: the only swap

This build uses Fedora's `zram-generator` for compressed RAM swap. There is no disk swap partition, no swap file and no hibernation configuration.

Why this is the right choice for this laptop:

- ZRAM keeps swap pages compressed in RAM and avoids SSD write amplification on the 120 GB drive.
- Hibernation needs a persistent disk swap area large enough to hold RAM. ZRAM cannot be a hibernation target.
- With 16 GB RAM, suspend-to-RAM plus a clean shutdown is more appropriate than reserving a 16 GiB swap file and maintaining resume configuration.

Verify that ZRAM is active:

```bash
zramctl
swapon --show
cat /proc/swaps
free -h
systemctl status systemd-zram-setup@zram0.service --no-pager
```

Expected: `zramctl` lists `zram0`; `swapon --show` and `/proc/swaps` show `/dev/zram0`; there is no `/var/swap/swapfile`, no disk-backed swap partition, and `/proc/cmdline` contains no `resume=`.

If ZRAM is missing:

```bash
sudo dnf install -y zram-generator-defaults
sudo systemctl daemon-reload
sudo systemctl start systemd-zram-setup@zram0.service
zramctl
```

Do not create a swap file or swap partition, do not add `resume=` to the kernel command line, and do not install a hibernation helper. If you later decide that hibernation is essential, you must add a disk swap area large enough for RAM and change this policy; ZRAM alone cannot hibernate.

If you followed an earlier version of this guide, remove the old disk-swap setup:

```bash
sudo swapoff /var/swap/swapfile 2>/dev/null || true
sudo sed -i '\#^/var/swap/swapfile[[:space:]]#d' /etc/fstab
sudo rm -f /var/swap/swapfile
sudo rm -f /etc/dracut.conf.d/resume.conf
sudo dracut -f
```

Then remove the `/var/swap` fstab entry, unmount `/var/swap`, and delete the old top-level `@swap` subvolume from a top-level mount. Do not delete `@` or `@home`.

### 7.5 Automatic security updates

```bash
sudo dnf install -y dnf5-plugin-automatic
# The package installs /etc/dnf/automatic.conf as an empty ghost file; seed it from the packaged default.
[ -s /etc/dnf/automatic.conf ] || sudo cp /usr/share/dnf5/dnf5-plugins/automatic.conf /etc/dnf/automatic.conf
sudo sed -i \
  -e 's/^apply_updates = no/apply_updates = yes/' \
  -e 's/^upgrade_type = default/upgrade_type = security/' \
  /etc/dnf/automatic.conf
sudo systemctl enable --now dnf5-automatic.timer
```

Check the resulting file:

```bash
grep -E '^(apply_updates|upgrade_type|download_updates)' /etc/dnf/automatic.conf
```

`apply_updates = yes` and `upgrade_type = security` are the intended values. Kernel updates still require a reboot; the timer does not reboot the machine.

### 7.6 Power management

Keep **tuned-ppd**. It provides the D-Bus PowerProfiles interface used by Plasma's Power Management panel and is the default on Fedora 44 KDE. The `tuned-ppd` package does **not** ship `/usr/bin/powerprofilesctl`; that command belongs to `power-profiles-daemon`, and the two packages conflict.

```bash
sudo dnf install -y tuned tuned-ppd
rpm -q tuned tuned-ppd
systemctl status tuned tuned-ppd --no-pager
tuned-adm active
```

Use KDE **Power Management** to change the profile, or `sudo tuned-adm profile balanced` from a terminal. Do **not** install `power-profiles-daemon` alongside `tuned-ppd`, and do **not** install TLP or auto-cpufreq alongside it; they compete for the same controls. If idle power is still poor, measure first with `powertop` before adding a second power daemon.

Optional thermal diagnostics:

```bash
sudo dnf install -y powertop
sudo powertop
```

If the fan runs constantly or the CPU throttles under light load, clean the fan and replace the thermal paste before adding software workarounds. `thermald` is optional:

```bash
sudo dnf install -y thermald
sudo systemctl enable --now thermald
```

### 7.7 Verify

```bash
getenforce
firewall-cmd --state
systemctl is-enabled fstrim.timer
systemctl is-enabled dnf5-automatic.timer
systemctl is-active tuned-ppd
tuned-adm active
zramctl
systemctl --failed
```

`systemctl --failed` should print no failed units.

---

## 8. Shell and Command-Line Tools

### 8.1 Install the set

```bash
sudo dnf install -y \
  htop btop ncdu duf tree \
  7zip unzip zip \
  rsync rclone aria2 wget1-wget curl \
  tmux screen mc jq yq \
  ripgrep fd-find fzf bat eza zoxide \
  nano vim-enhanced neovim \
  man-db man-pages bash-completion \
  openssl gnupg2 pv progress \
  lshw dmidecode smartmontools nvme-cli hdparm pciutils usbutils lm_sensors hardinfo2 \
  desktop-file-utils ShellCheck shfmt gh \
  wl-clipboard gnuplot
```

`p7zip` is retired; Fedora 44 uses `7zip`, which provides the `7z` command. `fd-find` provides `fd`, and `ripgrep` provides `rg`.

### 8.2 Verify

```bash
for c in 7z htop btop ncdu duf tree rclone tmux screen mc jq yq rg fd fzf bat eza zoxide nano vim nvim shellcheck shfmt gh; do
  command -v "$c" >/dev/null || echo "MISSING: $c"
done

rpm -q desktop-file-utils
desktop-file-validate --help >/dev/null
```

No `MISSING` lines should appear. `htop`, `btop` and `hardinfo2` already ship desktop entries; the launcher section does not duplicate them.

---

## 9. Development Toolchain

### 9.1 Build tools

```bash
sudo dnf group install -y development-tools
sudo dnf install -y \
  gcc gcc-c++ make cmake ninja-build meson \
  autoconf automake libtool pkgconf-pkg-config \
  git git-lfs gh \
  kernel-devel kernel-headers akmods \
  openssl-devel libcurl-devel libxml2-devel libffi-devel \
  readline-devel zlib-ng-compat-devel bzip2-devel xz-devel sqlite-devel \
  ncurses-devel gdb lldb strace ltrace valgrind \
  clang llvm llvm-devel \
  protobuf-devel protobuf-compiler
```

### 9.2 TeX Live

`texlive-scheme-medium` is a sensible size for this 120 GB SSD. Install the full scheme only if a document actually needs it.

```bash
sudo dnf install -y texlive-scheme-medium
```

### 9.3 Documents and conversion

```bash
sudo dnf install -y \
  pandoc-cli poppler-utils ghostscript ImageMagick \
  libreoffice-writer libreoffice-calc libreoffice-impress libreoffice-draw \
  tesseract tesseract-langpack-eng
```

For Arabic OCR, add `tesseract-langpack-ara` separately.

### 9.4 Verify

```bash
gcc --version | head -n1
g++ --version | head -n1
cmake --version | head -n1
meson --version
pandoc --version | head -n1
pdflatex --version | head -n1
libreoffice --version
tesseract --version | head -n1
```

---

## 10. Languages and Runtimes

### 10.1 Go

Install the current upstream toolchain under `/usr/local/go`:

```bash
GO_VERSION="$(curl -fsSL 'https://go.dev/VERSION?m=text' | head -n1)"
curl -fsSLO "https://go.dev/dl/${GO_VERSION}.linux-amd64.tar.gz"
sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf "${GO_VERSION}.linux-amd64.tar.gz"

grep -q '/usr/local/go/bin' "$HOME/.bashrc" || \
  echo 'export PATH=/usr/local/go/bin:$HOME/go/bin:$PATH' >> "$HOME/.bashrc"
source "$HOME/.bashrc"
go version
```

The download is the official `go.dev` tarball; `GO_VERSION` is resolved at install time, so the guide does not hard-code a version that will age.

### 10.2 Rust

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y
source "$HOME/.cargo/env"
rustc --version
cargo --version
```

### 10.3 Node.js with nvm

Use nvm rather than a distribution Node package. This keeps project runtimes independent. Resolve the latest nvm tag instead of pinning a version that may not exist:

```bash
NVM_TAG="$(curl -fsSL https://api.github.com/repos/nvm-sh/nvm/releases/latest \
  | sed -n 's/.*"tag_name": *"\([^"]*\)".*/\1/p' | head -n1)"
[[ -n "$NVM_TAG" ]] || { echo "ERROR: could not resolve the latest nvm tag"; return 1 2>/dev/null || exit 1; }
echo "nvm: $NVM_TAG"
curl -fsSL "https://raw.githubusercontent.com/nvm-sh/nvm/${NVM_TAG}/install.sh" | bash
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && . "$NVM_DIR/nvm.sh"
nvm install --lts
nvm alias default 'lts/*'
node --version
npm --version
```

The GitHub API allows 60 unauthenticated requests per hour per IP. If the request fails, wait and retry rather than falling back to an old tag.

### 10.4 Java

```bash
sudo dnf install -y java-25-openjdk-devel
java -version
javac -version
```

Temurin 17 was installed in [Section 4.8](#48-adoptium-temurin-17). It lives under `/usr/lib/jvm/temurin-17-jdk` and is selected per project with `JAVA_HOME`; the system default remains the current OpenJDK package. Resolve the active JDK instead of hard-coding a path:

```bash
export JAVA_HOME="$(dirname "$(dirname "$(readlink -f "$(command -v javac)")")")"
echo "$JAVA_HOME"
```

For a project that needs Temurin 17, set `JAVA_HOME=/usr/lib/jvm/temurin-17-jdk` in that project's shell only.

### 10.5 Python

```bash
sudo dnf install -y python3 python3-devel python3-pip pipx
pipx ensurepath
python3 -m venv "$HOME/.venvs/main"
source "$HOME/.venvs/main/bin/activate"
python -m pip install --upgrade pip
deactivate
```

### 10.6 Ruby and Perl

```bash
sudo dnf install -y ruby ruby-devel rubygems rubygem-bundler
ruby --version
gem --version
perl --version | head -n2
```

### 10.7 Global npm tools

```bash
source "$HOME/.nvm/nvm.sh"
npm install -g pnpm reverse-sourcemap retire
```

### 10.8 Verify

```bash
go version
rustc --version
cargo --version
node --version
npm --version
java -version
python3 --version
pipx --version
ruby --version
```

Open a new terminal before relying on `go`, `cargo` or `nvm`; the current shell may not yet have the updated `PATH`.

## 11. Security and Pentest Toolkit

### 11.1 Native packages

Fedora 44 packages more of this toolkit than earlier releases. Use native packages where they exist; do not build a second copy with `go install`. At the time of writing, `whatweb-0.6.3-3.fc44` and `grype-0.117.0-2.fc44` are in Fedora 44. Verify the uncertain names before the large transaction; DNF5 aborts the transaction if one name is missing:

```bash
for p in whatweb grype govulncheck staticcheck golangci-lint gitleaks medusa cowpatty; do
  dnf repoquery --quiet "$p" | grep -q . || echo "MISSING: $p"
done
```

Remove or replace any `MISSING` name; do not use `--skip-unavailable`, because it hides typos and unavailable packages.

```bash
sudo dnf install -y \
  nmap nmap-ncat masscan zmap arp-scan bind-utils whois traceroute mtr \
  hydra ncrack medusa john hashcat \
  gobuster ffuf whatweb subfinder dnsx gitleaks \
  golangci-lint govulncheck staticcheck \
  wireshark wireshark-cli tcpdump \
  ettercap cutter-re radare2 rizin \
  dsniff hping3 \
  binwalk foremost sleuthkit yara \
  clamav clamtk lynis chkrootkit rkhunter \
  nuclei grype \
  aircrack-ng reaver hcxtools kismet pixiewps cowpatty \
  libpcap-devel libnl3-devel libcurl-devel libxslt-devel \
  ruby-devel rubygems perl pocl
```

`nuclei` and `grype` are Fedora 44 packages now. Do not use `go install` or the Anchore install script for them; that creates a second, unmanaged copy.

Configure the tools that need data or groups:

```bash
sudo usermod -aG wireshark "$USER"
nuclei -update-templates
grype db update
sudo freshclam
sudo systemctl enable --now clamav-freshclam
```

`wireshark` group membership takes effect after a new login. `nuclei -update-templates` downloads the official template repository; it is intentionally separate from the RPM.

### 11.2 Source installs Fedora does not package

```bash
sudo git clone --depth 1 https://github.com/sqlmapproject/sqlmap.git /opt/sqlmap
sudo chmod +x /opt/sqlmap/sqlmap.py
sudo ln -sf /opt/sqlmap/sqlmap.py /usr/local/bin/sqlmap

sudo git clone --depth 1 https://github.com/sullo/nikto.git /opt/nikto
sudo chmod +x /opt/nikto/program/nikto.pl
sudo ln -sf /opt/nikto/program/nikto.pl /usr/local/bin/nikto

sudo git clone --depth 1 https://github.com/digininja/CeWL.git /opt/CeWL
cd /opt/CeWL && sudo bundle install
sudo tee /usr/local/bin/cewl >/dev/null <<'EOF'
#!/usr/bin/env bash
exec env BUNDLE_GEMFILE=/opt/CeWL/Gemfile bundle exec ruby /opt/CeWL/cewl.rb "$@"
EOF
sudo chmod 0755 /usr/local/bin/cewl

sudo gem install wpscan
```

`sqlmap` and `nikto` are installed under `/opt` and symlinked into `/usr/local/bin`. `cewl` is the upstream Ruby script, wrapped so it runs from any directory; its gems are installed with Bundler. `wpscan` is a Ruby gem. WPScan is optional; if the gem fails to build on a new Ruby release, use its official Docker image instead of forcing native extensions.

### 11.3 Go, Python and Rust tools

```bash
source "$HOME/.bashrc"

# Go tools not packaged by Fedora
go install github.com/projectdiscovery/httpx/cmd/httpx@latest
go install github.com/projectdiscovery/katana/cmd/katana@latest
go install github.com/projectdiscovery/naabu/v2/cmd/naabu@latest
go install github.com/lc/gau/v2/cmd/gau@latest
go install github.com/BishopFox/jsluice/cmd/jsluice@latest
go install github.com/owasp-amass/amass/v4/cmd/amass@latest
go install github.com/sensepost/gowitness@latest
go install github.com/bettercap/bettercap/v2@latest

# Python tools
pipx install semgrep
pipx install dirsearch
pipx install drozer
pipx install frida-tools
pipx install objection
pipx install sherlock-project
pipx install uro
pipx install libsast
pipx install detect-secrets
pipx install waymore
pipx install subdominator
pipx install rembg
pipx install gallery-dl
pipx install litecli
pipx install weasyprint
pipx install pyinstaller
pipx install git+https://github.com/initstring/cloud_enum
pipx install git+https://github.com/derv82/wifite2

# Rust tool
cargo install feroxbuster

# Official install scripts
curl -sSfL https://raw.githubusercontent.com/anchore/syft/main/install.sh | sh -s -- -b "$HOME/.local/bin"
curl -sSfL https://raw.githubusercontent.com/trufflesecurity/trufflehog/main/scripts/install.sh | sh -s -- -b "$HOME/.local/bin"

# Nosey Parker release binary
NP_URL="$(curl -s https://api.github.com/repos/praetorian-inc/noseyparker/releases/latest \
  | grep -o 'https://[^"]*x86_64-unknown-linux-gnu.tar.gz' | head -1)"
[[ -n "$NP_URL" ]] || { echo "ERROR: could not resolve the Nosey Parker URL"; exit 1; }
curl -L -o /tmp/noseyparker.tar.gz "$NP_URL"
NP_TMP="$(mktemp -d)"
tar -xzf /tmp/noseyparker.tar.gz -C "$NP_TMP"
sudo install -d /usr/local/lib/noseyparker
sudo cp -a "$NP_TMP"/. /usr/local/lib/noseyparker/
sudo ln -sfn /usr/local/lib/noseyparker/bin/noseyparker /usr/local/bin/noseyparker
rm -rf "$NP_TMP"
```

`go install` binaries land in `~/go/bin`; the shell block in [Section 20](#20-migration-and-dotfiles) adds that directory to `PATH`. `pipx` binaries land in `~/.local/bin`. `feroxbuster` lands in `~/.cargo/bin`.

The `cloud_enum` repository includes a `pyproject.toml`, so `pipx install git+...` should work. If upstream changes packaging or the command is not on `PATH`, remove that line rather than keeping a broken install.

### 11.4 Optional CodeQL CLI

CodeQL is large; install it only if you use it locally.

```bash
CODEQL_URL="$(curl -s https://api.github.com/repos/github/codeql-cli-binaries/releases/latest \
  | grep -o 'https://[^"]*codeql-linux64\.zip' | head -1)"
[[ -n "$CODEQL_URL" ]] || { echo "ERROR: could not resolve the CodeQL URL"; exit 1; }
curl -L -o /tmp/codeql.zip "$CODEQL_URL"
sudo unzip -q /tmp/codeql.zip -d /opt
sudo ln -sfn /opt/codeql/codeql /usr/local/bin/codeql
codeql version
```

### 11.5 Verify

```bash
for c in nmap masscan hydra john hashcat sqlmap nikto gobuster ffuf whatweb \
         nuclei grype syft trufflehog noseyparker semgrep feroxbuster; do
  command -v "$c" >/dev/null || echo "MISSING: $c"
done

nuclei -version
grype version
syft version
trufflehog --version
semgrep --version
```

No `MISSING` lines should appear. WPScan and CeWL are Ruby executables; check them with `command -v wpscan cewl`.

---

## 12. Wireless and RF Tooling

### 12.1 What this laptop can do

The Intel Wireless 7260 is handled by the in-tree `iwlwifi` driver. Do not install out-of-tree Wi-Fi drivers for it. Monitor mode and frame injection depend on the card and firmware; check before relying on them:

```bash
iw list | grep -A8 'Supported interface modes'
```

If `monitor` is absent, use an external USB adapter for wireless audits. The tools below still install and run; the hardware limit is the issue, not the software.

### 12.2 Native tools

```bash
sudo dnf install -y \
  aircrack-ng reaver hcxtools kismet pixiewps cowpatty \
  hostapd dnsmasq dhcp-server macchanger iw \
  libpcap-devel libnl3-devel libcurl-devel
```

### 12.3 Build the tools Fedora does not package

```bash
git clone --depth 1 https://github.com/ZerBea/hcxdumptool.git "$HOME/github-tools/hcxdumptool"
make -C "$HOME/github-tools/hcxdumptool"
sudo make -C "$HOME/github-tools/hcxdumptool" install

git clone --depth 1 https://github.com/aircrack-ng/mdk4.git "$HOME/github-tools/mdk4"
make -C "$HOME/github-tools/mdk4"
sudo make -C "$HOME/github-tools/mdk4" install

git clone --depth 1 https://github.com/aanarchyy/bully.git "$HOME/github-tools/bully"
make -C "$HOME/github-tools/bully/src"
sudo make -C "$HOME/github-tools/bully/src" install
```

`bully`'s repository root contains an OpenWrt package `Makefile`; build from `src/` as shown, not from the repository root.

`wifite` is installed with `pipx` in [Section 11.3](#113-go-python-and-rust-tools). `bettercap` is a Go tool in the same section.

### 12.4 Verify

```bash
for c in airmon-ng airodump-ng aireplay-ng aircrack-ng reaver bully mdk4 \
         hcxdumptool hcxpcapngtool kismet pixiewps cowpatty wifite bettercap; do
  command -v "$c" >/dev/null || echo "MISSING: $c"
done

sudo airmon-ng
iw dev
```

`sudo airmon-ng` should list the wireless interface and driver without errors. Do not leave the adapter in monitor mode when you are finished.

---

## 13. Networking, VPN and Remote Access

### 13.1 Packages

```bash
sudo dnf install -y \
  NetworkManager-openvpn NetworkManager-openvpn-gnome \
  NetworkManager-openconnect NetworkManager-openconnect-gnome \
  openvpn openconnect wireguard-tools \
  openssh-server \
  remmina freerdp virt-viewer putty \
  samba samba-client samba-common-tools \
  nfs-utils nfs-common-utils \
  gpsd gpsd-clients \
  mosquitto \
  iftop iptraf-ng nethogs
```

### 13.2 SSH and VPN

```bash
sudo systemctl enable --now sshd
sudo firewall-cmd --permanent --add-service=ssh
sudo firewall-cmd --reload
```

Add VPN connections through **System Settings → Network → VPN**. The NetworkManager plugins provide OpenVPN and OpenConnect; do not paste a `.ovpn` file into `/etc/openvpn` unless you specifically need a system-wide tunnel.

### 13.3 Samba and NFS

Create a Samba share for the NTFS data disk:

```bash
sudo tee -a /etc/samba/smb.conf >/dev/null <<'EOF'

[DATA]
path = /media/DATA
browseable = yes
read only = no
valid users = user
create mask = 0664
directory mask = 0775
EOF

sudo smbpasswd -a user
sudo systemctl enable --now smb nmb
sudo firewall-cmd --permanent --add-service=samba
sudo firewall-cmd --reload
```

The NTFS mount in [Section 3.5](#35-mount-the-ntfs-data-disk) must exist before Samba starts. `smbpasswd` sets a Samba-only password; it does not change the Linux password.

NFS is optional. Export the same directory only on a trusted LAN:

```bash
echo '/media/DATA 192.168.1.0/24(rw,sync,no_subtree_check)' | sudo tee -a /etc/exports
sudo exportfs -ra
sudo systemctl enable --now nfs-server
sudo firewall-cmd --permanent --add-service=nfs
sudo firewall-cmd --permanent --add-service=mountd
sudo firewall-cmd --permanent --add-service=rpc-bind
sudo firewall-cmd --reload
```

Replace `192.168.1.0/24` with your actual LAN. Never export the NTFS disk to an untrusted network.

### 13.4 Verify

```bash
systemctl is-active sshd smb nmb nfs-server
testparm -s
showmount -e localhost
smbclient -L localhost -U user
ip -br address
nmcli device status
```

`smbclient` asks for the Samba password. `systemctl is-active` should print `active` for every unit you enabled.

---

## 14. Virtualization and Containers

### 14.1 KVM, QEMU and libvirt

```bash
sudo dnf group install -y virtualization
sudo dnf install -y libvirt-client virt-manager virt-install edk2-ovmf swtpm guestfs-tools
sudo systemctl enable --now libvirtd
sudo usermod -aG libvirt,kvm "$USER"

sudo virsh net-list --all
sudo virsh net-start default
sudo virsh net-autostart default
```

Log out and back in for the groups. `virt-manager` and `virt-viewer` already ship desktop entries.

### 14.2 VirtualBox 7.2

VirtualBox is in RPM Fusion **free**, not nonfree. The kernel module is built by akmods. If Secure Boot is enabled, enroll the akmods key before installing the module:

```bash
sudo dnf install -y akmods mokutil kernel-devel
sudo kmodgenca -a
sudo mokutil --import /etc/pki/akmods/certs/public_key.der
```

Reboot. The firmware shows the **MOK management** screen; choose **Enroll MOK → Continue → Yes**, enter the password you set, and reboot again. If Secure Boot is disabled, skip the `kmodgenca`/`mokutil` commands.

Then install and load VirtualBox:

```bash
sudo dnf install -y VirtualBox akmod-VirtualBox
sudo systemctl enable --now vboxdrv
sudo usermod -aG vboxusers "$USER"
```

Log out and back in for `vboxusers`. `VirtualBox` is a meta package; `VBoxManage`, the service and the kernel module live in `VirtualBox-server`/`akmod-VirtualBox`, which DNF pulls automatically.

### 14.3 Docker

```bash
sudo systemctl enable --now docker
sudo docker run --rm hello-world
```

If the second command says permission denied, log out and back in; `usermod -aG docker` was applied in [Section 4.7](#47-docker-ce).

### 14.4 Podman and Distrobox

```bash
sudo dnf install -y podman podman-compose buildah skopeo distrobox
podman info
distrobox --version
```

Podman is rootless and does not need a daemon. Use either Docker or Podman for a given project, not both against the same containers.

### 14.5 Databases and MQTT

```bash
sudo dnf install -y postgresql-server postgresql-contrib valkey mosquitto
sudo postgresql-setup --initdb
sudo tee /etc/mosquitto/mosquitto.conf >/dev/null <<'EOF'
listener 1883 localhost
allow_anonymous true
EOF
sudo systemctl enable --now postgresql valkey mosquitto
```

Verify:

```bash
sudo -u postgres psql -c 'SELECT version();'
valkey-cli ping
mosquitto_pub -h localhost -t test -m hello
```

PostgreSQL 18 is initialized once. `valkey` is Fedora's Redis-compatible server; do not install the retired `redis` package. The Mosquitto configuration above listens only on `localhost` and allows anonymous local clients for testing. Change `/etc/mosquitto/mosquitto.conf` only if you need remote clients, and add authentication before exposing it.

### 14.6 Verify

```bash
VBoxManage --version
systemctl is-active vboxdrv libvirtd docker
virsh list --all
docker info --format '{{.ServerVersion}}'
podman info --format '{{.Version.Version}}'
psql --version
valkey-cli ping
```

`virsh net-list --all` should show the `default` network as active; `virsh list --all` lists guests. `VBoxManage --version` prints the RPM Fusion build; do not compare it to a hard-coded upstream version. RPM Fusion can lag Oracle, so check `dnf info VirtualBox` before assuming you have the newest release.

## 15. Desktop Applications

### 15.1 Native RPM applications

```bash
sudo dnf install -y \
  firefox thunderbird telegram-desktop \
  keepassxc kleopatra \
  gimp inkscape \
  vlc haruna \
  obs-studio HandBrake-gui \
  audacity kdenlive \
  okular ark filelight kcalc kde-partitionmanager \
  filezilla remmina \
  guvcview transmission-gtk bleachbit \
  hardinfo2
```

These packages all ship their own desktop entries. Do not create duplicate launchers for them.

### 15.2 Flatpak applications

Install the applications that are not packaged natively, or where the Flatpak is the better-maintained build:

```bash
flatpak install -y flathub \
  com.discordapp.Discord \
  com.slack.Slack \
  com.rtosta.zapzap \
  md.obsidian.Obsidian \
  com.github.xournalpp.xournalpp \
  io.github.aandrew_me.ytdn \
  com.github.tchx84.Flatseal \
  org.gnome.NetworkDisplays \
  org.signal.Signal \
  org.onlyoffice.desktopeditors \
  org.zaproxy.ZAP \
  com.getpostman.Postman \
  us.zoom.Zoom
```

Flatpak installs its own desktop entries and updates through `flatpak update`. If a Flatpak needs access to the data disk, grant it explicitly:

```bash
flatpak override --user --filesystem=/media/DATA com.rtosta.zapzap
```

Change the application ID to the one that actually needs access; do not grant every Flatpak the whole disk.

### 15.3 AppImages

Copy the HTTP Toolkit and LM Studio AppImages from the old disk, or download them from their official sites. Install the FUSE runtime first; AppImages need it unless you extract them:

```bash
sudo dnf install -y fuse fuse-libs
mkdir -p "$HOME/Applications"
find "$HOME/Applications" -maxdepth 1 -type f -name '*.AppImage' -exec chmod +x {} +
```

If an AppImage still fails, run it with `--appimage-extract-and-run` instead of installing a second FUSE stack.

The launcher generator in [Section 22](#22-desktop-launchers) creates a menu entry for every executable AppImage in `~/Applications`.

### 15.4 Gaming

The HD 4600 is far below the GPU expected by modern Proton titles. Older and 2D games work; current 3D games will not be playable.

```bash
sudo dnf install -y steam
sudo dnf install -y mesa-dri-drivers.i686 mesa-vulkan-drivers.i686
```

The RPM Fusion `steam` package pulls 32-bit multilib dependencies and the Fedora `steam-devices` package automatically. NTSYNC is already in the Fedora 44 kernel and needs no configuration.

### 15.5 Verify

```bash
rpm -q firefox thunderbird telegram-desktop keepassxc gimp vlc obs-studio HandBrake-gui
flatpak list --app --columns=application | sort
ls -l "$HOME"/Applications/*.AppImage 2>/dev/null || true
```

---

## 16. Fonts, Printing and Scanning

### 16.1 Fonts

```bash
sudo dnf install -y \
  google-noto-sans-fonts google-noto-serif-fonts google-noto-sans-mono-fonts \
  google-noto-emoji-fonts google-noto-sans-arabic-fonts \
  dejavu-sans-fonts dejavu-serif-fonts dejavu-sans-mono-fonts \
  liberation-fonts-all jetbrains-mono-fonts-all fira-code-fonts \
  adobe-source-code-pro-fonts fontawesome-fonts-all

fc-cache -f -v
```

Verify Arabic coverage:

```bash
fc-list :lang=ar | head
fc-list | wc -l
```

### 16.2 Printing

```bash
sudo dnf install -y cups hplip hplip-gui system-config-printer foomatic-db gutenprint
sudo systemctl enable --now cups
```

Open **HP Device Manager** or **Print Settings** from the menu. `hplip` is for HP printers; `system-config-printer` is the generic CUPS front end.

### 16.3 Scanning

```bash
sudo dnf install -y simple-scan sane-backends sane-airscan
scanimage -L
```

`sane-airscan` handles many network scanners without a vendor driver. `simple-scan` is the desktop front end. For an HP all-in-one, use **HP Device Manager** if `scanimage -L` does not detect it.

### 16.4 Verify

```bash
fc-list | wc -l
lpstat -r
scanimage -L
```

`lpstat -r` should print `scheduler is running`. `scanimage -L` may print `No scanners were identified` if no scanner is connected; that is not a package failure.

---

## 17. Vendor and Manual Installs

### 17.1 Android Studio and SDK

The SDK and AVDs are portable; copy them instead of downloading several gigabytes again.

```bash
sudo dnf install -y android-tools

STUDIO_URL="$(curl -sL https://developer.android.com/studio \
  | grep -oE 'https://[^"]*android-studio-[^"]*-linux\.tar\.gz' | head -1)"
[[ -n "$STUDIO_URL" ]] || { echo "ERROR: could not resolve the Android Studio URL"; exit 1; }
curl -L -o /tmp/android-studio.tar.gz "$STUDIO_URL"
sudo rm -rf /opt/android-studio
sudo tar -xzf /tmp/android-studio.tar.gz -C /opt
```

Add the environment to `~/.bashrc`:

```bash
cat >> "$HOME/.bashrc" <<'EOF'

# Android
export STUDIO_JDK="/opt/android-studio/jbr"
export ANDROID_HOME="$HOME/Android/Sdk"
export ANDROID_SDK_ROOT="$HOME/Android/Sdk"
export ANDROID_AVD_HOME="$HOME/.android/avd"
export PATH="$ANDROID_HOME/emulator:$ANDROID_HOME/platform-tools:$ANDROID_HOME/cmdline-tools/latest/bin:$PATH"
EOF
```

Copy `~/Android/Sdk` and `~/.android` from the backup. The AVD path is the default `$HOME/.android/avd`; the old `~/.config/.android/avd` path is not used by current Android Studio and conflicts with the copied `~/.android` directory.

`STUDIO_JDK` points at the JetBrains runtime bundled with Studio. Do not replace it with the system JDK.

### 17.2 apktool

```bash
sudo dnf install -y java-25-openjdk-devel
sudo curl -L -o /usr/local/bin/apktool \
  https://raw.githubusercontent.com/iBotPeaches/Apktool/master/scripts/linux/apktool
sudo curl -L -o /usr/local/bin/apktool.jar \
  "$(curl -s https://api.github.com/repos/iBotPeaches/Apktool/releases/latest \
     | grep -o 'https://[^"]*apktool_[0-9.]*\.jar' | head -1)"
sudo chmod +x /usr/local/bin/apktool
apktool --version
```

The wrapper expects the JAR to be named `apktool.jar` in the same directory. apktool 3.x runs on OpenJDK 25.

### 17.3 jadx

```bash
mkdir -p "$HOME/github-tools"
JADX_URL="$(curl -s https://api.github.com/repos/skylot/jadx/releases/latest \
  | grep -o 'https://[^"]*jadx-[0-9.]*\.zip' | head -1)"
[[ -n "$JADX_URL" ]] || { echo "ERROR: could not resolve the jadx release URL"; exit 1; }
curl -L -o /tmp/jadx.zip "$JADX_URL"
rm -rf "$HOME/github-tools/jadx"
unzip -q /tmp/jadx.zip -d "$HOME/github-tools/jadx"
chmod +x "$HOME/github-tools/jadx/bin/jadx" "$HOME/github-tools/jadx/bin/jadx-gui"
"$HOME/github-tools/jadx/bin/jadx" --version
```

The GUI is `~/github-tools/jadx/bin/jadx-gui`.

### 17.4 Ghidra

Install the latest public Ghidra release. Current releases require a modern JDK; the system OpenJDK is sufficient.

```bash
GHIDRA_URL="$(curl -s https://api.github.com/repos/NationalSecurityAgency/ghidra/releases/latest \
  | grep -o 'https://[^"]*_PUBLIC_[0-9]*\.zip' | head -1)"
[[ -n "$GHIDRA_URL" ]] || { echo "ERROR: could not resolve the Ghidra release URL"; exit 1; }
curl -L -o /tmp/ghidra.zip "$GHIDRA_URL"
sudo unzip -q /tmp/ghidra.zip -d /opt
sudo ln -sfn "$(find /opt -maxdepth 1 -type d -name 'ghidra_*_PUBLIC' | sort -V | tail -1)" /opt/ghidra
sudo chmod +x /opt/ghidra/ghidraRun
ls -d /opt/ghidra_*
```

Create a small wrapper so Ghidra uses the active system JDK instead of a hard-coded path:

```bash
sudo tee /usr/local/bin/ghidra >/dev/null <<'EOF'
#!/usr/bin/env bash
set -euo pipefail
JAVA_HOME="${JAVA_HOME:-$(dirname "$(dirname "$(readlink -f "$(command -v javac)")")")}"
export JAVA_HOME
exec /opt/ghidra/ghidraRun "$@"
EOF
sudo chmod 0755 /usr/local/bin/ghidra
```

Launch with `ghidra` (or `/opt/ghidra/ghidraRun`). Do not copy an old Ghidra installation that was built against an older JDK; copy only projects.

### 17.5 Burp Suite Community

Download the Linux installer from <https://portswigger.net/burp/communitydownload>, then:

```bash
chmod +x burpsuite_community_linux_*.sh
sudo ./burpsuite_community_linux_*.sh
sudo ln -sf "$(find /opt /usr/local -maxdepth 3 -type f -name BurpSuiteCommunity | head -1)" \
  /usr/local/bin/burpsuite
command -v burpsuite
```

The installer uses `/opt/BurpSuiteCommunity` or `/usr/local/BurpSuiteCommunity` depending on the choices in the installer. The symlink makes the command predictable; project data lives in `~/.BurpSuite`. The installer may create a menu entry; if it does, remove the generated `tool-burp.desktop` to avoid a duplicate.

### 17.6 ONLYOFFICE

Use the Flatpak installed in [Section 15.2](#152-flatpak-applications). Do not also enable the vendor DNF repository; one installation source is enough and avoids mixing two update paths.

### 17.7 Ollama

Use the upstream installer, not Fedora's `ollama` RPM. The Fedora package depends on `hipblas` and `rocm-hip`; on this Intel-only laptop that pulls in the AMD/ROCm stack for no benefit.

```bash
curl -fsSL https://ollama.com/install.sh | sh
systemctl is-active ollama
ollama --version
```

The installer creates the `ollama` service user with home `/usr/share/ollama`, so the service's models live in `/usr/share/ollama/.ollama/models`. If you have old models in `~/.ollama/models`, copy them there once:

```bash
sudo install -d -o ollama -g ollama /usr/share/ollama/.ollama
if [ -d "$HOME/.ollama/models" ]; then
  sudo rsync -a --info=progress2 "$HOME/.ollama/models/" /usr/share/ollama/.ollama/models/
  sudo chown -R ollama:ollama /usr/share/ollama/.ollama
fi
```

Do not put Ollama models on the NTFS disk. NTFS permissions and case handling are a poor fit for the model store. If you need more space, use an ext4 partition or an external ext4 drive and set `OLLAMA_MODELS` in a systemd drop-in for `ollama.service`.

Verify the API:

```bash
curl -s http://127.0.0.1:11434/api/tags
```

### 17.8 Ventoy

```bash
mkdir -p "$HOME/.local/bin"
cd "$HOME/github-tools"
curl -LO "$(curl -s https://api.github.com/repos/ventoy/Ventoy/releases/latest \
  | grep -o 'https://[^"]*ventoy-[0-9.]*-linux\.tar\.gz' | head -1)"
tar xf ventoy-*-linux.tar.gz
ln -sfn "$(find "$HOME/github-tools" -maxdepth 2 -type f -name VentoyGUI.x86_64 | head -1)" \
  "$HOME/.local/bin/ventoy-gui"
```

Ventoy needs root access to write the USB device; the GUI asks for it through PolicyKit. Use `Ventoy2Disk.sh` in the same directory for the command-line version.

### 17.9 Verify

```bash
adb version
apktool --version
"$HOME/github-tools/jadx/bin/jadx" --version
ls -d /opt/ghidra_* /opt/android-studio 2>/dev/null
systemctl is-active ollama
ollama --version
"$HOME/.local/bin/ventoy-gui" --help >/dev/null 2>&1 || true
```

`ollama` must be `active`. The Ventoy GUI may not support `--help`; the important check is that the symlink exists.

## 18. Hardware and System Configuration

### 18.1 Kernel modules and sysctl

Do not copy `/etc/modprobe.d` wholesale. A stale option for a driver that no longer exists looks like missing hardware. Start with only the safe, tested options:

```bash
sudo tee /etc/modprobe.d/99-zbook.conf >/dev/null <<'EOF'
# Silence the PC speaker; safe on this laptop.
blacklist pcspkr
EOF
```

If you previously used a tested i915 power-saving option and it improved this exact machine, copy that one file and rebuild the initramfs:

```bash
sudo dracut -f
```

For development tools that watch many files:

```bash
sudo tee /etc/sysctl.d/99-zbook.conf >/dev/null <<'EOF'
fs.inotify.max_user_watches=524288
EOF
sudo sysctl --system
```

Do not add random `vm.swappiness`, I/O scheduler or GPU options. Measure first; the defaults on Fedora 44 are already tuned for this hardware.

### 18.2 Out-of-tree Wi-Fi

The Intel Wireless 7260 uses the in-tree `iwlwifi` driver. Do not install Broadcom or Realtek out-of-tree drivers for it. If you add an external USB adapter for monitor mode, install only that vendor's driver and keep it out of the base system until it is proven stable.

### 18.3 Services

Enable only the services you use:

| Service | Enable when |
|---|---|
| `sshd` | You need remote SSH |
| `smb` and `nmb` | You use the Samba share |
| `nfs-server` | You export NFS |
| `libvirtd` | You use KVM/QEMU |
| `docker` | You use Docker |
| `vboxdrv` | You use VirtualBox |
| `ollama` | Installed by the Ollama script |
| `postgresql` | You use PostgreSQL |
| `valkey` | You use Valkey |
| `mosquitto` | You use MQTT |

```bash
# Enable only the lines that match the services you actually use.
# sudo systemctl enable --now sshd
# sudo systemctl enable --now smb nmb
# sudo systemctl enable --now nfs-server
# sudo systemctl enable --now libvirtd
# sudo systemctl enable --now docker
# sudo systemctl enable --now vboxdrv
# sudo systemctl enable --now postgresql valkey mosquitto
```

Do not run a command for a service you do not need. `systemctl --failed` is the final authority.

### 18.4 Virtualization firmware and network

```bash
sudo virsh net-list --all
sudo virsh net-start default
sudo virsh net-autostart default
sudo virt-host-validate qemu
```

`virt-host-validate` reports whether KVM, IOMMU and the required libraries are usable. On this CPU, hardware virtualization must be enabled in the HP firmware.

### 18.5 Verify

```bash
findmnt /media/DATA
zramctl
swapon --show
cat /proc/swaps
cat /proc/cmdline
systemctl --failed
sudo btrfs filesystem usage /
```

`findmnt` must show `ntfs3`. `zramctl` must list `zram0`; `swapon --show` and `/proc/swaps` must show `/dev/zram0` and no disk-backed swap; `/proc/cmdline` must not contain `resume=`.

---

## 19. KDE Plasma Desktop

### 19.1 Login manager and session

A fresh Fedora 44 KDE installation uses **Plasma Setup** and **Plasma Login Manager**, not SDDM. Do not install or configure SDDM on this system. The login manager package is `plasma-login-manager`; it is part of the KDE edition.

This guide is Wayland-only. Do **not** install `plasma-workspace-x11`, and do not choose an X11 session at login. At the Plasma Login Manager screen, select **Plasma (Wayland)** if the session menu appears. XWayland is only a compatibility layer for unported applications; it is not an Xorg session.

Do not use `versionlock` or `exclude` to pin Plasma alone: that can leave Plasma, Frameworks and KDE apps at incompatible versions. Take a Snapper snapshot, run `sudo dnf upgrade --refresh`, and review the transaction before accepting a Plasma upgrade.

### 19.2 KDE packages and thumbnails

```bash
sudo dnf install -y \
  kdegraphics-thumbnailers \
  ffmpegthumbs \
  kio-extras \
  plasma-browser-integration
```

These provide video thumbnails in Dolphin, extra KIO protocols and the browser integration service.

### 19.3 Settings to apply by hand

Open **System Settings** and set:

- **Appearance → Global Theme:** Breeze (the Fedora default).
- **Appearance → Colors:** Breeze.
- **Appearance → Fonts:** Noto Sans, 10 pt; Noto Sans Arabic for Arabic; enable **Sub-pixel rendering: RGB** and **Hinting: Slight**.
- **Display and Monitor:** native 1920×1080 at 100%; enable Night Color for evening use.
- **Input Devices → Touchpad:** tap-to-click, two-finger scrolling, natural scrolling if preferred.
- **Power Management:** use the **Balanced** profile; switch to **Power Saver** on battery.
- **Shortcuts:** assign a key to **Spectacle** for screenshots.
- **Panel:** Application Menu, Task Manager, System Tray, Digital Clock.

Do not install Kvantum, Latte Dock, or a third-party global theme on a fresh install. They add maintenance work and can break after a Plasma minor update.

### 19.4 Screenshots and desktop behavior

Spectacle is installed with Plasma. Use it for rectangular, active-window and full-screen captures. KRunner is available with `Alt+Space`; Dolphin opens with `Meta+E`; the application menu is the left panel button.

### 19.5 Verify

```bash
plasmashell --version
echo "$XDG_SESSION_TYPE"
kwin_wayland --version
kreadconfig6 --file kdeglobals --group General --key ColorScheme
```

`XDG_SESSION_TYPE` must be `wayland`; anything else means the wrong session was selected.

---

## 20. Migration and Dotfiles

### 20.1 Copy the data back

With the backup disk mounted at `/media/BACKUP`:

```bash
rsync -a --info=progress2 /media/BACKUP/THE_DATABASE/ "$HOME/THE_DATABASE/"
rsync -a --info=progress2 /media/BACKUP/github-tools/ "$HOME/github-tools/"
rsync -a --info=progress2 /media/BACKUP/Android/ "$HOME/Android/"
rsync -a --info=progress2 /media/BACKUP/.android/ "$HOME/.android/"
rsync -a --info=progress2 /media/BACKUP/.ssh/ "$HOME/.ssh/"
rsync -a --info=progress2 /media/BACKUP/.gnupg/ "$HOME/.gnupg/"
rsync -a --info=progress2 /media/BACKUP/.BurpSuite/ "$HOME/.BurpSuite/"
```

Copy browser profiles selectively:

```bash
rsync -a /media/BACKUP/.mozilla/ "$HOME/.mozilla/"
rsync -a /media/BACKUP/.thunderbird/ "$HOME/.thunderbird/"
rsync -a /media/BACKUP/.config/google-chrome/ "$HOME/.config/google-chrome/"
rsync -a /media/BACKUP/.config/BraveSoftware/ "$HOME/.config/BraveSoftware/"
rsync -a /media/BACKUP/.config/Code/User/ "$HOME/.config/Code/User/"
```

Do not copy the entire old `~/.config`; it contains XFCE, LightDM and MX settings that do not belong in Plasma.

### 20.2 Shell environment

The Android block was added in [Section 17.1](#171-android-studio-and-sdk). Add the general environment once:

```bash
cat >> "$HOME/.bashrc" <<'EOF'

# General environment
export EDITOR=nano
export GOPATH="$HOME/go"
export PATH="$HOME/.local/bin:$HOME/go/bin:$HOME/.cargo/bin:/usr/local/go/bin:$PATH"
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && . "$NVM_DIR/nvm.sh"

# Java: resolve the active system JDK; override per project with Temurin 17.
export JAVA_HOME="$(dirname "$(dirname "$(readlink -f "$(command -v javac)")")")"
EOF

source "$HOME/.bashrc"
```

The AVD path is deliberately the default:

```bash
echo "$ANDROID_AVD_HOME"     # /home/user/.android/avd
echo "$ANDROID_HOME"         # /home/user/Android/Sdk
```

### 20.3 Aliases

```bash
cat >> "$HOME/.bashrc" <<'EOF'

# Aliases
alias ll='eza -al --group-directories-first'
alias la='eza -a'
alias lt='eza --tree --level=2'
alias df='df -h'
alias free='free -h'
alias update='sudo dnf upgrade --refresh'
alias snap='sudo snapper -c root list'
alias gs='git status'
EOF

source "$HOME/.bashrc"
```

### 20.4 Where everything lands

| Data | Path |
|---|---|
| Notes and documents | `~/THE_DATABASE` |
| Cloned tools | `~/github-tools` |
| Android SDK | `~/Android/Sdk` |
| Android AVDs | `~/.android/avd` |
| Go binaries | `~/go/bin` |
| Rust binaries | `~/.cargo/bin` |
| pipx binaries | `~/.local/bin` |
| Ollama service models | `/usr/share/ollama/.ollama/models` |
| Burp projects | `~/.BurpSuite` |
| Snapper snapshots | `/.snapshots` (Btrfs subvolume `@snapshots`) |

### 20.5 Browser profiles

Chrome, Brave and Firefox profiles contain cookies and saved logins. Copy them only while the browsers are closed. After the first launch, sign in again if a profile was tied to a different device or keyring.

### 20.6 Verify

```bash
test -d "$HOME/THE_DATABASE" && echo THE_DATABASE
test -d "$HOME/github-tools" && echo github-tools
test -d "$HOME/Android/Sdk" && echo Android-SDK
test -d "$HOME/.android/avd" && echo AVDs
command -v go rustc cargo node npm java python3
```

---

## 21. What Not to Migrate

| Old item | Why not | Fedora 44 replacement |
|---|---|---|
| MX tools (`mx-remaster`, `live-usb-maker`, `repo-manager`, `bootrepair`, `cleanup`, `tweak`, `codecs`, `service-manager`, `user-installed-packages`, `apt-notifier`, `ddm`) | Debian/MX-specific | Snapper, DNF, KDE Partition Manager, Plasma System Monitor |
| XFCE stack (`xfce4-*`, `xfwm4`, `thunar*`, `tumbler`, `exo`, `mousepad`, `orage`, `ristretto`, `xfburn`, `catfish`) | Replaced by Plasma | Dolphin, Kate, Gwenview, Spectacle |
| LightDM, `lightdm-gtk-greeter`, `slick-greeter`, `light-locker` | Fedora 44 KDE uses Plasma Login Manager | Plasma Login Manager |
| SDDM configuration | Not used by a fresh Fedora 44 KDE install | Plasma Login Manager |
| compton, `zcompton`, `compton-conf`, `multiload-ng`, `volumeicon-alsa`, `gxkb`, `clipit` | KWin and Plasma widgets replace them | KWin, Plasma panel |
| AppArmor and `apparmor-utils` | Fedora uses SELinux | SELinux |
| ufw and gufw | Fedora uses firewalld | firewalld |
| The OBS server stack (`obs-server`, `obs-worker`, `obs-cli`, `obs-utils`, `obs-build`, `obs-productconverter`) | A server build system, not a desktop tool | Not installed |
| NVIDIA, CUDA, ROCm, AMDGPU and `nvidia-*` packages | No such hardware | Intel i915/i965 |
| Old kernels, DKMS builds and VirtualBox images | Not portable across releases | Rebuild modules; recreate VMs |
| Python 2 virtual environments | End of life | Python 3 virtual environments |
| `p7zip` | Retired in Fedora | `7zip` |
| `redis` | Retired in Fedora | `valkey` |
| Other snapshot managers | Two snapshot schedules on one Btrfs root cause duplicate snapshots and ambiguous rollback | Snapper only |

## 22. Desktop Launchers

### 22.1 What already has a launcher

Do not duplicate packages that already install a `.desktop` file. In this guide that includes:

`htop`, `btop`, `hardinfo2`, `bleachbit`, `wireshark`, `ettercap`, `cutter-re`, `clamtk`, `chkrootkit`, `transmission-gtk`, `guvcview`, `filezilla`, `remmina`, `virt-viewer`, `putty`, `VirtualBox`, `HandBrake-gui`, `gimp`, `inkscape`, `vlc`, `haruna`, `obs-studio`, `audacity`, `kdenlive`, `okular`, `ark`, `filelight`, `kcalc`, `kde-partitionmanager`, `firewall-config`, `system-config-printer`, `simple-scan`, `hplip-gui`, and every Flatpak.

The launcher table below contains only tools that do **not** ship a usable menu entry. It covers the interactive command-line tools; each one opens Konsole, runs the tool, and leaves you in a shell when the tool exits. Non-interactive helpers and compilers (`curl`, `jq`, `gcc`) are intentionally not given menu entries.

### 22.2 Terminal wrapper

```bash
install -d "$HOME/.local/bin"

cat > "$HOME/.local/bin/desktop-term" <<'EOF'
#!/usr/bin/env bash
set -euo pipefail
if (($# == 0)); then
  exec /usr/bin/konsole
fi
printf -v cmd '%q ' "$@"
exec /usr/bin/konsole -e bash -lc "source \"$HOME/.bashrc\"; $cmd; echo; exec bash"
EOF

chmod 0755 "$HOME/.local/bin/desktop-term"
```

`desktop-term nmap` opens Konsole, runs `nmap`, and leaves a shell open. Commands that need `sudo` prompt inside the terminal.

### 22.3 Launcher generator

```bash
cat > "$HOME/.local/bin/mk-launchers" <<'EOF'
#!/usr/bin/env bash
set -euo pipefail
apps="$HOME/.local/share/applications"
install -d "$apps"

while IFS='|' read -r file name exec icon comment categories; do
  [[ -n "${file:-}" ]] || continue
  exec="${exec//@HOME@/$HOME}"
  icon="${icon//@HOME@/$HOME}"
  cat > "$apps/$file.desktop" <<DESKTOP
[Desktop Entry]
Type=Application
Version=1.0
Name=$name
Comment=$comment
Exec=$exec
Icon=$icon
Terminal=false
Categories=${categories:-Utility;}
StartupNotify=true
DESKTOP
  chmod 0644 "$apps/$file.desktop"
done

if command -v kbuildsycoca6 >/dev/null 2>&1; then
  kbuildsycoca6 >/dev/null 2>&1 || true
fi
EOF

chmod 0755 "$HOME/.local/bin/mk-launchers"
```

Run it with the table below. The `@HOME@` placeholder is replaced with `/home/user` when the file is generated.

### 22.4 Launcher table

```bash
"$HOME/.local/bin/mk-launchers" <<'LAUNCHERS'
tool-ncdu|Ncdu|@HOME@/.local/bin/desktop-term ncdu @HOME@|drive-harddisk|Disk usage analyzer|Utility;
tool-duf|Duf|@HOME@/.local/bin/desktop-term duf|drive-harddisk|Filesystem usage|Utility;
tool-tree|Tree|@HOME@/.local/bin/desktop-term tree -L 2 @HOME@|folder|Directory tree|Utility;
tool-rclone|Rclone|@HOME@/.local/bin/desktop-term rclone config|folder-remote|Cloud storage|Utility;
tool-tmux|Tmux|@HOME@/.local/bin/desktop-term tmux|utilities-terminal|Terminal multiplexer|Utility;
tool-screen|Screen|@HOME@/.local/bin/desktop-term screen|utilities-terminal|Terminal multiplexer|Utility;
tool-mc|Midnight Commander|@HOME@/.local/bin/desktop-term mc|system-file-manager|File manager|Utility;
tool-nano|Nano|@HOME@/.local/bin/desktop-term nano|accessories-text-editor|Text editor|Utility;
tool-vim|Vim|@HOME@/.local/bin/desktop-term vim|accessories-text-editor|Text editor|Utility;
tool-neovim|Neovim|@HOME@/.local/bin/desktop-term nvim|accessories-text-editor|Text editor|Utility;
tool-lshw|Hardware Summary|@HOME@/.local/bin/desktop-term sudo lshw -short|computer|Hardware inventory|System;
tool-dmidecode|DMI Information|@HOME@/.local/bin/desktop-term sudo dmidecode -t system|computer|Firmware details|System;
tool-powertop|PowerTOP|@HOME@/.local/bin/desktop-term sudo powertop|battery|Power diagnostics|System;
tool-sensors|Sensors|@HOME@/.local/bin/desktop-term watch -n 2 sensors|utilities-system-monitor|Temperatures and voltages|System;
tool-gnuplot|Gnuplot|@HOME@/.local/bin/desktop-term gnuplot|applications-development|Plotting|Development;
tool-python|Python REPL|@HOME@/.local/bin/desktop-term python3|text-x-python|Python shell|Development;
tool-gdb|GDB|@HOME@/.local/bin/desktop-term gdb|applications-development|Debugger|Development;
tool-lldb|LLDB|@HOME@/.local/bin/desktop-term lldb|applications-development|Debugger|Development;
tool-gh|GitHub CLI|@HOME@/.local/bin/desktop-term gh|applications-development|GitHub command line|Development;
tool-mtr|MTR|@HOME@/.local/bin/desktop-term mtr|network-wired|Network diagnostic|Network;

tool-nmap|Nmap|@HOME@/.local/bin/desktop-term sudo nmap|network-wired|Network scanner|System;
tool-masscan|Masscan|@HOME@/.local/bin/desktop-term sudo masscan|network-wired|Port scanner|System;
tool-zmap|ZMap|@HOME@/.local/bin/desktop-term sudo zmap|network-wired|Internet scanner|System;
tool-arp-scan|ARP Scan|@HOME@/.local/bin/desktop-term sudo arp-scan --localnet|network-wired|Local network scan|System;
tool-hydra|Hydra|@HOME@/.local/bin/desktop-term hydra|security-high|Login cracker|System;
tool-ncrack|Ncrack|@HOME@/.local/bin/desktop-term ncrack|security-high|Authentication cracker|System;
tool-medusa|Medusa|@HOME@/.local/bin/desktop-term medusa|security-high|Login brute force|System;
tool-john|John the Ripper|@HOME@/.local/bin/desktop-term john|security-high|Password cracker|System;
tool-hashcat|Hashcat|@HOME@/.local/bin/desktop-term hashcat|security-high|Password recovery|System;
tool-sqlmap|sqlmap|@HOME@/.local/bin/desktop-term /opt/sqlmap/sqlmap.py|security-high|SQL injection|System;
tool-nikto|Nikto|@HOME@/.local/bin/desktop-term /opt/nikto/program/nikto.pl|security-high|Web server scanner|System;
tool-gobuster|Gobuster|@HOME@/.local/bin/desktop-term gobuster|security-high|Directory brute force|System;
tool-ffuf|ffuf|@HOME@/.local/bin/desktop-term ffuf|security-high|Web fuzzer|System;
tool-whatweb|WhatWeb|@HOME@/.local/bin/desktop-term whatweb|security-high|Web fingerprint|System;
tool-wpscan|WPScan|@HOME@/.local/bin/desktop-term wpscan --help|security-high|WordPress scanner|System;
tool-feroxbuster|Feroxbuster|@HOME@/.local/bin/desktop-term feroxbuster --help|security-high|Content discovery|System;
tool-nuclei|Nuclei|@HOME@/.local/bin/desktop-term nuclei -h|security-high|Template scanner|System;
tool-grype|Grype|@HOME@/.local/bin/desktop-term grype --help|security-high|Vulnerability scanner|System;
tool-syft|Syft|@HOME@/.local/bin/desktop-term syft --help|security-high|SBOM generator|System;
tool-trufflehog|TruffleHog|@HOME@/.local/bin/desktop-term trufflehog --help|security-high|Secret scanner|System;
tool-noseyparker|Nosey Parker|@HOME@/.local/bin/desktop-term noseyparker --help|security-high|Secret scanner|System;
tool-semgrep|Semgrep|@HOME@/.local/bin/desktop-term semgrep --help|security-high|Static analysis|System;
tool-tshark|TShark|@HOME@/.local/bin/desktop-term tshark|network-wired|Packet analyzer|Network;
tool-tcpdump|tcpdump|@HOME@/.local/bin/desktop-term sudo tcpdump -i any|network-wired|Packet capture|Network;
tool-radare2|Radare2|@HOME@/.local/bin/desktop-term r2|applications-development|Reverse engineering|Development;
tool-rizin|Rizin|@HOME@/.local/bin/desktop-term rizin|applications-development|Reverse engineering|Development;
tool-binwalk|Binwalk|@HOME@/.local/bin/desktop-term binwalk|applications-development|Firmware analysis|System;
tool-foremost|Foremost|@HOME@/.local/bin/desktop-term foremost|drive-harddisk|File carving|System;
tool-yara|YARA|@HOME@/.local/bin/desktop-term yara|security-high|Pattern matching|System;
tool-lynis|Lynis|@HOME@/.local/bin/desktop-term sudo lynis audit system|security-high|Security audit|System;
tool-rkhunter|rkhunter|@HOME@/.local/bin/desktop-term sudo rkhunter --check --sk|security-high|Rootkit check|System;
tool-dsniff|dsniff|@HOME@/.local/bin/desktop-term sudo dsniff|security-high|Network auditing|System;
tool-hping3|hping3|@HOME@/.local/bin/desktop-term sudo hping3|network-wired|Packet crafting|System;
tool-subfinder|Subfinder|@HOME@/.local/bin/desktop-term subfinder -h|security-high|Subdomain discovery|System;
tool-dnsx|dnsx|@HOME@/.local/bin/desktop-term dnsx -h|security-high|DNS toolkit|System;
tool-gitleaks|Gitleaks|@HOME@/.local/bin/desktop-term gitleaks --help|security-high|Secret scanner|System;
tool-httpx|httpx|@HOME@/.local/bin/desktop-term httpx -h|security-high|HTTP toolkit|System;
tool-katana|Katana|@HOME@/.local/bin/desktop-term katana -h|security-high|Web crawler|System;
tool-naabu|Naabu|@HOME@/.local/bin/desktop-term naabu -h|security-high|Port scanner|System;
tool-gau|gau|@HOME@/.local/bin/desktop-term gau --help|security-high|URL discovery|System;
tool-jsluice|jsluice|@HOME@/.local/bin/desktop-term jsluice --help|security-high|JavaScript analysis|System;
tool-bettercap|Bettercap|@HOME@/.local/bin/desktop-term sudo bettercap|network-wired|Network attack framework|System;
tool-gowitness|gowitness|@HOME@/.local/bin/desktop-term gowitness --help|security-high|Web screenshots|System;
tool-amass|Amass|@HOME@/.local/bin/desktop-term amass -h|security-high|Attack surface mapping|System;
tool-cewl|CeWL|@HOME@/.local/bin/desktop-term cewl --help|security-high|Wordlist generator|System;
tool-dirsearch|dirsearch|@HOME@/.local/bin/desktop-term dirsearch --help|security-high|Web path discovery|System;
tool-drozer|Drozer|@HOME@/.local/bin/desktop-term drozer --help|security-high|Android security|System;
tool-frida|Frida|@HOME@/.local/bin/desktop-term frida --help|security-high|Dynamic instrumentation|System;
tool-objection|Objection|@HOME@/.local/bin/desktop-term objection --help|security-high|Mobile runtime exploration|System;
tool-sherlock|Sherlock|@HOME@/.local/bin/desktop-term sherlock --help|security-high|Username search|System;
tool-gallery-dl|gallery-dl|@HOME@/.local/bin/desktop-term gallery-dl --help|folder-download|Image gallery downloader|Network;
tool-litecli|LiteCLI|@HOME@/.local/bin/desktop-term litecli|network-server|SQLite shell|Development;
tool-cloud-enum|Cloud Enum|@HOME@/.local/bin/desktop-term cloud_enum --help|security-high|Cloud reconnaissance|System;
tool-retire|Retire.js|@HOME@/.local/bin/desktop-term retire --help|security-high|JavaScript vulnerabilities|System;

tool-airmon-ng|airmon-ng|@HOME@/.local/bin/desktop-term sudo airmon-ng|network-wireless|Monitor mode|Network;
tool-airodump-ng|airodump-ng|@HOME@/.local/bin/desktop-term sudo airodump-ng|network-wireless|Wireless capture|Network;
tool-aireplay-ng|aireplay-ng|@HOME@/.local/bin/desktop-term sudo aireplay-ng|network-wireless|Packet injection|Network;
tool-aircrack-ng|aircrack-ng|@HOME@/.local/bin/desktop-term aircrack-ng|network-wireless|WPA/WEP cracking|Network;
tool-reaver|Reaver|@HOME@/.local/bin/desktop-term sudo reaver|network-wireless|WPS attack|Network;
tool-bully|Bully|@HOME@/.local/bin/desktop-term sudo bully|network-wireless|WPS attack|Network;
tool-mdk4|MDK4|@HOME@/.local/bin/desktop-term sudo mdk4|network-wireless|Wi-Fi testing|Network;
tool-hcxdumptool|hcxdumptool|@HOME@/.local/bin/desktop-term sudo hcxdumptool|network-wireless|PMKID capture|Network;
tool-hcxtools|hcxtools|@HOME@/.local/bin/desktop-term hcxpcapngtool --help|network-wireless|Handshake conversion|Network;
tool-kismet|Kismet|@HOME@/.local/bin/desktop-term sudo kismet|network-wireless|Wireless detector|Network;
tool-wifite|Wifite|@HOME@/.local/bin/desktop-term sudo wifite|network-wireless|Wireless auditor|Network;
tool-pixiewps|Pixiewps|@HOME@/.local/bin/desktop-term pixiewps --help|network-wireless|WPS offline|Network;
tool-cowpatty|Cowpatty|@HOME@/.local/bin/desktop-term cowpatty|network-wireless|WPA cracking|Network;
tool-iw|iw|@HOME@/.local/bin/desktop-term iw dev|network-wireless|Wireless configuration|Network;
tool-macchanger|MAC Changer|@HOME@/.local/bin/desktop-term sudo macchanger|network-wireless|MAC address|Network;
tool-rfkill|rfkill|@HOME@/.local/bin/desktop-term rfkill list|network-wireless|Radio status|Network;
tool-nmcli|nmcli|@HOME@/.local/bin/desktop-term nmcli|network-wired|NetworkManager CLI|Network;
tool-nmtui|nmtui|@HOME@/.local/bin/desktop-term nmtui|network-wired|NetworkManager TUI|Network;

tool-iftop|iftop|@HOME@/.local/bin/desktop-term sudo iftop|network-wired|Bandwidth monitor|Network;
tool-iptraf-ng|IPTraf-ng|@HOME@/.local/bin/desktop-term sudo iptraf-ng|network-wired|Network monitor|Network;
tool-nethogs|NetHogs|@HOME@/.local/bin/desktop-term sudo nethogs|network-wired|Per-process bandwidth|Network;
tool-ssh|SSH|@HOME@/.local/bin/desktop-term ssh|network-vpn|Secure shell|Network;
tool-smbclient|Samba Client|@HOME@/.local/bin/desktop-term smbclient -L localhost -U user|network-wired|SMB client|Network;
tool-showmount|NFS Exports|@HOME@/.local/bin/desktop-term showmount -e localhost|network-wired|List NFS exports|Network;
tool-mosquitto-pub|MQTT Publish|@HOME@/.local/bin/desktop-term mosquitto_pub -h localhost -t test -m hello|network-wired|Publish a test message|Network;
tool-mosquitto-sub|MQTT Subscribe|@HOME@/.local/bin/desktop-term mosquitto_sub -h localhost -t test|network-wired|Subscribe to a test topic|Network;
tool-valkey-cli|Valkey CLI|@HOME@/.local/bin/desktop-term valkey-cli|network-server|Valkey shell|Development;
tool-psql|PostgreSQL|@HOME@/.local/bin/desktop-term sudo -u postgres psql|network-server|PostgreSQL shell|Development;
tool-virsh|Virsh|@HOME@/.local/bin/desktop-term virsh|network-server|Libvirt shell|System;
tool-VBoxManage|VBoxManage|@HOME@/.local/bin/desktop-term VBoxManage|application-x-executable|VirtualBox CLI|System;
tool-docker|Docker|@HOME@/.local/bin/desktop-term docker|application-x-executable|Container CLI|Development;
tool-podman|Podman|@HOME@/.local/bin/desktop-term podman|application-x-executable|Container CLI|Development;
tool-distrobox|Distrobox|@HOME@/.local/bin/desktop-term distrobox|application-x-executable|Container distro|Development;
tool-adb|ADB|@HOME@/.local/bin/desktop-term adb devices|phone|Android debug bridge|Development;
tool-apktool|APKTool|@HOME@/.local/bin/desktop-term apktool|applications-development|APK reverse engineering|Development;
tool-jadx|Jadx GUI|@HOME@/github-tools/jadx/bin/jadx-gui|applications-development|Dex decompiler|Development;
tool-jadx-cli|Jadx CLI|@HOME@/.local/bin/desktop-term @HOME@/github-tools/jadx/bin/jadx|applications-development|Dex decompiler CLI|Development;
tool-ghidra|Ghidra|ghidra|applications-development|Reverse engineering|Development;
tool-android-studio|Android Studio|/opt/android-studio/bin/studio.sh|applications-development|Android IDE|Development;
tool-burp|Burp Suite|burpsuite|security-high|Web security testing|System;
tool-ventoy|Ventoy|@HOME@/.local/bin/ventoy-gui|drive-removable-media|USB installer|System;
tool-ollama|Ollama|@HOME@/.local/bin/desktop-term ollama|application-x-executable|Local LLM runtime|Development;
tool-gpsmon|GPS Monitor|@HOME@/.local/bin/desktop-term gpsmon|network-wireless|Live GPS data|Network;
tool-cgps|CGPS|@HOME@/.local/bin/desktop-term cgps|network-wireless|GPS client|Network;
LAUNCHERS
```

### 22.5 AppImage launchers

```bash
cat > "$HOME/.local/bin/mk-appimage-launchers" <<'EOF'
#!/usr/bin/env bash
set -euo pipefail
apps="$HOME/.local/share/applications"
install -d "$apps"
shopt -s nullglob

for f in "$HOME"/Applications/*.AppImage; do
  [[ -x "$f" ]] || continue
  base="$(basename "$f" .AppImage)"
  slug="$(printf '%s' "$base" | tr '[:upper:]' '[:lower:]' | tr -cs 'a-z0-9._-' '-')"
  case "$base" in
    *[Ll][Mm]*[Ss]tudio*) icon="applications-development" ;;
    *[Hh][Tt][Tt][Pp]*[Tt]oolkit*) icon="network-wired" ;;
    *) icon="application-x-executable" ;;
  esac
  cat > "$apps/appimage-$slug.desktop" <<DESKTOP
[Desktop Entry]
Type=Application
Version=1.0
Name=$base
Comment=AppImage application
Exec="$f"
Icon=$icon
Terminal=false
Categories=Utility;
DESKTOP
  chmod 0644 "$apps/appimage-$slug.desktop"
done

if command -v kbuildsycoca6 >/dev/null 2>&1; then
  kbuildsycoca6 >/dev/null 2>&1 || true
fi
EOF

chmod 0755 "$HOME/.local/bin/mk-appimage-launchers"
"$HOME/.local/bin/mk-appimage-launchers"
```

### 22.6 Validate and refresh

```bash
find "$HOME/.local/share/applications" -maxdepth 1 -name 'tool-*.desktop' -print0 \
  | xargs -0 -r desktop-file-validate

find "$HOME/.local/share/applications" -maxdepth 1 -name 'appimage-*.desktop' -print0 \
  | xargs -0 -r desktop-file-validate

kbuildsycoca6
```

`desktop-file-validate` must produce no errors. Warnings about an unregistered category are harmless, but the table uses standard `Utility`, `System`, `Network` and `Development` categories to avoid them.

### 22.7 Check the menu

Open the application launcher and search for **Nmap**, **Ncdu**, **Ghidra**, **Android Studio**, **Burp Suite**, **Ventoy**, **Ollama** and **Grype**. Each should launch. If an entry is missing:

```bash
ls -l "$HOME/.local/share/applications/tool-nmap.desktop"
grep '^Exec=' "$HOME/.local/share/applications/tool-nmap.desktop"
kbuildsycoca6
```

If Konsole does not open, confirm `/usr/bin/konsole` exists. If a launcher opens and exits immediately, run the same command in a terminal to see the error.

## 23. Verification Checklist

Run this from a terminal **inside the Plasma session**, not over SSH; `vainfo` and `vulkaninfo` need the graphics session.

```bash
systemctl --failed
getenforce
firewall-cmd --state
systemctl is-enabled fstrim.timer
systemctl is-enabled dnf5-automatic.timer
systemctl is-active tuned-ppd
tuned-adm active
zramctl

vainfo | grep -E 'i965|H264'
vulkaninfo --summary | grep -E 'deviceName|apiVersion'
echo "$XDG_SESSION_TYPE"

ip -br address
nmcli device status
lsblk -f
findmnt /media/DATA
swapon --show

sudo snapper -c root list
systemctl is-enabled snapper-timeline.timer snapper-cleanup.timer
flatpak list --app --columns=application | sort

rpm -q firefox thunderbird keepassxc gimp vlc obs-studio HandBrake-gui \
  google-chrome-stable brave-browser code docker-ce temurin-17-jdk
apktool --version
"$HOME/github-tools/jadx/bin/jadx" --version
ls -d /opt/ghidra_* /opt/android-studio 2>/dev/null
systemctl is-active ollama docker libvirtd vboxdrv
ollama --version
VBoxManage --version
virsh list --all
psql --version
valkey-cli ping
nuclei -version
grype version

fc-list | wc -l
lpstat -r
scanimage -L
```

Then check the launchers:

```bash
find "$HOME/.local/share/applications" -maxdepth 1 -name 'tool-*.desktop' -print0 \
  | xargs -0 -r desktop-file-validate
kbuildsycoca6

for f in tool-nmap tool-ncdu tool-ghidra tool-android-studio tool-burp \
         tool-ventoy tool-ollama tool-grype; do
  test -f "$HOME/.local/share/applications/$f.desktop" || echo "MISSING: $f"
done
```

`systemctl --failed` must be empty. `getenforce` must print `Enforcing`. `XDG_SESSION_TYPE` must print `wayland`; there is no X11 session fallback in this guide. `findmnt /media/DATA` must show `ntfs3`. `zramctl` must list `zram0`; `swapon --show` must show `/dev/zram0` and no disk-backed swap. `vainfo` must show `i965`. A missing `ollama` or `VBoxManage` means the corresponding optional section was skipped.

---

## 24. Maintenance and Troubleshooting

### 24.1 Routine maintenance

```bash
sudo snapper -c root create -d "before maintenance"
sudo dnf upgrade --refresh
sudo dnf autoremove
sudo dnf clean all
flatpak update -y
sudo snapper -c root list
sudo btrfs filesystem usage /
```

Run `sudo dnf distro-sync` only when a repository or version mismatch is actually diagnosed; it is not routine maintenance.

### 24.2 Common problems

| Symptom | Check and fix |
|---|---|
| DNF reports duplicate packages | `sudo dnf repoquery --duplicates`; remove the older duplicate only after checking what depends on it |
| Kernel module missing after an update | `sudo akmods --force`; `sudo dracut -f`; reboot |
| VirtualBox refuses to start a VM | `systemctl status vboxdrv`; `sudo modprobe vboxdrv`; `sudo akmods --force`; check `mokutil --sb-state` |
| Secure Boot rejects the VirtualBox module | `mokutil --list-enrolled`; re-enroll the akmods key with `kmodgenca -a` and `mokutil --import` |
| Snapper has no snapshots | `systemctl status snapper-timeline.timer`; `sudo snapper -c root list`; `findmnt /.snapshots`; check `SUBVOLUME="/"` in `/etc/snapper/configs/root` |
| Flatpak app does not start | `flatpak repair`; `flatpak update`; run `flatpak run <app-id>` in a terminal |
| SELinux denial | `sudo ausearch -m AVC -ts recent`; fix the label with `sudo restorecon -Rv <path>`; do not set SELinux permissive |
| Wayland display problem | `kscreen-doctor -o`; `journalctl -b --grep=kscreen`; do not switch to X11, because this guide does not install it |
| Wi-Fi drops or is slow | `rfkill list`; `nmcli device status`; `dmesg \| grep iwlwifi`; `sudo modprobe -r iwlmvm iwlwifi && sudo modprobe iwlwifi` |
| No audio | `wpctl status`; `systemctl --user restart wireplumber`; check the output device in Plasma |
| Docker permission denied | `groups`; log out and back in after `usermod -aG docker`; `sudo systemctl status docker` |
| Ollama not responding | `systemctl status ollama`; `journalctl -u ollama -b`; `curl http://127.0.0.1:11434/api/tags` |
| NTFS disk will not mount | `dmesg \| grep ntfs3`; confirm Windows is shut down; `sudo ntfsfix /dev/sdXN` only then |
| ZRAM is not active or swap is missing | `zramctl`; `swapon --show`; `systemctl status systemd-zram-setup@zram0.service`; `sudo dnf install -y zram-generator-defaults`; do not add a swap file |
| Launcher does nothing | `desktop-file-validate ~/.local/share/applications/tool-*.desktop`; `kbuildsycoca6`; run its `Exec` line in a terminal |

### 24.3 Package and repository problems

```bash
sudo dnf repolist --enabled
sudo dnf makecache --refresh
sudo dnf check
```

If a vendor repository breaks, disable it rather than using `--nogpgcheck`:

```bash
sudo dnf config-manager setopt <repo-id>.enabled=0
```

Never install an unsigned package or disable GPG checking to work around a repository error.

### 24.4 Snapshot restore

List the snapshots and choose the one to restore:

```bash
sudo snapper -c root list
```

For a full system rollback, use the script from Section 2.6. Do not run native `snapper rollback` on this layout:

```bash
sudo snapper-rollback SNAPSHOT_ID
sudo reboot
```

For a selective restore, review the difference, restore only the path you need, and reboot:

```bash
sudo snapper -c root status 1..2
sudo snapper -c root diff 1..2
sudo snapper -c root undochange 1..2 /etc
sudo reboot
```

After a rollback, verify `/`, `/.snapshots` and `/boot`, then delete the saved `@.old-*` subvolume only after the system is confirmed working. If the system cannot boot, use a Fedora 44 live image, mount the Btrfs top level with `subvolid=5`, rename the failed `@` aside and rename the saved `@.old-*` back to `@`; do not delete either copy until the rescue is verified.

### 24.5 Logs

Replace `NetworkManager` with the unit you are debugging.

```bash
journalctl -b -p err
journalctl -b -u NetworkManager
systemctl --failed
dmesg --level=err,warn
```

---

## 25. Frequently Asked Questions

**Does this laptop support CUDA or ROCm?**  
No. It has Intel HD Graphics 4600 only. CUDA requires NVIDIA and ROCm requires supported AMD GPUs. Ollama and other CPU workloads run, but without GPU acceleration.

**Should I install `intel-media-driver`?**  
No. It supports Gen 8 (Broadwell) and newer. Haswell uses the legacy `i965` driver from `libva-intel-driver`.

**Why not install Fedora's `ollama` package?**  
It depends on `hipblas`/`rocm-hip`, which pulls the AMD/ROCm stack onto an Intel-only machine. The upstream installer provides the same service without those dependencies.

**Why are `nuclei` and `grype` not installed with `go install`?**  
Fedora 44 packages both. Native packages update with DNF, carry signatures, and do not create a second copy in `~/go/bin`.

**Where do Android AVDs live?**  
`$HOME/.android/avd`. This is the current default. The old `~/.config/.android/avd` path is not used and conflicts with the copied `~/.android` directory.

**Does Fedora 44 KDE use SDDM?**  
No. A fresh Fedora 44 KDE installation uses Plasma Login Manager. Do not install or configure SDDM. This guide does not install the X11 session; use the Wayland session and treat XWayland only as a compatibility layer for unported applications.

**Should I use Snapper?**  
Yes. This guide uses Snapper with the `@`/`@home` layout in [Section 1.3](#13-partition-layout-for-snapper). The root configuration snapshots `@` only; `@home` and the four `/var` subvolumes are separate. Do not run a second snapshot manager on the same root.

**Why did `dnf install p7zip` fail?**  
`p7zip` is retired. Use `7zip`, which provides the `7z` command.

**Where is Redis?**  
Fedora 44 uses `valkey`, the Redis-compatible fork. Install `valkey`, not the retired `redis` package.

**Is hibernation supported?**  
No. This guide uses ZRAM only, and ZRAM cannot hold a hibernation image. Suspend-to-RAM works. If you need hibernation, you must add a disk swap area large enough for RAM and change the boot/resume configuration; that is outside this guide.

**Can I use the NTFS disk for Linux home directories or Ollama models?**  
No. Use NTFS only for data exchange. It does not preserve Linux ownership, permissions, symlinks or case semantics well enough for home directories, model stores or Git worktrees.

**Why does a launcher open a terminal and then a shell?**  
The `desktop-term` wrapper runs the command, then leaves an interactive shell. This makes CLI tools usable without editing the launcher every time you need a different argument.

---

## Appendix A — Package Name Changes and Equivalents

| Old name or habit | Fedora 44 |
|---|---|
| `p7zip`, `p7zip-full` | `7zip` |
| `wget` package | `wget1-wget` (provides `/usr/bin/wget`) |
| `dnf-automatic` package | `dnf5-plugin-automatic`, timer `dnf5-automatic.timer` |
| `redis` | `valkey` |
| `partitionmanager` | `kde-partitionmanager` |
| `ffmpeg-free` | `ffmpeg` from RPM Fusion |
| `libva-intel-media-driver` | Not for Haswell; use `libva-intel-driver` from RPM Fusion |
| `nvidia-*`, `cuda-*`, `rocm-*` | Not applicable; do not install |
| SDDM configuration | Plasma Login Manager on a fresh Fedora 44 KDE install |
| Snapper | Installed; root configuration only; snapshots stored in `@snapshots` |
| `auto-cpufreq`, TLP | Keep `tuned-ppd`; use the others only after measuring a problem |
| Google Chrome | Vendor RPM: `google-chrome-stable` |
| Brave | Vendor RPM: `brave-browser` |
| Visual Studio Code | Vendor RPM: `code` |
| Docker CE | Vendor RPMs: `docker-ce`, `docker-ce-cli`, `containerd.io`, plugins |
| Temurin 17 | Adoptium RPM: `temurin-17-jdk` |
| `nuclei` via `go install` | Fedora package `nuclei` |
| `grype` via install script | Fedora package `grype` |
| `ollama` RPM | Upstream installer; the RPM pulls ROCm |

---

## Appendix B — Tools Installed Outside Fedora Repositories

Some of these have native Fedora packages; the listed route is the one this guide uses. Others are not packaged at all.

| Tool | Route |
|---|---|
| `bully`, `mdk4`, `hcxdumptool` | Build from upstream source in `~/github-tools` |
| `nikto`, `sqlmap` | Upstream Git clones under `/opt` |
| `CeWL`, `wpscan` | Ruby from upstream; CeWL is a script, WPScan is a gem |
| `feroxbuster` | `cargo install feroxbuster` |
| `syft`, `trufflehog` | Official install scripts into `~/.local/bin` |
| `noseyparker` | Official GitHub release binary |
| `semgrep` | `pipx install semgrep` |
| `apktool`, `jadx`, `Ghidra`, Android Studio | Vendor/GitHub installs in [Section 17](#17-vendor-and-manual-installs) |
| Burp Suite Community | PortSwigger installer |
| ONLYOFFICE, ZAP, Postman, Zoom, Discord, Slack, Signal, Obsidian, Xournal++, ZapZap, ytDownloader, Flatseal, Network Displays | Flathub |
| `auto-cpufreq` | Not packaged; not recommended while `tuned-ppd` is active |
| NVIDIA, CUDA, ROCm, AMDGPU | Not applicable to this hardware |

---

## Appendix C — Order of Operations

1. Back up the old machine; check firmware and make the Fedora 44 KDE installer.
2. Install `@`, `@home`, `@snapshots`, `@var_cache`, `@var_log`, `@var_spool` and `@var_tmp`; separate ext4 `/boot`; no swap.
3. First boot: verify every mount, `rootflags=subvol=@`, and the default subvolume `@`.
4. Install Snapper, create the root configuration, baseline snapshot, retention and timers; optionally enable the DNF5 hook.
5. Configure DNF5, update Fedora, update firmware, set hostname/timezone, mount the NTFS disk.
6. Enable RPM Fusion, Flathub, OpenH264, Chrome, Brave, VS Code, Docker and Adoptium.
7. Install Intel graphics, multimedia and Bluetooth codecs.
8. Harden SELinux/firewalld, enable TRIM, ZRAM and automatic security updates.
9. Install shell tools, development tools and language runtimes.
10. Install the security toolkit; build the missing wireless tools.
11. Configure networking, Samba/NFS, virtualization, Docker/Podman and databases.
12. Install native and Flatpak desktop applications, fonts, printing and scanning.
13. Install Android Studio, apktool, jadx, Ghidra, Burp, Ollama and Ventoy.
14. Tune the kernel and enable only the services you use.
15. Apply KDE settings; restore data and dotfiles; build launchers; run the checklist.

---

## Sources and Verification

- Fedora 44 release notes: <https://docs.fedoraproject.org/en-US/fedora/latest/release-notes/>
- Fedora KDE Plasma Desktop 44: <https://fedoramagazine.org/whats-new-in-fedora-kde-plasma-desktop-44/>
- Fedora DNF5 documentation: <https://docs.fedoraproject.org/en-US/quick-docs/dnf/>
- Fedora 44 `plasma-workspace` package: <https://packages.fedoraproject.org/pkgs/plasma-workspace/plasma-workspace/fedora-44.html>
- Plasma 6.8 release notes (X11 retirement context): <https://kde.org/announcements/plasma/6/6.8.0/>
- RPM Fusion configuration: <https://rpmfusion.org/Configuration>
- RPM Fusion multimedia: <https://rpmfusion.org/Howto/Multimedia>
- RPM Fusion VirtualBox: <https://rpmfusion.org/Howto/VirtualBox>
- Flathub: <https://flathub.org/>
- Intel libva: <https://github.com/intel/libva>
- Intel media driver (Gen 8+): <https://github.com/intel/media-driver>
- Linux NTFS3 documentation: <https://docs.kernel.org/filesystems/ntfs3.html>
- Fedora zram-generator: <https://github.com/systemd/zram-generator>
- Fedora 44 `zram-generator-defaults` package: <https://packages.fedoraproject.org/pkgs/zram-generator/zram-generator-defaults/fedora-44.html>
- Android environment variables: <https://developer.android.com/tools/variables>
- Android Studio downloads: <https://developer.android.com/studio>
- Ghidra: <https://github.com/NationalSecurityAgency/ghidra>
- Ollama Linux install: <https://ollama.com/download/linux>
- VirtualBox manual: <https://www.virtualbox.org/manual/>
- Docker on Fedora: <https://docs.docker.com/engine/install/fedora/>
- Adoptium installation: <https://adoptium.net/installation/linux/>
- Go installation: <https://go.dev/doc/install>
- Rust installation: <https://rustup.rs/>
- nvm: <https://github.com/nvm-sh/nvm>
- Fedora Btrfs subvolumes: <https://fedoramagazine.org/working-with-btrfs-subvolumes/>
- Snapper: <https://snapper.io/>
- openSUSE Snapper concepts: <https://documentation.suse.com/smart/systems-management/html/snapper-basic-concepts/index.html>
- Btrfs documentation: <https://btrfs.readthedocs.io/>
- Fedora 44 `libdnf5-plugin-actions`: <https://packages.fedoraproject.org/pkgs/dnf5/libdnf5-plugin-actions/fedora-44-updates.html>
- Fedora firewalld: <https://docs.fedoraproject.org/en-US/quick-docs/firewalld/>
- Fedora SELinux: <https://docs.fedoraproject.org/en-US/quick-docs/selinux-getting-started/>
- Fedora CUPS/printing: <https://docs.fedoraproject.org/en-US/quick-docs/cups-useful-tricks/>

**Verification method:** package names and versions were checked against the Fedora 44 `Everything` and updates metadata and RPM Fusion 44 free/nonfree metadata. Flatpak IDs were checked against the Flathub API. Vendor versions were checked against the current official release pages. Commands that depend on a moving version use the vendor's `releases/latest` endpoint or a documented default rather than a hard-coded obsolete version.

---

## Appendix D — 100 Essential Fedora Commands

Run these as your normal user. Commands with `sudo` change the system. Replace uppercase placeholders such as `PACKAGE`, `UNIT`, `APP_ID`, `NUMBER`, `N1`, `N2`, `PID`, `NAME`, `PATH`, `URL`, `SRC`, `DEST`, `DIR`, `PATTERN` and `TEXT` with real values. Each command has its explanation directly above it and is in its own code block.

1. **Show the Fedora release**

```bash
cat /etc/fedora-release
```

2. **Show the running kernel**

```bash
uname -r
```

3. **Show hostname and machine information**

```bash
hostnamectl
```

4. **Show time, timezone and NTP state**

```bash
timedatectl
```

5. **Show keyboard, language and locale settings**

```bash
localectl status
```

6. **Show uptime and load average**

```bash
uptime
```

7. **Show RAM and swap usage**

```bash
free -h
```

8. **Show filesystem usage by mount point**

```bash
df -hT
```

9. **List block devices, filesystems and UUIDs**

```bash
lsblk -f
```

10. **Show what is mounted at root**

```bash
findmnt /
```

11. **Update all enabled DNF repositories**

```bash
sudo dnf upgrade --refresh
```

12. **Install a package**

```bash
sudo dnf install -y PACKAGE
```

13. **Remove a package**

```bash
sudo dnf remove -y PACKAGE
```

14. **Search repositories for a package**

```bash
sudo dnf search TERM
```

15. **Show package details before installing**

```bash
sudo dnf info PACKAGE
```

16. **List installed packages**

```bash
dnf list --installed
```

17. **List available updates without installing them**

```bash
sudo dnf check-upgrade
```

18. **Refresh repository metadata**

```bash
sudo dnf makecache
```

19. **Clear cached package data**

```bash
sudo dnf clean all
```

20. **Remove automatically unused dependencies**

```bash
sudo dnf autoremove
```

21. **Show DNF transaction history**

```bash
sudo dnf history
```

22. **Undo a DNF transaction by ID**

```bash
sudo dnf history undo ID
```

23. **Find which package provides a command or file**

```bash
dnf provides /usr/bin/htop
```

24. **List package groups**

```bash
dnf group list
```

25. **Reinstall a package**

```bash
sudo dnf reinstall PACKAGE
```

26. **List enabled repositories**

```bash
dnf repolist --enabled
```

27. **Show the Fedora release number**

```bash
rpm -E %fedora
```

28. **Check whether a package is installed**

```bash
rpm -q PACKAGE
```

29. **Find the package that owns a file**

```bash
rpm -qf /usr/bin/bash
```

30. **Show package information**

```bash
rpm -qi PACKAGE
```

31. **List files installed by a package**

```bash
rpm -ql PACKAGE
```

32. **Verify installed package files**

```bash
sudo rpm -Va
```

33. **Show the most recently installed packages**

```bash
rpm -qa --last | head
```

34. **Install a Flatpak application from Flathub**

```bash
flatpak install flathub APP_ID
```

35. **Uninstall a Flatpak application**

```bash
flatpak uninstall APP_ID
```

36. **Update all Flatpak applications**

```bash
flatpak update -y
```

37. **List installed Flatpak applications**

```bash
flatpak list --app
```

38. **Search Flathub**

```bash
flatpak search TERM
```

39. **Run a Flatpak application by ID**

```bash
flatpak run APP_ID
```

40. **Repair Flatpak installations**

```bash
flatpak repair
```

41. **Remove unused Flatpak runtimes**

```bash
flatpak uninstall --unused
```

42. **List Snapper snapshots**

```bash
sudo snapper -c root list
```

43. **Create a manual Snapper snapshot**

```bash
sudo snapper -c root create -d "before change"
```

44. **Delete a Snapper snapshot by number**

```bash
sudo snapper -c root delete NUMBER
```

45. **Show changed files between two snapshots**

```bash
sudo snapper -c root status N1..N2
```

46. **Show the diff between two snapshots**

```bash
sudo snapper -c root diff N1..N2
```

47. **Undo changes for a specific path**

```bash
sudo snapper -c root undochange N1..N2 /etc
```

48. **Show the active Snapper configuration**

```bash
sudo snapper -c root get-config
```

49. **Run Snapper cleanup for numbered snapshots**

```bash
sudo snapper -c root cleanup number
```

50. **Show Btrfs filesystem usage**

```bash
sudo btrfs filesystem usage /
```

51. **List Btrfs subvolumes**

```bash
sudo btrfs subvolume list /
```

52. **Show the default Btrfs subvolume**

```bash
sudo btrfs subvolume get-default /
```

53. **Start a Btrfs scrub**

```bash
sudo btrfs scrub start /
```

54. **Show Btrfs scrub status**

```bash
sudo btrfs scrub status /
```

55. **Show Btrfs device error counters**

```bash
sudo btrfs device stats /
```

56. **Trim unused blocks on SSDs**

```bash
sudo fstrim -av
```

57. **Show a systemd service status**

```bash
sudo systemctl status UNIT
```

58. **Restart a systemd service**

```bash
sudo systemctl restart UNIT
```

59. **Enable and start a service now**

```bash
sudo systemctl enable --now UNIT
```

60. **Disable and stop a service now**

```bash
sudo systemctl disable --now UNIT
```

61. **List failed systemd units**

```bash
systemctl --failed
```

62. **List all systemd timers**

```bash
systemctl list-timers --all
```

63. **Show errors from the current boot**

```bash
journalctl -b -p err
```

64. **Show logs for one service from this boot**

```bash
journalctl -u UNIT -b
```

65. **Follow the system journal live**

```bash
journalctl -f
```

66. **Show kernel errors and warnings**

```bash
dmesg --level=err,warn
```

67. **Show processes sorted by memory use**

```bash
ps aux --sort=-%mem | head
```

68. **Show processes sorted by CPU use**

```bash
ps aux --sort=-%cpu | head
```

69. **Find a process by name**

```bash
pgrep -a NAME
```

70. **Send a normal termination signal to a process**

```bash
kill PID
```

71. **Terminate processes by name**

```bash
pkill NAME
```

72. **Show ZRAM compressed swap devices**

```bash
zramctl
```

73. **Show active swap devices**

```bash
swapon --show
```

74. **Show SATA/SAS disk health**

```bash
sudo smartctl -a /dev/sda
```

75. **Show IP addresses in brief form**

```bash
ip -br address
```

76. **Show the routing table**

```bash
ip route
```

77. **Show NetworkManager device state**

```bash
nmcli device status
```

78. **Bring a NetworkManager connection up**

```bash
nmcli connection up NAME
```

79. **Test basic network connectivity**

```bash
ping -c 4 1.1.1.1
```

80. **Resolve a hostname using the system resolver**

```bash
getent hosts example.com
```

81. **Fetch HTTP response headers**

```bash
curl -I URL
```

82. **Download a file and resume if interrupted**

```bash
wget -c URL
```

83. **Show listening TCP/UDP sockets**

```bash
ss -tulpn
```

84. **Show the active firewalld rules**

```bash
sudo firewall-cmd --list-all
```

85. **Show SELinux mode**

```bash
getenforce
```

86. **Search recent SELinux denials**

```bash
sudo ausearch -m AVC -ts recent
```

87. **Restore SELinux labels recursively**

```bash
sudo restorecon -Rv PATH
```

88. **Show PipeWire and audio device status**

```bash
wpctl status
```

89. **Open the Bluetooth controller shell**

```bash
bluetoothctl
```

90. **Show and change display configuration**

```bash
kscreen-doctor -o
```

91. **Check for firmware updates**

```bash
sudo fwupdmgr get-updates
```

92. **Read motherboard and CPU sensors**

```bash
sensors
```

93. **Show the active power profile**

```bash
tuned-adm active
```

94. **List files with human-readable sizes**

```bash
ls -lah
```

95. **Find files by name below the current directory**

```bash
find . -name "PATTERN"
```

96. **Search file contents recursively**

```bash
grep -R "TEXT" .
```

97. **Create a gzip-compressed tar archive**

```bash
tar -czf archive.tar.gz DIR
```

98. **Extract a gzip-compressed tar archive**

```bash
tar -xzf archive.tar.gz
```

99. **Copy files while preserving attributes and showing progress**

```bash
rsync -a --info=progress2 SRC/ DEST/
```

100. **Edit a system file safely with sudoedit**

```bash
sudoedit /etc/fstab
```
