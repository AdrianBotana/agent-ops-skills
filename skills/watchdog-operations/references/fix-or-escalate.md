---
name: fix-or-escalate
description: "The helper contract, the kind policy table, bounded repair counters, the escalation payload and the notification chain behind a watchdog."
---

# Fix or escalate

## The two paths

A watcher detects, then chooses exactly one of two paths. That choice belongs
in one place — a helper — not in the watcher body. Mixing "repair it" and
"tell someone" in the same script is what makes a watchdog dangerous.

```
<watcher> --kind=<kind> --entity=<name> --context="<text>" --label=<topic>
```

The helper holds the policy for each kind. The watcher only classifies.

## Kind policy table

| Kind class | Action | Reason |
|---|---|---|
| Releasable resource (a pinned runtime, a warm model held in memory) | Repair: release the resource through its own API, then verify. Escalate if the release fails. | The resource is not the service. Releasing it is reversible and local. |
| Paused workload the watcher owns | Repair: unpause, verify, escalate on failure. | Local, reversible, in scope. |
| Destructive process kill | Escalate only. | A kill is not reversible and takes unrelated state with it. |
| Physical hardware fault (a CRC counter that keeps growing, a link that keeps dropping) | Escalate only. | The cause is the hardware. Restarting the service repeats the action against the same hardware and changes nothing. |
| Configuration-only fault (a missing tuning key, a coverage gap) | Escalate only. | The fix is an editorial decision, not a mechanical one. |
| Unknown kind | Escalate. | An unclassified failure is not a repair candidate. |

Policy lives in the helper, not in the watcher, so a new watcher reuses it by
choosing a kind. The scope of "auto-fix" differs per system; a policy that
assumes one system's permissions breaks on another.

Never auto-apply a destructive fix — killing processes, restarting critical
services, deleting data — without explicit human approval. A comment that
says "investigate first" expresses that intent: it means escalate.

## Bounded repair counters

The counter is per entity and lives in a file, because the watcher process
does not survive between ticks.

```
<cache-file>   one line per entity: <name>|<count>
```

Rules:

- Increment only on a failed repair, never on a detection needing no repair.
- Compare against a limit (`<attempts>`) and a delay (`<backoff>`) between
  attempts.
- When the limit is reached, escalate and keep the counter. Do not reset it
  on a failed attempt — that is the loop with extra steps.
- Reset the counter only when the entity is confirmed healthy again.
- Notify only when the entity is new in the cache, or when the state changes
  (FAIL to OK). Log the repetitions as information without a push.

Without the cache, a permanent failure produces the same alert every cycle:
the observed rate was seventeen identical entries in two hours, all of them
rate-limited, so the operator saw nothing.

## The escalation payload

Three facts, always:

```
entity: <name>
state:  <what was broken>
tried:  <attempt 1: action> / <attempt 2: action>
left:   <what is still needed>
```

The journal is not a substitute: the retries bury the evidence.

## The notification chain

Four hops: caller script, wrapper client, local API, companion app that
pushes to the device. The user only sees "no notification", so walk it
backwards from the closest hop:

1. Does the local API accept the push? Expected `202` with a body carrying
   the downstream status.
2. What did the downstream say? `200` accepted, `401` credential rejected,
   `0` never called.
3. When was the last success and the first failure? The gap between those
   two timestamps is when the chain broke.
4. Does the credential still authenticate?

```bash
sqlite3 <notify-db> \
  "SELECT datetime(ts), title, source, ha_status FROM notifications ORDER BY id DESC LIMIT 10;"
```

The column names are `ha_status` and `ha_response`. Querying `response` or
`error` returns nothing and reads as "no data".

If the push path is down, fall back to the system log so the event is not
lost: `logger -t <watcher> -p user.err "<message>"`.

## Calling the notify wrapper

Use the flag form. Positional arguments are rejected with an `unknown arg`
error and a non-zero exit, and a caller that appends `|| true` turns a
broken alert into silence:

```bash
<notify-wrapper> --to=<recipient> --title="<title>" --message="<message>" --source=<watcher>
```

Check the exit code. A non-zero exit means the push did not land, and the
watchdog must treat that as a failure of the alert path, not of the service.

## Verifying an escalation

After changing a watcher, exercise the helper by hand with a known kind and
confirm the push reached the expected topic with the expected escalation
tag. Then confirm the helper logged the run with
`journalctl -t <helper-tag> --since "5 min ago" --no-pager`. Only then
claim the watchdog works.
