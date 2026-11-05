---
name: disk-and-hardware-failures
description: "SATA link failure triage, the CRC measurement that does not mislead, link power management mitigation, the raw-device write ban and the filesystem recovery order."
---

# Disk and hardware failures

The goal is to decide **which component to change** — cable, port, controller or the whole drive — with evidence, instead of swapping hardware at random.

## "High CRC means bad cable" does not work

A SATA CRC counter in the hundreds of thousands is not proof of a bad cable. Before blaming the cable, rule out that the counter is real:

```bash
# 1. Does the counter GROW? Measure it, do not assume it.
smartctl -A /dev/<device> | grep SATA_CRC_Error_Count   # value T1
sleep 90
smartctl -A /dev/<device> | grep SATA_CRC_Error_Count   # value T2
# T1 == T2 -> frozen counter -> firmware artefact, not a cable.

# 2. Is this the same physical disk renumbered?
smartctl -i /dev/<device> | grep -E 'Serial Number|Device Model'
# Device names change across boots (sda <-> sde <-> sdb). The serial does not.
# If the same value shows up under another name, the CRC is inherent to that disk.

# 3. Does the CRC reappear on other disks behind the same controller?
for d in sda sdb sdc sdd; do
  printf '%s ' "$d"; smartctl -A /dev/$d 2>/dev/null | grep SATA_CRC_Error_Count
done
# One disk with a huge counter and the rest at 0-1 -> that disk is the problem.
```

A genuinely bad cable produces a **growing** CRC count and link renegotiations. Bad controller firmware produces a frozen counter and abrupt hangs.

## The `DID_TIME_OUT` signature

```bash
journalctl -k --since '24 hours ago' | grep -E 'DID_TIME_OUT|Aborting journal'
```

`hostbyte=DID_TIME_OUT driverbyte=DRIVER_OK` with a command age of 30-40 seconds means the command was transmitted correctly and the disk did not answer for half a minute. The transport worked. That is not signal noise — noise produces CRC and uncorrectable errors. It is the controller or the link hanging. A defective cable rarely retains a command that long; it fails earlier with CRC errors or renegotiation. A high command age rules out the cable: no cable swap fixes this.

## Controller families that hang on link power management

Some SSD controllers (the Phison S11 family, used across several consumer brands) have a characteristic failure: the host negotiates link power management, the drive hangs, and the kernel aborts the filesystem journal.

```bash
# Device-initiated power management is the risky one
cat /sys/class/scsi_host/host*/link_power_management_policy
# 'med_power_with_dipm' = device-initiated = risky
# 'max_performance'    = ALPM effectively off = the upstream mitigation

journalctl -u fstrim --since '2 days ago' --no-pager   # a large TRIM run can precede the hang
smartctl -A /dev/<device> | grep -E 'Unsafe_Shutdown_Count|Power_On_Hours'   # dirty shutdowns degrade firmware
```

## The upstream mitigation

Disable ALPM on the affected host. This is the documented fix for that controller family, not a cosmetic patch: it removes the power-state transition that breaks the handshake. Persist it with a udev rule so it survives reboots:

```
# /etc/udev/rules.d/99-sata-link-power-management.rules
ACTION=="add|change", KERNEL=="host[0-9]", ATTR{link_power_management_policy}=="*", ATTR{link_power_management_policy}="max_performance"
```

```bash
sudo udevadm control --reload-rules && sudo udevadm trigger --subsystem-match=scsi
cat /sys/class/scsi_host/host0/link_power_management_policy   # must read max_performance
```

## Decision order: what to change first

1. **Can the filesystem be saved and migrated?** If yes, migrating to another disk resolves the root cause and makes an offline repair unnecessary — a new filesystem is clean, with zero errors to fix. Lowest risk.
2. **Disable ALPM** on the affected host. Free, reversible, no downtime, and it attacks the upstream cause. Do it even if you migrate.
3. **The cable**, only when the CRC counter is **growing** and several disks on the same port show it. Not the first suspect when the SMART data is clean and the failure is a timeout.
4. **The whole drive**, when the SMART data is clean but the controller hangs. The flash is fine and the firmware is not. No filesystem repair fixes it; only replacing the drive or migrating does.

## Never write to the raw block device

**Every write test goes to a file inside the mounted filesystem, never to `/dev/<device>`.** A write to the raw device destroys the filesystem metadata — the primary superblock, the group descriptor table, the early inodes — and leaves the filesystem unmountable on the next boot.

It is easy to fall into: when you want to test whether the disk survives writes, the direct write looks like the honest test. But a direct write to a *mounted* device fails with "device busy", so the natural workaround is to take the filesystem out of the way — and that is the raw device.

```bash
# WRONG - destroys the filesystem header
dd if=/dev/zero of=/dev/<device> bs=4M count=3072 oflag=direct

# RIGHT - writes inside the filesystem, leaves the metadata alone
dd if=/dev/zero of=<mountpoint>/.writetest bs=4M count=3072 oflag=direct && sync && rm -f <mountpoint>/.writetest
```

Corollaries:

- **The damage is bounded by the offset.** What is destroyed is the primary superblock, the group descriptor table and the inodes in that range. The backup superblocks (ext4 spreads them across the disk with `sparse_super`) and the metadata image survive, so the filesystem is usually rebuildable with an offline repair or a clean reformat.
- **A mounted filesystem keeps serving data while its on-disk metadata is destroyed**, because the kernel holds a copy in memory. That gives false confidence: it "works" until the next reboot. Never declare a filesystem healthy because reads and writes succeed today.
- **`dumpe2fs` may still read it** (it uses the kernel's in-memory copy) while the repair tools fail with an inode bitmap checksum mismatch. Do not use `dumpe2fs` as an integrity proof.
- **A hardware test with no write risk**: read the device into `/dev/null` with direct I/O to measure the real link throughput. If it reads at full speed without errors, the disk is not degraded.

## Migrating the data volume

Before migrating, confirm the **cold backup covers everything**: `comm -23 <(ls <live-projects> | sort) <(ls <cold-projects> | sort)` — empty output means every live project exists in the cold mirror.

Then look for large directories with no backup — those are what break a migration: loop over the live mount's directories, measure each with `du -sm`, and report the ones with no counterpart in the cold mount.

Verify integrity with hashes before migrating: `md5sum <live-projects>/<config> <cold-projects>/<config>`. Caveat: application state files legitimately differ — a running application writes new state after the incident. That is not data loss. Verify those with content (parseable) and timestamps rather than with the hash.

## What not to do

- Do not force an offline repair on a filesystem that already writes without I/O errors. The residual bitmap errors are counters, not corruption. Forcing it over hundreds of gigabytes of production data carries more risk than it removes, and it requires stopping everything on that filesystem.
- Do not diagnose "bad cable" from a high CRC number without measuring whether it grows.
