# system-snapshot

Local backup of the EXT4 root to a Btrfs mirror, then versioned with btrbk
snapshots. Designed for hosts where `/` is on an SSD (EXT4) and `/var/backups`
is a mounted Btrfs filesystem on a separate disk (typically an HDD). CoW makes
repeated snapshots cheap.

## How it works

1. **rsync** mirrors `/` to `/mnt/mirror/`, a Btrfs subvolume mounted on
   demand from the backup filesystem. Excludes (caches, container images,
   dev-tool servers, etc.) live in `rsync.exclude`.
2. **btrbk** snapshots the mirror to `/var/backups/local/system-snapshots/`.
   Retention: minimum 48 h, then 14 daily / 8 weekly / 12 monthly.
3. **Recovery metadata** (package list, NSS users/groups, enabled services,
   storage layout, kernel) is written to `/var/lib/system-snapshot/metadata/`
   on the live root, so step 1 sweeps it into the mirror and step 2 freezes
   it per snapshot.

The systemd timer fires at 00:00 and 12:00 (with up to 30 min of jitter). The
script only runs a real backup once `MIN_INTERVAL_DAYS` (default 30)
has elapsed since the last success — every other firing is a silent no-op
under the timer, or prints a one-line `[INFO]` when invoked manually. A failed
run leaves `last-success` stale, so the next firing (~12 h later) retries
automatically.

## Layout after install

```
/etc/system-snapshot/{btrbk.conf,rsync.exclude,system-snapshot.rc}
/etc/systemd/system/{system-snapshot.service,system-snapshot.timer}
/mnt/mirror/                                     # Btrfs subvolume, mounted on demand
/usr/local/sbin/system-snapshot
/var/backups/local/system-snapshots/             # btrbk subvolumes: system.<ts>
/var/lib/system-snapshot/{last-success,metadata/}
```

`/mnt/mirror/` is a Btrfs subvolume on the backup filesystem. The systemd
service mounts it via `/etc/fstab` only for the duration of a run and
`umount -l`s it on exit. `metadata/` lives on the live root so it travels
into the mirror with the regular rsync pass.

## Deployment

Prerequisites:
- `/var/backups` is a mounted Btrfs subvolume.
- `/var/backups/@mirror` exists as a Btrfs subvolume.
- `/etc/fstab` has a `noauto` entry for `/mnt/mirror` on the same Btrfs
  filesystem. Recommended options:
  `noatime,nodev,nosuid,noexec,nofail,x-systemd.device-timeout=30s,compress=zstd:9,subvol=@mirror`
- You are running as root.

Run from the repo root:

```bash
# Configs.
install -m 755 -d /etc/system-snapshot
install -m 644 btrbk.conf rsync.exclude system-snapshot.rc /etc/system-snapshot/

# Script.
install -m 755 system-snapshot /usr/local/sbin/system-snapshot

# systemd units.
install -m 644 system-snapshot.service system-snapshot.timer \
    /etc/systemd/system/

# State directory + snapshots dir.
install -m 700 -d /var/lib/system-snapshot
/usr/local/sbin/system-snapshot --init

# Activate the timer.
systemctl daemon-reload
systemctl enable --now system-snapshot.timer

# Optional: trigger the first real backup now (otherwise it runs at the
# next 00:00 / 12:00 firing).
/usr/local/sbin/system-snapshot --now
```

## Operating

```bash
# When does it next run?
systemctl list-timers system-snapshot.timer

# How did the last run go?
journalctl -u system-snapshot.service -n 100 --no-pager

# What snapshots exist?
ls /var/backups/local/system-snapshots/

# Run unconditionally now.
/usr/local/sbin/system-snapshot --now

# Pause (timer stays disabled until you re-enable).
systemctl disable --now system-snapshot.timer
```

## Restoring

A snapshot is a complete root mirror (minus the paths in `rsync.exclude`).

Single file:

```bash
snap=/var/backups/local/system-snapshots/system.YYYY-MM-DDTHH-MM-SS
cp --archive "$snap/etc/foo" /etc/foo
```

Full restore — boot from rescue media, mount the new root and the backup
disk, then:

```bash
rsync \
    --archive \
    --hard-links \
    --acls \
    --xattrs \
    --delete \
    --numeric-ids \
    --human-readable \
    /mnt/backup/local/system-snapshots/system.<ts>/ /mnt/restore/
```

The `metadata/` directory inside each snapshot has the package list, enabled
services, etc. captured at the time of that snapshot — useful for rebuilding
on fresh hardware before the rsync restore.

## Caveats

- **Separate `/boot` or `/boot/efi` mounts are not backed up.** rsync runs
  with `--one-file-system`, which skips anything that crosses a mount
  boundary. On UEFI dual-partition layouts, `/boot/efi` is a separate FAT
  filesystem and won't appear in the snapshot — a full restore from this
  backup alone will not produce a bootable system. If your layout has these
  as separate mounts, either back them up separately or drop
  `--one-file-system` from `run_rsync`.
- **`--checksum` is enabled.** rsync hashes every file on every run instead
  of trusting size+mtime. More reliable, slower; expect noticeably longer
  runs than a typical rsync.
- **AC power is required.** The service has `ConditionACPower=true`, so
  laptops on battery skip the run.
- **`RestrictSUIDSGID` is deliberately OFF in the service unit.** rsync must
  preserve SUID bits on `/usr/bin/sudo`, `/usr/bin/mount`, `/usr/bin/ping`,
  etc.; that flag would block the chmod that sets them. Other sandboxing
  (`MemoryDenyWriteExecute`, `NoNewPrivileges`, `ProtectControlGroups`,
  `ProtectHostname`, `ProtectKernelLogs`, `ProtectKernelTunables`,
  `RestrictNamespaces`, `RestrictRealtime`, `SystemCallArchitectures=native`,
  `LockPersonality`) is on.
- **The mirror contains `/etc/shadow`.** It's mode 700 and root-only, but
  treat the backup disk with the same trust boundary as the source disk.
- **First run is expensive.** A complete copy of `/` plus a checksum read
  pass means the initial backup takes a while. Subsequent runs only
  transfer changed blocks and only re-hash modified files.
- **Cadence is enforced by `last-success` mtime.** Don't `touch` that file
  manually; pass `--now` to override the cadence check.
- **`/mnt/mirror/` is only mounted while a backup is running.** The
  script `mount`s it via fstab when it begins a real run and `umount -l`s
  it on exit (via a bash `EXIT` trap), so manual runs (`system-snapshot
  --now`) get the same mount/umount lifecycle as the timer. To browse
  the mirror outside a run, look at the latest snapshot under
  `/var/backups/local/system-snapshots/` instead.
- **The source root is live during the backup.** rsync reads files over
  several minutes and the kernel keeps writing to `/` the whole time. A
  file modified mid-pass can land in the mirror with mixed-version content,
  and a file deleted mid-pass triggers exit 24 (see below). For files that
  need point-in-time consistency — databases, VM disk images, large
  in-progress writes — pause the writer or take a filesystem-level snapshot
  of `/` before backing up. This script does not do either; it relies on
  the rsync caveats and your workload tolerating them.
- **rsync exit 24 is treated as success.** Vanished files (something deleted
  while rsync was reading the source list) are routine on a live root and
  shouldn't trip retry.
