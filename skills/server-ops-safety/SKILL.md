---
name: server-ops-safety
description: "Use when an agent changes a running Linux host: Docker networking, firewall rules, systemd units, service ports, backup timers or credential files. Encodes the verification steps that keep a change from taking the host down."
version: 0.1.0
author: Adrián Botana
license: MIT
platforms: [linux]
---

# Server Ops Safety

Production runbook for agents that change a live host. Every rule states the
root cause, the exact symptom it produces, the command that proves it, and what
not to do. Nothing here is generic advice: each item is a failure that has
already happened on a real machine.

The order of the document is the order of operations. Read the tier, then the
source-of-truth rule, then the area you are about to touch.

## When to Use

- Changing firewall or NAT rules on a host that runs containers.
- Restarting, replacing or re-binding a service; moving a port.
- Editing systemd units, drop-ins or timers.
- Touching backup scripts, snapshots or restore drills.
- Adding, retiring or re-pointing a service in the catalogue or the proxy.
- Sharing, copying or rotating a credential.
- Publishing a new panel or app behind the reverse proxy.

Do not use for: read-only log inspection that changes nothing, or editing
application code inside a single repository.

## The Rules

### Operating posture

**1. Classify the change before you make it.** Tier A (autonomous): read-only
audits, log rotation, restarting a failed unit, committing work already
reviewed in the conversation. Tier B (explain, then act): one service, one
port, one unit fix, retiring a service you know is dying. Tier C (pause and
confirm): bulk deletion, mass renames, secrets rotation, anything that changes
the state of a shared or remote repository. Say which tier you are in before
acting. When the operator has delegated everything, do all of Tier A and B and
come back with Tier C as a short list of decisions.

**2. Runtime wins.** When the same fact lives in several places — the running
process, the config file in version control, the documentation, the operator's
notes — the order is runtime, then config, then documentation, then notes. When
runtime and config disagree, fix the config (or restart the service). When
config and documentation disagree, fix the documentation. Never edit the
documentation to match a broken runtime.

**3. Never declare something broken without live evidence.** Show the current
state and the last lines of the log. Guessing costs the operator more time
than the diagnosis does and destroys trust in every later report.

**4. A change is not done until the post-change check passes.** Finish with the
verification commands, not with a description of what you did. The check list
is in `references/verify-before-change.md`.

**5. A service is a three-place edit.** The catalogue, the proxy vhost and the
operator-facing documentation all describe the same service. Adding, retiring
or moving one means editing all three, in that order, or the drift shows up a
week later as a 502 or a stale entry. The procedure and the drift-detection
diff are in `references/verify-before-change.md`.

### Docker, firewall and NAT

**6. Never write `flush ruleset` on a host that runs Docker.** The flush
removes every nft/iptables table in the kernel, including the ones Docker
created for container masquerading. Docker does not recreate them on its own;
only a daemon restart does. From the flush until that restart, every container
has no outbound connectivity.

**7. The base unit can re-break the NAT after you fixed the config.** A
packaged unit that ships `ExecStop=/usr/sbin/nft flush ruleset` re-flushes the
ruleset every time the firewall service is stopped or restarted, regardless of
what the config file now says. Override it with an empty `ExecStop=` in a
drop-in, and read the base unit and the drop-ins, not just the config file.

**8. `ping` proves nothing about DNS.** The symptom that misleads: `ping` to a
public address works, name resolution returns nothing, HTTP requests return
`000`. Ping does not use DNS. The host can have full IP connectivity and no
name resolution at all, and the second state is what breaks the house.

**9. Check the NAT tables before blaming your change.** If the masquerade rules
are already at zero count, the breakage predates your edit and your edit is
not the cause. Count the rules before and after.

**10. `reload` is the correct verb for a firewall reload, and only once the
config no longer flushes the whole ruleset.** With a flushing config, both
`reload` and `restart` break connectivity for every container.

**11. Validate the ruleset without applying it.** `nft -c -f <file>` parses and
checks without installing. For a real test, copy the set into a scratch table
with a different name, load that, then delete the scratch table. Never test a
firewall change by applying it to production and rolling back.

**12. Restarting the container runtime fixes the NAT but takes everything down
with it.** It is the emergency escape hatch, not a repair. Containers with
`restart: no` do not come back on their own and must be started by hand.

**13. The container that resolves DNS for the network is a dependency of
everything.** If the DNS resolver runs as a container, stopping containers on
the data volume stops DNS for the host and for every device that uses it as
its resolver. Identify that dependency before any operation that stops
containers.

**14. A container whose port is reached from other machines must bind the
wildcard address, not loopback.** A service bound to `127.0.0.1` is
unreachable from any other device: the server's own `curl` against loopback
succeeds while a client on the network gets an immediate connection refused.
Bind `0.0.0.0` (or the specific interface) and confirm with `ss -tlnp`: the
local address column must show the wildcard or the LAN address.

### systemd units and services

**15. `systemctl --failed` reports "1 loaded unit" even when nothing failed.**
The summary line counts itself. For a clean check, count the entries with the
legend suppressed and compare against the baseline, not against the number the
tool prints.

**16. `journalctl -p err -b --since <time>` returns the whole boot, not the
window.** The "since boot" filter and the timestamp filter do not intersect the
way you expect: the timestamp trims the right edge, the boot filter pins the
left edge. To verify that a fix removed errors, use the timestamp alone, or use
`--since <t> --until <now>`.

**17. `EADDRINUSE` after a restart means an orphaned process still holds the
port.** Find it with `ss -tlnpH sport = :<port>`, kill that PID, then restart.
Do not restart repeatedly and hope the orphan exits.

**18. The real configuration of a unit often lives in a drop-in, not in the
unit file.** `systemctl cat <unit>` shows both; `systemctl show <unit>
-p Environment` shows the merged environment. Reading the unit file alone
hides environment overrides that change behaviour.

**19. A `systemctl --user` call from a system-scope service needs an explicit
bus.** A system unit running as a user has no user session bus in its
environment, so `systemctl --user start <unit>` fails with a bus connection
error every time. Use the machine form that routes to the target user's
manager without needing session variables.

**20. Never run a daemon-managed binary as your own user to see what it does.**
Launching it in the foreground creates a second instance with its own state
directory, generating credentials that are not the ones the service uses. Read
the unit, the service's own environment and its logs instead.

**21. A service that is `enabled` but `inactive` usually exited cleanly.**
The default restart policy restarts only on non-zero exits, so a clean exit
reads as success and the unit stays parked. Decide whether the unit is meant
to be always-on (then widen the restart policy) or on-demand (then the dead
state is the design). Read the unit and its controlling unit before changing
anything.

### Accelerators and GPU configuration

**22. A page-fault storm from the graphics driver is a runtime configuration
mismatch, not a driver bug.** Faults such as `GCVM_L2_PROTECTION_FAULT_STATUS`
with `Queue preemption failed` at model-load time are the classic signature of
an accelerator daemon configured for a different GPU architecture than the
installed card. Identify the real chip with `lspci -k | grep -A3 VGA`, then
read the daemon's environment for an architecture override and for a GPU count.
The override forces the wrong micro-architecture onto a card that runs its own
generation natively.

**23. The GPU count variable counts layers, not devices.** Setting it above
the number of physical GPUs saturates memory and produces failures that look
like driver faults. Set it to the physical device count.

**24. The fix lives in the daemon's environment file.** Do not recompile the
kernel, swap kernel modules, edit `/etc/modprobe.d/`, or install a second
compute stack to fix a runtime misconfiguration.

### Backups

**25. The snapshot must run before the mirror that copies it.** If the mirror
fires first, the cold copy contains yesterday's snapshot and the whole chain
is quietly a day behind. Give the snapshot an earlier schedule with enough
jitter to finish before the mirror starts.

**26. A backup timer can vanish from the host on redeploy.** Unit and timer
files that live only in a deployment repository get removed by a deploy that
replaces the directory. Check that the timer file exists on the host and that
it is enabled, not that the repository contains it.

**27. "Last run OK" is not "current state OK".** Report two independent
facts: is the backup target reachable right now, and what was the state of the
last copy. A remote that is up today can hold a copy that is weeks old, and
the copy can have failed while the remote was asleep.

**28. The status file is the source of truth, not the log and not the exit
code.** A service's success status only says the process exited zero; it
cannot tell you whether anything was copied. Parse the status file for the
result, the timestamp, the file counts and the free space after the run.

**29. A drill warning that repeats for weeks is an outage, not a warning.**
A backup drill that has been reporting the same problem on every run has
already failed silently for the user. Surface every open warning with how long
it has been open.

**30. The notification chain has several hops; walk it backwards.** The user
only sees "no notification". Test each hop from the closest to the farthest:
does the local API accept the push, what status did the push backend return,
when was the last success and the first failure, and does the credential
still authenticate. The gap between the last success and the first failure is
when the credential stopped working.

**31. A push failure in the middle of a drill output is how a broken backup
survives for months.** Treat any notification failure inside a drill report as
a real outage, even when the backup itself passed.

### Credentials and secrets

**32. Copying one credential into a second service is a legitimate emergency
exit, if you state the trade-off.** When the only broken thing is a token that
cannot be regenerated on the spot, reusing an existing working credential
restores service faster than waiting for the operator. The cost is coupling:
rotating it in one place breaks the other. Say so explicitly in the report.

**33. The credential the operator manages stays the operator's.** Regenerating
a long-lived token in the upstream UI, or editing the file that holds it, is
not the agent's call. Validate it, report the failure, give the exact command
the operator has to run.

### Change verification

**34. Use idempotent checks.** Every verification in this pack is a command
that can be run twice with the same result. The table of them is in
`references/verify-before-change.md`.

**35. Diff the catalogue against the live listeners before you declare
success.** Mismatches are drift, and the direction of the mismatch tells you
whether to fix the catalogue or to restart the listener.

**36. Verify a commit by content and by remote reference.** A new hash is not
proof of a push: the on-disk references lag and an amend regenerates the hash
whether or not the content changed. Compare the local commit with the remote
reference, and re-read the file to confirm the edit landed.

**37. Never retire an SSH key before you identify the one keeping the session
alive.** Key comments are unreliable: a key labelled with a retired machine
may still be the active one. Identify the key the live session is using,
keep that one plus the keys of the machines you are actually on, back the file
up, and only then remove the others. Removing the wrong key means the next
disconnect is a lockout with no way back in.

**38. A retired path must return 410, not fall through to a catch-all 200.**
A static site with a single-page fallback serves the home page for any path
that no longer exists, so a retired path keeps answering 200 and search
engines keep indexing it. Add an explicit `410 Gone` rule for the retired
paths before the catch-all.

**39. Do not harden after the incident is over.** If the symptom is gone and
the service is healthy, stop. A pending offline filesystem check on a
filesystem that already writes without errors is debt, not an emergency; it
costs downtime on a system that works. Report what remains as a list and let
the operator decide when to spend the downtime.

## Reference Files

- `references/docker-safety.md` — container networking, the NAT-flush trap, the
  firewall diagnosis ladder, and the post-change check.
- `references/backup-and-restore.md` — the backup chain, schedule ordering,
  the status-file contract, the restore layers and the drill semantics.
- `references/verify-before-change.md` — the idempotent verification command
  table, the three-place service edit, the drift diff and the SSH key
  retirement procedure.

Load with `skill_view(name='server-ops-safety', file_path='references/docker-safety.md')`.
Load with `skill_view(name='server-ops-safety', file_path='references/backup-and-restore.md')`.
Load with `skill_view(name='server-ops-safety', file_path='references/verify-before-change.md')`.
