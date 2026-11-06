---
name: watchdog-lifecycle
description: "Unit and timer shape, state files, the extension procedure, the dry-run contract, intentional stops, counter-reset traps and teardown order."
---

# Watchdog lifecycle

## Unit and timer shape

Service unit: `Type=oneshot`, `ExecStart=/usr/local/bin/<feature>-watch.sh`,
`TimeoutStartSec=10`, `Nice=10`. Timer unit: `OnBootSec=30`,
`OnUnitActiveSec=30`, `AccuracySec=5`, `Persistent=true`.

Thirty seconds is the default cadence: faster wastes CPU, slower makes a
transition feel laggy. Sixty seconds fits slow signals. Two rules on the
unit itself:

- **No `Restart=` on the watcher.** `Restart=on-failure` with a
  `RestartSec` in the hundreds of seconds accumulates one extra retry per
  cycle against a permanent failure; the observed counter reached 39. The
  timer retries at a fixed cadence on its own.
- **The script exits early when the feature is disarmed.** Do not disable
  the timer to stop the feature: disable plus enable creates boot-order
  ambiguity and races with a manual start.

## State files

One canonical file per feature, under the volatile runtime directory:
`/run/<feature>.state` holds the mode (`auto` | `off` | `manual`), and
`/run/<feature>.last-active` the timestamp of the last upstream "active"
signal.

- One source of truth. No per-state flags, no second "currently X" booleans.
- Derive "am I in state X" from the actual state file. A separate boolean
  lets manual and automatic flips fight each other.
- Update the "last active" timestamp **only on the active path**. Updating
  it every tick resets the idle counter and the threshold is never reached.
- Keep the two timestamps separate: "the upstream last showed activity" and
  "we last flipped our own state" are different facts.

## Extending an existing watcher

Do not rewrite it. Seven steps:

1. Back up the live file, preserving permissions and content:
   `cp -p <path>/watch.sh <path>/watch.sh.bak-$(date +%Y%m%d-%H%M%S)`
2. Patch the live file. Add new entries to the existing arrays (`SYSTEMD_SVCS`, `PORTS`, `DOCKER_CRITICAL`), not parallel arrays.
3. Syntax check before running: `bash -n <path>/watch.sh && echo SYNTAX_OK`
4. Start the unit now, do not wait for the next tick:
   `systemctl start <unit>.service`
5. Read the last log lines and confirm the expected summary line.
6. Exercise the notification path by hand and verify the delivery record.
7. Only then claim the watcher works.

Keep the file under 200 lines. Past that, split it per class and dispatch
from a master.

## The dry-run contract

A dry run proves the failure path without a real disturbance and without
alarming the operator.

- The switch is an environment variable, read at the top of the script.
- The dry run **logs the decision and sends nothing**. It still writes the
  state file, because skipping the state update changes the behaviour
  under test.
- Add the switch the first time you need it, not after the first alert.

Never validate the failure path by inducing a real failure and re-running
the live unit. The watch pushes for every run that finds a failed entry,
including the synthetic one you just created and reverted, so the operator
receives a real alert for a problem you made. Instead: restore the broken
configuration to a good state first, syntax-check, then trace the failure
block by hand against the synthetic input. For a true end-to-end run, use
the dry-run switch and confirm no new notification rows appeared.

## Intentional stops

A guard that alerts on state transitions reports a deliberate stop as a
problem: its own state still says healthy, so the next tick sees the
transition and fires. Freeze, act, register, rearm:

```bash
sudo systemctl stop <guard>.timer        # freeze: no tick can fire mid-sequence
sudo systemctl stop <guarded-service>    # the intended change
sudo systemctl set-environment <GUARD>_NOTIFY=0   # register WITHOUT notify
sudo systemctl start <guard>.service
sudo systemctl unset-environment <GUARD>_NOTIFY
sudo systemctl start <guard>.timer       # rearm
```

- The registration step is load-bearing. If the script exposes only an
  on/off, write the state file yourself before rearming.
- Freezing the timer first closes the race: a tick between the stop and the
  registration would notify on its own.
- Always unset the variable afterwards. A leaked `NOTIFY=0` silently
  disables alerting for every future run.
- Verify afterwards that **no notification was sent** — the log shows the
  dry-run marker, not a delivery — not merely that the state changed.

Do not silence the guard by disabling its timer for the whole window: you
lose alerting, and the first tick after re-enabling still fires.

## Teardown

Disabling is not the same as removing. The full order:

1. `systemctl disable --now <feature>-watch.timer`, stop the service unit, `systemctl reset-failed` both.
2. Remove the unit files and `systemctl daemon-reload`.
3. Remove the state-machine script and the watcher script.
4. Remove the state files.
5. Remove the backend endpoint or field the feature exposed. If other
   consumers read it, replace it with a hardcoded default.
6. Remove the frontend prop or badge classification.
7. Verify: the script is gone (`command not found`), the endpoint returns
   404 or omits the field, the panel renders.

Skipping the last three leaves dead references that become "why is this
broken" mysteries.
