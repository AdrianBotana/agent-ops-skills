---
name: watchdog-operations
description: "Use when an agent writes, reviews or debugs a watchdog, health-check timer, retry loop or self-healing script that restarts services, and must decide when to retry and when to escalate."
version: 0.1.0
author: Adrián Botana
license: MIT
platforms: [linux]
---

# Watchdog Operations

A watchdog is an unattended agent with write access to production. Every
dangerous thing about it — an unbounded retry, a destructive default, a
silent notification failure — is a decision somebody made once and never
revisited.

The rule that separates a useful watchdog from a dangerous one: **a bounded
repair, then an escalation**. The watchdog attempts a small number of
reversible repairs. If they do not work, it stops and tells a human, with
enough context to act. A watchdog that keeps restarting a service that cannot
start is worse than no watchdog: it turns one failure into a repeating event
that touches the service, the data and the notification channel every cycle.

## When to Use

- Writing a watchdog, a health-check timer, or a self-healing script.
- Adding a service, a port or a container to an existing watchdog.
- A watchdog restarted a service and the service still did not come up.
- Deciding whether a failure deserves an automatic repair or a human.
- A watchdog alert fires on a schedule for a condition that never resolves.
- A watchdog was edited and nobody knows whether the change still works.
- A watchdog, a deploy or a second watchdog now manages the same thing.
- An intentional stop of a guarded service produced an alert.

Do not use for: a one-off incident diagnosis with a human in the loop, or
scheduling a job that repairs nothing.

## The Rules

### The core pattern: fix-or-escalate

**1. A watchdog that restarts in a loop is worse than no watchdog.** The
cause is an unbounded retry against a failure that the retry cannot change.
A unit whose own comment said "retrying in a loop does not help" still
carried `Restart=on-failure` with `RestartSec=600`, and the restart counter
reached 39 against a permanent failure. The watchdog unit carries no
`Restart=` at all; the timer retries at a fixed cadence instead.

```bash
systemctl show <unit> -p NRestarts -p Restart -p RestartSec
```

Do not add `Restart=` to a watchdog unit to "make it more resilient".

**2. Every detection ends in exactly one of two outcomes: a bounded repair
or an escalation.** The decision is made in one place — a helper that takes
`--kind=<kind> --entity=<name> --context="<text>" --label=<topic>` and
applies the policy for that kind. Repair kinds apply a local, reversible fix;
repair-less kinds escalate directly; an unknown kind escalates. Do not put
the repair logic inside the watcher body: mixing "repair it" and "tell
someone" in the same script is what makes a watchdog dangerous.

**3. The repair path is bounded: `<attempts>` attempts, `<backoff>` seconds
between them, then escalate.** The attempt counter lives in the state file,
per entity, not in the watcher process, so it survives between ticks. After
the last attempt the entity is escalated and the counter is not reset until
the entity is confirmed healthy again.

```bash
attempts=$(( $(cat <state-file> 2>/dev/null || echo 0) + 1 ))
echo "$attempts" > <state-file>
[ "$attempts" -ge "<attempts>" ] && escalate
```

Do not reset the counter on a failed attempt.

**4. A repair that destroys something is never automatic.** Killing a
process, restarting a critical service, deleting data — those need an
explicit human decision. The policy lives in the helper, keyed by kind, so a
new watchdog reuses it by choosing the right kind and never by adding a
destructive branch to its own body.

**5. "Investigate first" means escalate, not repair.** When the correct next
step is a diagnosis only a human can make, the watchdog hands over the
context instead of improvising.

**6. The escalation carries three facts: what was broken, what was
attempted, and what is left.** Without them the human has to reconstruct the
incident from the journal, and the journal is where the watchdog's own
retries buried the evidence.

```
entity: <name>
state:  <what was broken>
tried:  <attempt 1: action> / <attempt 2: action>
left:   <what is still needed>
```

**7. An escalation that never reaches a human is not an escalation.** The
notification path has several hops: caller, wrapper, local API, device.
Verify the delivery in the receiver's own record, not in the wrapper's exit
code. If the push path is down, fall back to the system log so at least a
durable record exists.

### Detection: dead versus degraded

**8. A PID check is not a health check.** The process layer catches exactly
what `systemctl --failed` catches. It misses a container that is alive with
its port unbound, and it misses a service that answers with the wrong
content. Every critical service needs the three layers: process, wire,
content.

**9. The wire layer asks whether the port is listening where it is
expected.** A process bound to loopback while the client is on the network
is a process that is up and a service that is down. Check with
`ss -tlnH sport = :<port>`.

**10. The content layer asks a sample operation against the data path.**
Count files inside the container against files on the host, issue a HEAD and
assert a 2xx, or query a known-good row. This is the layer most often
missing from home-grown watchdogs and the one that catches the failures the
user actually notices.

```bash
host=$(find <path> -maxdepth 5 -type f 2>/dev/null | wc -l)
in_container=$(docker exec <container> find /<mount-target> -maxdepth 5 -type f 2>/dev/null | wc -l)
```

**11. A unit that is `enabled` but `inactive` for hours is usually a
restart-policy gap, not a crash.** A clean exit reads as success under
`Restart=on-failure`, which is the default for most hand-written units, so
the unit stays parked. Read the journal and the policy before acting.

```bash
journalctl -u <unit> --since "2 days ago" --no-pager | grep -iE 'deactivated|status=' | tail -3
systemctl show <unit> -p Restart -p RemainAfterExit
```

If the unit is meant to run continuously, widen the policy
(`Restart=always`, or `Restart=on-success,Restart=on-failure,RestartSec=5`).
If it is on-demand, the dead state is the design. Never let the watchdog
start it on every tick — that is the loop.

**12. A missing port is not always a failure.** Some services start on
demand. Check that the unit exists (`systemctl list-unit-files | grep
'<unit>'`) before the watchdog tries to raise it.

**13. A restart erases the evidence of the cause.** The journal after a
restart shows a clean start; the lines that explain why it died are gone.
Capture the last lines of the log before acting, and have the watchdog
record what it saw.

**14. A threshold must absorb the device's normal variation.** Some
hardware returns non-deterministic readings that differ by a unit or two
between two reads of the same idle state. A watcher that compares against a
baseline it treats as exact wakes the operator for nothing. Take the
absolute value, compare against a threshold above the normal variation, and
update the baseline and continue when under it.

```bash
delta=${delta#-}
if (( abs_delta < THRESHOLD )); then update_baseline; continue; fi
```

### Escalation and noise

**15. Alert on the transition, not on every tick.** A persistent failure
must not produce an alert on every cycle. Keep the last known state per
entity and alert when the state changes.

**16. Rate limiting is not the same as silence, and neither of them is
spam.** Persist a cache of consecutive failures per service
(`<name>|<count>`), alert only when the entity is new in the cache or when
the state changes (FAIL to OK), and log the repetitions as information
without a push. A failure cache is also what stops the watcher from
re-alerting on a condition it already reported.

**17. A notification failure inside a watchdog run is an outage of the
watchdog itself.** The wrapper can exit zero while the push was dropped, and
a watchdog that has watched silently for days is worse than one that is
broken loudly. Check the wrapper's exit code and the delivery record.

**18. Use the flag form of the notify wrapper and check its exit code.**
Positional arguments are rejected with an `unknown arg` error and a
non-zero exit; a caller that swallows that with `|| true` turns a broken
alert into silence. The call is
`<notify-wrapper> --to=<recipient> --title="<title>" --message="<message>" --source=<watcher>`.

**19. A permanent state gets one alert with a cooldown, not an alert every
cycle.** A remote host that is powered off and stays off will not fix
itself. Use a cooldown on the repeat notification (hours, not minutes) and
keep the count of consecutive failures so the escalation can say how long
the condition has been open.

**20. The dry-run switch must still write the state file.** A dry run that
skips the state update changes the behaviour under test. The switch skips
the notification, nothing else.

### Coordination with the rest of the host

**21. Two mechanisms that manage the same thing must not overlap.** A
watchdog that patches configuration and restarts a service on a heartbeat
transition must first write the configuration that stops the internal
component from fighting the external one — otherwise the two restart the
service against each other forever. Write the configuration, then restart,
and log both steps so the pair reads as one action.

**22. A watchdog must not fight a deployment.** Freeze, act, register,
rearm: stop the timer so no tick can fire mid-sequence, make the intended
change, register the new state without notifying, restart the timer. The
registration step is the load-bearing one — without it the next tick sees
the change as a fresh transition and fires an alert for an action the
operator took on purpose.

**23. An intentional stop is not a failure.** A guard that alerts on state
transitions reports the deliberate stop of the guarded service as a problem,
because its own state still says healthy. Use the same freeze-register-rearm
sequence with the dry-run switch set, and verify afterwards that no
notification was sent — the log must show the dry-run marker, not a
delivery.

**24. Unset the dry-run switch immediately after use.** A leaked
notification-suppression variable silently disables alerting for every
future run, and the failure looks like a healthy host.

**25. A watchdog must not act on services it does not own, and never on
data.** Its scope is the list it declares, nothing more. If the failure is
physical — a disk, a cable, a port — the policy is escalate-only: a restart
repeats the same action against the same hardware and changes nothing.

**26. The grace window is part of the contract.** A service that legitimately
takes minutes to produce its first output needs a warm-up state, and the
watchdog must compare against the moment the service started, not against
the wall-clock age of the artefact. Without that, every restart of the host
produces a false alarm.

**27. A watchdog that watches a retired target goes stale in silence.** A
watched port that nobody listens on, whose unit was removed, produces the
same alert every cycle and nothing else. Diff the watched list against the
live listeners and the installed units, with
`comm -23 <watched-ports> <listening-ports>`.

The fix is not to delete the entry: it is to re-point it at the service that
absorbed the retired one, in the right scope, and to clear the stale entry
from the failure cache so the next run does not report it as recovered. A
user-scope unit started from a system-scope service always fails to reach
the user bus; the cross-scope form names the target manager explicitly.

### Proving the watchdog works

**28. A watchdog that has never fired is unproven.** Test the failure path
with a synthetic input and a dry run. Never validate it by inducing a real
failure: the watch pushes for every run that finds a failed entry, including
the synthetic one you just induced and reverted, so the operator gets a
real alert for a problem you created.

```bash
<watcher> --dry-run          # logs the decision, sends nothing
```

**29. Extend an existing watchdog, do not rewrite it.** Back up the live
file with its permissions intact, add the new entries to the existing
arrays rather than creating parallel ones, syntax-check, then start the unit
immediately instead of waiting for the next tick.

```bash
cp -p <path>/watch.sh <path>/watch.sh.bak-$(date +%Y%m%d-%H%M%S)
bash -n <path>/watch.sh && echo SYNTAX_OK
systemctl start <unit>.service
tail -10 <logfile>
```

**30. Prove the success path after every change.** Read the log line the run
should have produced, then exercise the notification path by hand and
confirm the delivery in the receiver's record. Only then claim it works.

**31. Keep the file small enough to read.** Past a couple of hundred lines,
split it per class — one per service family — and dispatch from a master
script. A watchdog nobody can read is a watchdog nobody dares change.

**32. A watchdog and a service catalogue drift the same way.** When a
service is retired, moved or merged, the watchdog's map has to follow it in
the same edit, or it spends its life trying to raise a unit that no longer
exists. The sync check is the same diff used everywhere else: declared
entries against live listeners.

## Reference Files

- `references/fix-or-escalate.md` — the helper contract, the kind policy
  table, the bounded-repair counters, the escalation payload and the
  notification chain.
- `references/health-check-layers.md` — the three check layers, dead versus
  degraded, the on-demand exception, threshold drift and the stale-map diff.
- `references/watchdog-lifecycle.md` — unit and timer shape, state files,
  the extension procedure, the dry-run contract, intentional stops and
  teardown.

Load with `skill_view(name='watchdog-operations', file_path='references/fix-or-escalate.md')`.
Load with `skill_view(name='watchdog-operations', file_path='references/health-check-layers.md')`.
Load with `skill_view(name='watchdog-operations', file_path='references/watchdog-lifecycle.md')`.
