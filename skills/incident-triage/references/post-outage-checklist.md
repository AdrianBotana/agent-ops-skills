---
name: post-outage-checklist
description: "Post-outage and post-reboot procedure: the 60-second sweep, boot failure catalogue, filesystem restore layers, stacked mounts and the closing protocol."
---

# Post-outage checklist

Run this after a power cut, an unexpected reboot, or a return from a scheduled outage. The operator expects the host to look the same after a reboot as it did before. When it does not, this checklist produces the complete triage list in under a minute.

## 1. Establish the outage window

`journalctl --list-boots --no-pager` gives the previous boot's last entry against this boot's first; `date -Is; cat /proc/uptime` gives now and uptime.

A power cut and a server shutdown are different events: the UPS can keep the server running for a while after the mains power fails. Tell them apart with the UPS's own log: `journalctl --no-pager -o short-iso | grep -iE "on battery|on line power"`. A clean shutdown reaches the shutdown target; an abrupt cut leaves a restart record and truncated journal entries.

## 2. The 60-second sweep

Six steps, in this order:

```bash
journalctl --since "$(uptime -s)" --until "$(uptime -s) +1min" -p warning --no-pager | head -30
systemctl list-units --type=service --state=failed --no-pager
journalctl --since "$(uptime -s)" -p err --no-pager | grep -iE 'Failed to start|cannot|missing' | head
docker ps --format '{{.Names}}\t{{.Status}}\t{{.RestartCount}}' | awk '$3>0'
diff <(systemctl list-unit-files --state=enabled --no-pager | awk '{print $1}' | sort) \
     <(systemctl list-units --type=service --state=running --no-pager | awk '{print $1}' | sort) | head
mount | grep -E 'mnt/(ssd|data|cold)' || echo "MISSING MOUNTS"
sqlite3 <notify-db> "SELECT datetime(ts),title,source FROM notifications \
  WHERE ts > datetime('now', '-2 hours') ORDER BY ts DESC LIMIT 10"
```

In order: boot baseline; units that failed or restart-looped at boot; containers that restarted after coming up; units enabled but not running (the silent catch); the mounts; and the alerts that show what the watchdog caught.

Steps 2, 3 and 4 produce the triage list. Each item is a candidate root cause. Do not start fixing individual items while others are silently broken behind them.

## 3. Boot failure catalogue

In order of frequency: **runtime version drift** (a service pinned to an interpreter version that no longer exists — bump the pin or retire the unit); **missing mount namespace directory** (a service with a home or mount isolation directive pointing at a path that does not exist on first boot — create it in a pre-start hook or drop the directive); **bind mount pointing at a missing file** (a proxy sub-unit pointing at a config fragment that was never created — create the file or retire the unit); **filesystem table ordering against a service dependency** (a mount marked `nofail` boots even when missing, but a service that declares an ordering on it races — drop the ordering dependency or order on the specific mount unit); **a watchdog script with a shell syntax error** (a recent edit broke it and the service fails silently — `bash -n <script>` and run it in isolation).

## 4. Crash loops from stale configuration

A clean reboot can leave services in a restart loop because their config references something that no longer exists — a removed network interface, a migrated tunnel, a decommissioned address.

```bash
docker ps -a --format '{{.Names}}\t{{.Status}}' | grep -v Up   # restart loops
ss -tln                                                    # key ports
grep -rn "<retired-address>" /etc/<service>/conf.d/ /etc/systemd/system/ | grep -v '^\s*#'
```

**Do not report comments as configuration.** Config that documents a migration contains the old value in a comment. Filter the commented lines out before concluding that a retired address is still configured. **Find the first failure, not the loudest.** A broker that will not start takes down every client that depends on it, and the clients produce more log lines than the broker did.

## 5. Load after a reboot

`cat /proc/loadavg; uptime` for the load, `sar -q -f /var/log/sysstat/sa$(date +%d)` for the run queue, `sar -u` for iowait versus idle, `sar -S` for swap used, `free -h` for memory, and `ps -eo stat --no-headers | grep -c '^D'` for the D-state count.

High load with 90 percent idle CPU is D-state and memory pressure, not CPU. A run queue far above the post-restart baseline with the swap at 100 percent points at a service that reserved a large amount of memory and left worker processes behind.

## 6. Filesystem incidents

**Symptoms.** Many unrelated applications fail at once with an I/O error on open. It is not a bug in each application — it is the disk.

```bash
journalctl -k --since '24 hours ago' | grep -E 'DID_TIME_OUT|Aborting journal|EXT4-fs error'
tune2fs -l /dev/<partition> | grep -E 'Filesystem state|needs_recovery'
```

A timeout plus an aborted journal means the hardware hung the writes, the filesystem aborted its journal, and from that point **every** open in that filesystem returns an I/O error. Directory listings keep working because they are served from cache — do not trust `ls` as a health test. `clean with errors` plus `needs_recovery` means the filesystem was aborted: a plain reboot makes it writable again because the kernel replays the journal, but the residual bitmap checksums stay wrong.

**Recovering data without a healthy filesystem:** `debugfs -c -R 'dump <path> <destination>' <device>`. The `-c` flag is mandatory — without it the command fails with an inode bitmap checksum mismatch. The recovered path nests the original directory structure under the destination.

**Restore layers, most recent to oldest.**

| Layer | What it holds |
|---|---|
| Rescue copy | Critical configuration and state dumped from the damaged filesystem |
| Cold mirror | The full project tree as of the last daily mirror |
| Salvage copy | The live tree at the moment of the incident |

**The salvage copy is often the only copy of large directories the cold mirror never covered.** Before declaring anything lost, check the mirror; if it is not there, copy the live tree to a spare disk first.

Restore order: base from the clean pre-damage copy, overlay the most recent copy, then repair from the base any file the recent copy has corrupted. A recent copy taken after the incident carries the files the failure zeroed, and files that were being written when the copy ran come out truncated. Serving the recent copy as-is propagates the corruption.

Detect the corrupt files against an independent source: walk the tree and flag files whose first blocks are all zero, then confirm each against the cold mirror. A legitimate binary can start with zeros and be fine. A permission error on a root-only file means retry as root, not leave a hole. A directory read error with a "bad message" errno means a corrupt inode in that directory: replace it from the mirror. Count the recovered volume per directory, not only in total, and check the free space in the destination before rebuilding — an offline repair needs room for its own temporary data.

## 7. Operations that stop the data volume

**Stopping the containers on the data volume stops the network's DNS.** If the DNS resolver runs as a container there, any operation that stops containers takes DNS down for the host and for every device that uses it.

Mandatory mitigations in any repair script: list the services that resolve DNS or that the house depends on, and start them in the **restoration** phase, not the stopping phase; install a `trap restore EXIT INT TERM` that **actually starts them again** (a trap that only logs a warning is worse than no trap — it looks like a safety net while the network stays without DNS); have the trap re-enable the auto-repair timers as well, or the services that were auto-repaired do not come back and the same failure returns days later with nobody touching it; verify the state after **each** phase and never continue blind; and remember that health checks that read a directory always fail and trigger a false restoration.

## 8. Stacked mounts

`umount` reporting "target is busy" after every process with an open descriptor has been killed means the mount is stacked: two mounts of the same device on the same point, usually from a lazy unmount followed by a mount, or from a mount-all over a filesystem table with a duplicate entry.

```bash
grep ' <mountpoint> ' /proc/1/mountinfo
# 115 31  8:1 / /mnt/data rw,relatime shared:201 - ext4 /dev/sda1 rw
# 279 115 8:1 / /mnt/data rw,relatime shared:389 - ext4 /dev/sda1 rw
#    ^^^ the second lists the first as its parent -> stacked
```

Unmount **layer by layer in reverse order** — the top layer first. A plain unmount only removes one layer. A layer propagated to other namespaces (`shared:NNN`) may need a lazy unmount as the last resort. This also explains a repair tool that refused to run with "device is in use": the cause was the stacked mount, not the filesystem. A mount can also refuse to unmount because of memory-mapped files — no process has its working directory there, but a mapped file still holds it. Kill the processes with open descriptors under the mount point before retrying.

## 9. Verification and log growth

- **The broker must deliver a message.** A connection that succeeds is not proof. Subscribe and wait for real traffic.
- **Containers must be `healthy`**, not merely `Up`.
- **Ports must be listening** — check with `ss -tln`, not only the container status.
- **Log files inside a container rarely rotate with the host's logrotate configuration.** The packaged rules point at host paths while the container writes into a bind mount of the project, so the file grows forever. Check `ls -la <project>/log/*.log` and `grep -n "log_dest" <project>/compose.yml`. The correct rule lives in the host's logrotate directory and needs `copytruncate` — the process keeps the descriptor open, and without `copytruncate` the rename leaves it writing to the old inode.

## 10. Closing the incident

1. Verify the final state **once**: containers healthy, ports listening, the application that failed working.
2. Report it: what was broken, what was fixed, and what remains **as a list**, not as a task.
3. If debt remains — a pending offline repair, a disk migration, a configuration change — **ask** whether the operator wants it now. Do not execute it on your own.

Distinguish "the operator asked for the disk to be pristine" from "the operator asked for it to stop crashing". If they did not say the first one literally, it is the second. A pending repair on a filesystem that already writes without I/O errors is debt, not an urgency: it costs downtime on a system that works, and the residual error it would remove causes no I/O errors.
