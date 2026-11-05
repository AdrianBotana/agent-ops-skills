---
name: incident-triage
description: "Use when a host, container or service is failing, after an outage or unexpected reboot, or when many applications break at once: ordered diagnosis, disk and hardware triage, and the post-outage checklist."
version: 0.1.0
author: Adrián Botana
license: MIT
platforms: [linux]
---

# Incident Triage

A diagnostic runbook. It covers the three shapes an incident takes on a
Linux host: one service failing, many services failing at once (which is
almost never the services), and the state of the host after an outage or an
unexpected reboot.

The discipline is the same in all three: measure before you change, change one
thing, measure again, and stop when the symptom is gone.

## When to Use

- The operator reports a service down, a 502, or a container that will not
  stay up.
- Several unrelated applications fail at the same time.
- The host was power-cut, rebooted unexpectedly, or came back wrong.
- Load average is high but the CPU is idle.
- A filesystem returns I/O errors, or a disk is suspected.
- You need a read-only health sweep before touching anything.

Do not use for: an application bug with no infrastructure surface, or a
routine deploy.

## The Rules

### The diagnostic order

**1. Follow the five-step flow before touching anything.** Live check, then
the last 50 lines of the relevant log, then the code and config sanity check,
then a one-sentence hypothesis plus the cheapest test that distinguishes it
from the next likeliest cause, then fix or escalate. Skipping to the fix
because the symptom is obvious is how a second incident starts.

**2. Never declare something broken without live evidence.** Show the last
five to ten lines of the log. Guessing costs the operator more time than the
diagnosis and erodes trust in every later report.

**3. A unit that is `enabled` but `inactive` has usually exited cleanly, not
crashed.** The journal says `Deactivated successfully` and
`Main PID ... (code=exited, status=0/SUCCESS)`. The default restart policy
only restarts on non-zero exits, so a clean exit reads as success and the
unit stays parked. Check whether a controlling unit stopped it on purpose —
the journal around the stop shows a `Stopping ...` line with no command after
it, which confirms intent. If the unit is meant to run continuously, widen
the restart policy; if it is on-demand, the dead state is the design. Read the
unit and its controller before changing anything.

**4. High load average with an idle CPU is not CPU.** It is I/O wait
(D-state tasks) or memory pressure with the swap full. Measure the run
queue, the I/O wait percentage, the swap usage and the number of D-state
processes before blaming the CPU.

**5. Many applications failing with I/O errors is the disk, not the
applications.** A filesystem whose journal was aborted returns an I/O error
on every `open()` in it. Directory listings keep working because they are
served from cache — **never trust `ls` as a health test**.

**6. A filesystem that is `clean with errors` and needs recovery had its
journal aborted.** A plain reboot makes it writable again because the kernel
replays the journal, but the residual bitmap checksums stay wrong.

**7. Test the filesystem with the no-fix mode first.** Run the check in
read-only report mode on the mounted filesystem. It skips journal recovery
and reports only. A clean pass with a benign prompt about an orphan inode
means the structure is fine and the problem is the kernel-side read-only
remount.

**8. The `mount` output saying `rw` is not proof that writes work.** The
kernel can implicitly remount read-only after I/O errors without the mount
table showing it. Remounting read-write will not bypass it: the command
exits 0 and the next write still fails. Trust a write test, not a flag.

**9. A deep write test with a hash comparison is the only reliable
verification.** Write a block of random data, hash the file, read it back and
hash it again. A `mkdir` that succeeds after recovery can pass on a flaky
disk; the double hash catches silent corruption.

**10. A container that is running, listening, healthy and silent in the logs
can still be serving the wrong content.** The symptom is divergence between
two filesystems that should mirror each other. Compare a file count on the
host against a file count inside the container. If they differ, a container
restart usually re-establishes the bind; if they still differ after a
restart, the mount declaration is wrong and the container is showing its own
internal directory. When the status, port, health and logs are all clean and
the operator reports content problems, the bind is the **first** hypothesis,
not the last.

### Disk and hardware

**11. A large SATA CRC counter does not prove a bad cable.** Measure whether
the counter grows: read it, wait about 90 seconds, read it again. A frozen
counter is a firmware artefact, not signal noise. Also check whether the same
disk already carried that value under a different device name — the serial
number is the identity, the device name is not, and device names get
renumbered across reboots.

**12. A timeout with a successful driver byte is a controller hang, not
signal noise.** `hostbyte=DID_TIME_OUT driverbyte=DRIVER_OK` with a command
age of tens of seconds means the command was sent and the disk did not answer
for half a minute. The transport worked. A bad cable almost never retains a
command that long; it fails earlier with CRC errors or link renegotiation. No
cable swap fixes this.

**13. Some SSD controller families hang when link power management is
active.** The host negotiates a low-power link state, the drive hangs, and
the kernel aborts the filesystem journal. Check the link power management
policy of every SCSI host; the risky value is the device-initiated one, and
the mitigation is to force maximum performance. Apply it with a udev rule so
it survives reboots, then re-trigger the SCSI subsystem and read the policy
back. Do it even if you migrate the disk — it is free and reversible.

**14. The decision order for a suspect disk is: migrate, then disable link
power management, then cable, then replace.** If the filesystem can be saved
and moved to another disk, that resolves the root cause and makes an offline
repair unnecessary — a fresh filesystem has no errors to repair. Disabling
link power management is free, reversible and attacks the upstream cause.
Blame the cable only when the CRC counter is **growing** and several disks on
the same port show it. Replace the drive when the SMART data is clean but the
controller hangs: the flash is fine and the firmware is not, and no
filesystem repair will touch it.

**15. Never write to the raw block device, even "to test the disk".** A
write to `/dev/<device>` destroys the filesystem metadata — the primary
superblock, the group descriptor table, the early inodes — and leaves the
filesystem unmountable on the next boot. It is tempting because the direct
write to a *mounted* device fails with "device busy", and the natural
workaround is to unmount first, which is exactly the raw device. Write a file
inside the mounted filesystem instead.

**16. The damage from a raw write is bounded by the offset, and the backup
superblocks usually survive.** A filesystem rebuild or an offline repair is
usually still possible. But a mounted filesystem keeps serving data from the
kernel's in-memory copy even when its on-disk metadata is destroyed, which
gives false confidence: it "works" until the next reboot. Never declare a
filesystem healthy because reads and writes succeed today.

**17. `dumpe2fs` reading without error is not an integrity proof.** It uses
the kernel's in-memory copy, while the repair tools fail on checksum
mismatch. Use the read-only check mode, not the dump, as the integrity
signal.

**18. A read-only throughput test measures the link without writing
anything.** Read a block of the device into `/dev/null` with direct I/O. If it
reads at full speed with no errors, the disk is not degraded.

**19. Do not force a filesystem repair on a filesystem that already writes
without I/O errors.** The residual bitmap errors are counters, not
corruption. Forcing the repair over hundreds of gigabytes of production data
carries more risk than it removes, and it requires stopping everything on
that filesystem.

### Outage and reboot

**20. A power cut and a server shutdown are different events.** The UPS can
keep the server running for a while after the mains power fails. Read the
UPS's own log to tell them apart, and list the boots to get the exact
window:

```bash
journalctl --list-boots --no-pager
date -Is; cat /proc/uptime
```

**21. A clean shutdown leaves a distinct trace.** A clean shutdown reaches
the shutdown target. An abrupt cut leaves a restart record and truncated
journal entries. That distinction decides whether you look for a power
problem or a crash.

**22. A clean reboot can leave services in a crash loop because their config
references something that no longer exists.** The classic case: a network
interface was removed or migrated, the service config still binds the old
address, `bind()` fails with "address not available", and the unit
restart-loops. Find it by comparing what the services listen on against what
the config says, and grep the config directories for the retired address.

**23. A crash loop in a low-level service cascades into everything above
it.** A broker that will not start takes down every client that needs it. When
several services fail at once, find the one that failed first — the timeline
is the diagnosis.

**24. Do not report comments as configuration.** Config that documents a
migration contains the old value in a comment. Filter out the commented lines
before concluding that a retired address is still configured.

**25. Run the post-reboot sweep before forming any hypothesis.** The
operator expects the host to look the same after a reboot as before it. The
sweep takes under a minute and produces the complete triage list. It has six
steps, in this order:

1. **Boot baseline** — `journalctl --since "$(uptime -s)" --until "$(uptime -s) +1min" -p warning`.
2. **Units that failed or restart-looped at boot** — `systemctl list-units --type=service --state=failed`.
3. **Containers that restarted after coming up** — `docker ps --format '{{.Names}}\t{{.Status}}\t{{.RestartCount}}'`.
4. **Units enabled but not running** — diff the enabled unit files against the running services. This is the silent catch.
5. **Mounts** — `mount | grep -E 'mnt/(ssd|data|cold)'`.
6. **Recent alerts** — the notification log shows what the watchdog caught.

The exact commands are in `references/post-outage-checklist.md`.

**26. The common boot-failure modes, in order of frequency.** A service
pinned to a runtime version that no longer exists; a mount namespace
directory that does not exist yet at first boot; a bind mount pointing at a
file that was never created; a mount listed as `nofail` in the filesystem
table that a service's ordering depends on; a watchdog script with a syntax
error introduced by a recent edit. The first three are fixed by creating the
missing path or bumping the pin; the fourth by removing the ordering
dependency; the fifth by a syntax check and an isolated run.

**27. Fixing items one at a time while others are silently broken is wasted
work.** The sweep output is the triage list. Work through it.

### Filesystem incidents

**28. Restoring a damaged filesystem has a layer order, from most recent to
oldest.** The rescue copy holds the critical configuration dumped from the
damaged filesystem; the cold mirror holds the full project tree as of the
last daily backup; the salvage copy holds the live tree at the moment of the
incident. The salvage copy is frequently the **only** copy of large
directories that the cold mirror never covered — before declaring anything
lost, check the mirror, and if it is not there, copy the live tree to a spare
disk first.

**29. Restore order: clean base, then recent overlay, then repair from the
base.** A recent copy taken after the incident carries the files the failure
zeroed, and files that were being written when the copy ran come out
truncated. Serving the recent copy as-is propagates the corruption.

**30. Detect the corrupt files against an independent source.** Walk the tree
and flag files whose first blocks are all zero, then confirm each against the
cold mirror. A legitimate binary can start with zeros and be perfectly fine.

**31. A permission error on a root-only file means retry as root**, not
leave a hole. A directory read error with a "bad message" errno means a
corrupt inode in that directory: replace it from the mirror.

**32. Count the recovered volume per directory, not only in total**, and
check the free space in the destination before rebuilding — an offline repair
needs room for its own temporary data.

### Recovery discipline

**33. Stopping the containers on the data volume stops the network's DNS.**
If the DNS resolver runs as a container on that volume, any operation that
stops containers takes DNS down for the host and for every device that uses
it. Identify the first-order dependencies explicitly and start them in the
restoration phase, not the stopping phase.

**34. A restoration trap that only logs is worse than no trap.** It looks like
a safety net while the network stays without DNS. The trap has to actually
start the containers and services again, and it has to re-enable the
auto-repair timers — otherwise the services that were auto-repaired stay
down and the same failure returns days later with nobody touching it.

**35. `umount` reporting "target is busy" after killing every process with an
open descriptor means the mount is stacked.** Two mounts of the same device
on the same point, usually from a lazy unmount followed by a mount, or from a
mount-all over a filesystem table with a duplicate entry. Read the mount
table and look at the parent column: the second mount lists the first as its
parent. Unmount layer by layer in reverse order — a plain unmount only
removes the top layer. This also explains a repair tool that refused to run
with "device is in use": the cause was the stacked mount, not the
filesystem.

**36. A mounted filesystem with memory-mapped files can refuse to unmount.**
There may be no process with its working directory there, but a mapped file
still holds it. Kill the processes with open descriptors under the mount
point before retrying.

**37. Verify the recovery by exercising the actual service, not by checking
that the port is open.** A broker that accepts a connection but never
delivers a message is still broken. Subscribe and wait for real traffic. For
containers, require the `healthy` status, not merely `Up`.

**38. Log files inside a container rarely rotate with the host's logrotate
configuration.** The packaged logrotate rules point at host paths while the
container writes into a bind mount, so the file grows forever. The correct
rule lives in the host's logrotate directory and needs `copytruncate` — the
process keeps the descriptor open, and without `copytruncate` the rename
leaves it writing to the old inode.

**39. Stop when the symptom is gone.** Once the reported failure is resolved
and everything is healthy, stop. A pending offline repair on a filesystem
that already writes without errors is debt, not an emergency, and every
minute of downtime on a working system costs more than the residual error.
Verify the final state once, report what was broken, what was fixed, and what
remains **as a list** — and ask whether the operator wants the remainder done
now. Distinguish "the operator asked for the disk to be pristine" from "the
operator asked for it to stop crashing". If they did not say the first one
literally, it is the second.

**40. Numeric comparisons in shell scripts break on non-numeric input.** A
sensor that reports "-" makes an integer comparison fail with a syntax error
and abort the whole alerting line. Guard every comparison with an
is-numeric check first.

## Reference Files

- `references/diagnostic-flow.md` — the five-step flow, the clean-exit unit
  trap, load average versus idle CPU, the bind divergence probe and the
  health sweep.
- `references/disk-and-hardware-failures.md` — the SATA link decision
  procedure, the link power management mitigation, the raw-device write ban
  and the filesystem recovery order.
- `references/post-outage-checklist.md` — the post-reboot sweep, the boot
  failure catalogue, the filesystem restore layers and the verification
  commands.

Load with `skill_view(name='incident-triage', file_path='references/diagnostic-flow.md')`.
Load with `skill_view(name='incident-triage', file_path='references/disk-and-hardware-failures.md')`.
Load with `skill_view(name='incident-triage', file_path='references/post-outage-checklist.md')`.
