---
name: diagnostic-flow
description: "The five-step diagnostic flow, the clean-exit unit trap, load average versus idle CPU, the bind divergence probe and the read-only health sweep."
---

# Diagnostic flow

## The five steps

Run them in order, every time.

1. **Live check.** `systemctl is-active <unit>`, `ss -tlnp | grep :<port>`, and a request against the hostname with `--resolve` pointing at the proxy's LAN address.
2. **Last 50 lines of the log.** `journalctl -u <unit> -n 50 --no-pager`.
3. **Code and config sanity.** Read the unit file, the environment files, the proxy vhost, the catalogue entry.
4. **Hypothesis plus cheapest test.** State the hypothesis in one sentence, then run the cheapest command that distinguishes it from the next likeliest cause.
5. **Fix or escalate.** If the fix is reversible and low-risk, apply it. Otherwise present two options and pause.

Never declare something broken without showing the live evidence — the last five to ten log lines. Guessing costs the operator more time than the diagnosis and erodes trust in every later report.

## The clean-exit unit trap

`systemctl is-active` says `inactive (dead)` while `is-enabled` says `enabled`, and the journal says `Deactivated successfully` / `Main PID 1234 (code=exited, status=0/SUCCESS)`. That is a clean exit. The default restart policy only restarts on non-zero exits, so systemd reads the clean exit as success and leaves the unit parked.

Confirm it: `journalctl -u <unit> --since "2 days ago" --no-pager | grep -iE 'deactivated|status=' | tail -3` and `systemctl show <unit> -p Restart -p RemainAfterExit`.

Then decide what the unit is supposed to be. **Always-on:** widen the restart policy to `Restart=always` (or `Restart=on-success,Restart=on-failure,RestartSec=5`). **On-demand:** the dead state is the design, change nothing. Distinguish the two by reading the unit's service block and the controlling unit that stops it — a `Stopping ...` line in the journal with no command after it confirms the stop was intentional.

## High load average with an idle CPU

This is not CPU consumption. It is D-state (I/O-blocked) tasks or memory pressure with the swap exhausted.

```bash
cat /proc/loadavg; uptime
sar -q -f /var/log/sysstat/sa$(date +%d)    # run queue size, blocked
sar -u -f /var/log/sysstat/sa$(date +%d)    # iowait vs idle
sar -S -f /var/log/sysstat/sa$(date +%d)    # swap used
free -h
ps -eo stat --no-headers | grep -c '^D'     # D-state processes
```

The tell is a run queue far above the post-restart baseline with the idle percentage at 90 or higher. A single process can hold dozens of D-state threads: a browser-driven service with a memory spike and leftover worker processes drives the swap to 100 percent and the load average follows.

## Container bind-mount divergence

**Pattern:** the service is running, the ports listen, the health check passes, the logs are silent — and the content is wrong. The operator reports that a path is empty, shows old files, or never updates. Every standard container check misses this, because the symptom is divergence between two filesystems that should mirror each other, not a process failure.

```bash
# From the host
HOST_FILES=$(find <host-source-path> -maxdepth 5 -type f 2>/dev/null | wc -l)
# From inside the container
CONTAINER_FILES=$(docker exec <container> find <mount-target> -maxdepth 5 -type f 2>/dev/null | wc -l)
echo "host=$HOST_FILES container=$CONTAINER_FILES"
```

If they differ, a container restart usually re-establishes the bind. If they still differ after a restart, the mount declaration in the compose file is wrong — a path typo or a missing mount — and the container is showing its own internal directory. The fix is in the compose file, not in the restart.

**Rule of thumb:** when the status, the port, the health check and the logs are all clean and the operator reports content problems, the bind is the **first** hypothesis.

## The read-only health sweep

Run in order. Each step is under 30 seconds and the full sweep is about five minutes. Do not skip steps even when earlier ones look clean: the value is in the chain. A healthy SMART report plus a nearly full disk means "buy more storage soon", which is invisible without the disk step.

```bash
systemctl --failed --no-pager                          # 1. failed units
systemctl --failed --no-legend | wc -l                 #    (the summary line counts itself)
journalctl -p err --since '7 days ago' --no-pager | tail -30   # 2. recurring errors
#    do NOT add -b (since boot): it overrides the timestamp filter
df -h | grep -v tmpfs; df -i | grep -v tmpfs | head -10 # 3. disk space and inodes
for d in sda sdb sdc sdd nvme0n1; do                   # 4. SMART health
  sudo smartctl -H -A /dev/$d 2>/dev/null \
    | grep -iE 'SMART|Reallocated|Pending|Uncorrectable|Temperature|Power_On'
done
apt list --upgradable 2>/dev/null | wc -l             # 5. pending updates
apt list --upgradable 2>/dev/null | grep -iE 'security|cve'
apt autoremove --dry-run 2>&1 | grep -c '^Remv'       # 6. orphaned packages
docker system df                                       # 7. container disk usage
systemctl list-timers --all | grep -E 'backup|snapshot|restore|drill'   # 8. backup chain
ss -tlnp 2>/dev/null | head -25                       # 9. outbound listeners
getent passwd | awk -F: '$7 != "/usr/sbin/nologin" && $7 != "/bin/false" && $1 != "root" {print}'   # 10. login shells
```

Finding from step 3b: a filesystem that returns an I/O error on write while `mount` still shows it writable is a filesystem the kernel remounted read-only after I/O errors. Check the filesystem state, the SMART counters and the kernel log before touching service state.

## Reporting the sweep

Group the findings into three buckets and give a recommendation, not just a list. **Important** — reliability impact: failed units, SMART warnings, pending security updates, a broken backup chain, a broken notification chain. **Nice to have** — cleanup, no urgency: orphan packages, container cache, unused accounts, logrotate drift. **Architecture** — offsite backup, monitoring gaps, documentation drift.

One sentence per finding. The operator delegated the host and wants "should I worry?" answered, not a dump of evidence. **When the sweep finds nothing, say so explicitly.** Do not invent problems to fill the report. "All green" is a legitimate result.
