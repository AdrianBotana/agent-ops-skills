---
name: health-check-layers
description: "The three check layers (process, wire, content), dead versus degraded, the on-demand exception, threshold drift absorption and the stale-map diff."
---

# Health check layers

## The three layers

A watchdog that checks only the process layer misses the failures users
actually notice. Every critical service gets all three.

**1. Process layer.** Is the thing running?

```bash
systemctl is-active <unit>
docker inspect --format '{{.State.Status}}' <container>
```

Catches: the process died, the container crashed, the unit failed. This
layer sees exactly what `systemctl --failed` sees.

**2. Wire layer.** Is it listening where it should be?

```bash
ss -tlnH sport = :<port>
curl --resolve <host>:443:<address> -sI https://<host>/
```

Catches: the process is up but the port is unbound, the port is bound to the
wrong interface, the backend behind the proxy is unreachable. This is the
common failure after a half-finished restart.

**3. Content layer.** Does a sample operation against the data path return
the expected result?

```bash
# file-serving service: compare the two sides of a bind
host=$(find <path> -maxdepth 5 -type f 2>/dev/null | wc -l)
inside=$(docker exec <container> find /<mount-target> -maxdepth 5 -type f 2>/dev/null | wc -l)
[ "$host" = "$inside" ] || alert

# HTTP service: HEAD and assert a 2xx
code=$(curl --resolve <host>:443:<address> -s -o /dev/null -w '%{http_code}' https://<host>/)
case "$code" in 2*) ;; *) alert ;; esac

# state store: query a known-good row
```

Catches: everything green while the user sees stale, empty or wrong content.
Bind divergence, a stale cache, a silently broken backend route. This is the
layer most often missing from home-grown watchdogs.

## Dead versus degraded

| Symptom | Layer that sees it | Verdict |
|---|---|---|
| Unit failed, container exited | 1 | Dead. Repair candidate if the repair is reversible and bounded. |
| Running, port unbound or bound to loopback | 2 | Degraded. The process is not the service. |
| Running, port listening, content wrong | 3 | Degraded. Restart may fix it; a config fix may be needed. |
| Enabled but inactive for hours | — | Usually a restart-policy gap, not a crash. Read the journal before acting. |

A restart often re-establishes a diverged bind. If the divergence survives
the restart, the mount declaration is wrong and the container is showing its
own internal directory — the fix is in the compose file, not in another
restart.

## The on-demand exception

A watched port that nobody listens on is not automatically a failure. Some
services start on demand and have no listener until someone asks for one.
Check that the unit exists before raising it:

```bash
systemctl list-unit-files | grep '<unit>'
```

A missing unit is a catalogue error, not an outage. Fix the map.

## Threshold drift

Some hardware returns non-deterministic readings: two reads of the same idle
state differ by a unit or two. A watcher that treats the baseline as exact
alerts on every run.

Make the threshold an environment override, not a constant, and take the
absolute value:

```bash
delta=${delta#-}
if (( abs_delta < THRESHOLD )); then
    update_baseline
    continue
fi
```

Update the baseline and continue when under the threshold. The normal
variation of the device must never wake an alert.

## The stale-map diff

A port map that nobody maintains goes stale in silence: the new port exists,
the old one does not, and the watcher alerts every cycle trying to raise a
unit that no longer exists.

```bash
ss -tlnH 2>/dev/null | awk '{print $4}' | sed 's/.*://' | sort -u > /tmp/listening.txt
grep -oE '^\s*\[[0-9]+\]=' <watcher> | tr -d ' []=' | sort -u > /tmp/watched.txt
comm -23 /tmp/watched.txt /tmp/listening.txt
```

The fix is not to delete the entry. It is to re-point it at the service that
absorbed the retired one, in the correct scope — a system-scope unit must
not be started through the user-scope bus, which fails every time. Also clear
the stale entry from the failure cache, or the next run reports it as
recovered and the log becomes misleading.

Verify by starting the watcher and confirming a clean summary line with no
alert for the retired port.
