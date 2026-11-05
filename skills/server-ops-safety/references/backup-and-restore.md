---
name: backup-and-restore
description: "Backup chain rules: layer ordering, schedule correctness, the status-file contract, restore layers and the notification-chain diagnosis."
---

# Backup and restore

A backup chain is a set of scheduled jobs that only work if their order, their paths and their result reporting are all correct. Each layer has a known failure mode.

## The layers

| Layer | Cadence | Purpose |
|---|---|---|
| Local mirror | daily | Accumulative mirror to a cold disk. No delete flag, so deleted files stay. Modified files are versioned by date. |
| Offsite mirror | daily | The only layer that survives a physical incident. Exact mirror (with delete) to a second machine. |
| System snapshot | on a timer | Root-only state: package selections, boot files, toolchains, users, crontabs. Required to rebuild a root filesystem. |
| Restore test | monthly | Samples random files from random directories and compares against live. Reports OK/WARN/FAIL by mismatch rate. |
| Restore drill | monthly | Validates the chain: mounts, snapshot presence, filesystem table entries, service configs, backup freshness, disk usage. |

## Ordering rules

**The snapshot must run before the mirror that copies it.** The mirror propagates whatever is on the live disk at trigger time. If the snapshot fires after the mirror, the cold copy contains yesterday's snapshot and the chain is quietly a day behind. Give the snapshot an earlier schedule with enough jitter to finish before the mirror starts.

**A timer unit can vanish from the host on redeploy.** Unit and timer files that live only in the deployment repository are removed when a deploy replaces that directory. Symptom: the log's last entry is older than a week and the snapshot directory's newest file is old too. Fix: copy the unit and timer back into the system directory, reload the daemon, enable and start the timer.

**A duplicated script in two locations diverges.** When the same script is installed in two paths and one copy has a guard the other lacks, the service may point at the unguarded one. Pick one canonical location and copy it over both.

**A purge block contradicts an accumulative mirror model.** If the model is "deleted files stay", any age-based purge in the script contradicts it. Wrap the purge in a disabled conditional so the code stays available but does not run, and say so in the documentation.

## The status-file contract

The status file is the source of truth — not the log, not the service exit code. A successful exit only says the process ended with code zero; it cannot tell you whether anything was copied. Parse the status file for: result (`OK|WARN|FAIL|RUNNING`), ISO timestamp, elapsed seconds, file counts per tree, free space after the run.

**Report two independent facts, never one.** Is the target reachable *now*, and what was the state of the *last* copy? A target that is up today can hold a copy that is weeks old, and the copy can have failed because the target was asleep at trigger time.

**No retries.** The offsite job preflights the target and writes `FAIL` then exits non-zero when it is unreachable. It does not wait for the target to wake up, so that day's copy simply does not happen.

**Reachability probe.** A plain TCP connect to the target's SSH port is enough and does not open a session. That is what the preflight does; reuse it for dashboards and scripts.

## Turning a powered-off target into a 24/7 server

1. **Mask the sleep targets** so the command fails instead of taking the machine down. Mask them one target per call — several at once fails silently — and delete any file in the system unit directory that blocks the mask.
2. **Kill idle suspend in logind** with `IdleAction=ignore`, `HandleLidSwitch=ignore`, `HandlePowerKey=ignore` in a logind drop-in. Do not restart the logind daemon to apply it: it kills the session running the change and is unnecessary.
3. **Enable wake-on-LAN persistently** with a oneshot unit and `RemainAfterExit=yes`. The NIC setting does not survive a reboot.
4. **Watch it from the hub** by extending the existing health watchdog — do not create a parallel probe. Check the port, send the magic packet, wait, then alert. The broadcast only reaches the machine on the same physical network.
5. **Verify the boot path**: the service units, SSH, the firewall and the wake-on-LAN unit must all be enabled on the target.
6. **Auto power-on after a power cut is a BIOS setting** ("Restore on AC Power Loss"), not a Linux one. Flag it as a manual step for the operator.
7. **Keep the shutdown possible on purpose.** Powering the machine off over SSH must keep working: it should be a deliberate decision.

## Notification chain

Four hops: caller script, thin client, local HTTP API, push backend. The user only sees "no notification", so walk the chain backwards.

1. Does the local API accept the push? Expect an accepted status and a body carrying the backend's own status.
2. Read that status field. `200` means delivered. `401` means the credential was rejected. `0` means the backend was never called.
3. Pinpoint the break: the last successful row and the first rejected row. The gap between those timestamps is when the credential stopped working.
4. Verify the credential yourself against the upstream API. `200` means it works, `401` means it is dead — usually rotated in the upstream UI.

**The credential the operator manages stays the operator's.** Regenerating a long-lived token in the upstream UI is not the agent's call. Validate it, report the failure, give the exact command the operator has to run.

**Copying a working credential into a second service is a legitimate emergency exit, if you say so.** When the only broken thing is a token that cannot be regenerated on the spot, reusing an existing working credential restores the service faster than waiting. The cost is coupling: rotating it in one place breaks the other. State the trade-off in the report.

## Restore layers

| Layer | What it holds |
|---|---|
| Rescue copy | Critical configuration and state dumped from the damaged filesystem |
| Cold mirror | The full project tree as of the last daily mirror |
| Salvage copy | A copy of the live filesystem taken at the moment of the incident |

**The salvage copy is the only copy of large directories with no cold backup.** Before declaring anything lost, check the cold mirror; if it is not there, copy the live tree to a spare disk first.

**Restore order:** base from the clean pre-damage copy, overlay the most recent copy, then repair from the base any file the recent copy has corrupted. A recent copy taken after the incident carries the files that were zeroed by the failure, and files that were being written when the copy ran come out truncated.

**Detect corrupt files against an independent source, not against the same one.** Walk the tree and flag files whose first blocks are all zero. A legitimate binary can start with zeros and be fine — confirm against the cold mirror before declaring it lost.

**Permission errors on root-only files mean retry as root**, not leaving a hole. A directory read error with a "bad message" errno means a corrupt inode in that directory; replace it from the cold mirror. **Count the volume per directory**, not only in total, and check free space in the destination before rebuilding.

## What a WARN means

A restore test that warns with "source missing" or "no eligible files" is usually a **path** failure, not corruption: the backup changed location and the script still compares against the old path. Check the paths before "fixing" the backup; the fix is to change the source, not the data. A low mismatch percentage with differing modification times is ordinary drift — the file changed after the last mirror — not corruption. An open warning that has been repeating for weeks is an outage: report every open warning with how long it has been open.
