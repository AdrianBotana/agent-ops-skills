---
name: scheduler-and-cron-audit
description: "Parallel scheduler inventory, reconciliation checklist, job-creation CLI shape and token accounting queries."
---

# Scheduler and activity audit

## Inventory, in parallel
Run all seven in one round. Every one is read-only and independent.

1. The agent's own scheduled jobs — the job list and the job status commands.
2. Root crontab. 3. The operator's user crontab.
4. `/etc/cron.d/` — list and read each file, skip the placeholders.
5. `/etc/cron.hourly`, `.daily`, `.weekly`, `.monthly`.
6. System timers — list all, including inactive ones. 7. User timers — same.

For each entry capture: name, schedule, last and next run when known, type, and a one-line purpose from the script header or the unit description. Do not load whole scripts into the context — metadata only.

## Reconcile
**Duplicates** (the same script driven by two schedulers), **deaths** (an entry whose target script no longer exists), **silent failures** (an active job with no watched log).

## Creating a scheduled job: the real CLI shape

```
<agent-cli> cron create '*/15 * * * *' --script <name>.sh --no-agent \
  --workdir=<path> --deliver <channel>
```

- The schedule is **positional**. Passing it as a named flag creates nothing while still printing a success message.
- The script argument wants a **bare filename**, not an absolute path.
- The no-agent flag makes the script the entire job and requires a script argument. Empty stdout means nothing is delivered — exactly the alert-only semantics a watchdog wants.
- Always confirm afterwards by listing the jobs. A green CLI message is not proof the job exists.

## Verify a script before wiring it to a schedule

```bash
bash <script>; echo "exit=$?"
```

Watch for `set -u` interacting with positional parameters: a function using `$1` inside a loop or before assignment aborts the whole script. Also watch for unquoted heredoc and `$1` collisions when the script is written through a file-write tool.

## Migration rule
Before proposing to move a job from a cron directory or a timer into the agent's scheduler:

- Check what the script needs: bash-only is fine, anything that drives the init system or its cgroups belongs to the host side. Check which paths and credentials it touches — secrets in system directories migrate as they are, do not re-wrap them.
- If it already runs correctly under a timer, **leave it alone**. Migration that adds a layer is technical debt, not improvement.
- The agent's scheduler gains visibility (dashboard, delivery on failure). Migrate only jobs whose failure mode is currently silent.

## Token accounting
### Fresh versus cached
Usage audits usually report prompt tokens that include cache reads. Always split them:

```sql
SELECT SUM(m.input_tokens) fresh, SUM(m.cache_read_tokens) cached,
       SUM(m.output_tokens) out, COUNT(DISTINCT m.session_id) fires
FROM session_model_usage m JOIN sessions s ON s.id = m.session_id
WHERE s.source = 'cron';
```

The actionable metric is **fresh tokens per firing**, not the total. One real host spent 7.26 M audit tokens on scheduled jobs, of which only 679 k were fresh; counting cache as consumption overstates the problem roughly tenfold. A watchdog that reads a diff and sends one message should stay under about 150 k fresh tokens per firing.

### Timestamps are epoch seconds

```python
db.execute("... where started_at >= ?", (int(time.time()) - 86400,))   # correct
db.execute("... where started_at >= ?", (now_ms - 86400_000,))         # wrong: returns zero
```

Comparing a seconds column against a millisecond clock makes every window filter return zero, silently. The report looks empty while the data is correct.

### Day keys are local, not UTC
`date(started_at,'unixepoch','localtime')` gives the local date. An ISO-string formatter converts to UTC and shifts the key by a day across midnight. Use an explicit local formatter on both sides.

## Activity snapshot without the dashboard

```bash
python3 - <<'EOF'
import sqlite3, os, time
c = sqlite3.connect(f"file:{os.path.expanduser('~/.agent/state.db')}?mode=ro", uri=True)
now = int(time.time())
print("24h:", c.execute("""SELECT COUNT(*), SUM(message_count), SUM(tool_call_count),
  SUM(input_tokens+output_tokens) FROM sessions WHERE started_at >= ?""", (now-86400,)).fetchone())
print("unclosed:", c.execute("SELECT COUNT(*) FROM sessions WHERE ended_at IS NULL").fetchone()[0])
print("fresh per cron firing:", c.execute("""
  SELECT SUM(m.input_tokens)/COUNT(DISTINCT m.session_id) FROM session_model_usage m
  JOIN sessions s ON s.id=m.session_id WHERE s.source='cron'""").fetchone()[0])
EOF
```

Read the state database **read-only**: it works against a large write-ahead log without blocking the running agent (about 15 ms). Only the sessions table is needed; the message table is large and its full-text indexes are unnecessary.

## Unclosed sessions
The prune command reports: "N open sessions also match these filters but will be skipped because prune only deletes ended sessions". Sessions the scheduler opens and never closes — the process died before finalising — stay open forever with zero messages, no title and zero tokens. Real host: 120 of 213 unclosed, 83 created in a single day. Deleting them is destructive: treat it as a confirmation-required action.

## Context bloat arithmetic
A loaded procedure file stays in the session context for the rest of the session and is re-paid on every later call. Measured on a real host: 105 loads accounted for 4.5 M characters (~1.13 M tokens) — 39% of all tool output; 19 loads exceeded 100 000 characters; one 170 KB file loaded four times in the same session added about 176 000 tokens.

Splitting large procedure files into a hub plus per-topic references:

| File | Before | After | Tokens per load |
|---|---|---|---|
| admin | 170 574 ch | 23 793 ch | 42 643 → 5 948 (−86.1%) |
| orchestrator | 102 194 ch | 23 383 ch | 25 475 → 5 846 (−77.1%) |
| build | 103 320 ch | 23 140 ch | 25 754 → 5 785 (−77.5%) |
| services | 79 349 ch | 21 022 ch | 19 758 → 5 256 (−73.4%) |

### How to split without losing content
1. A script cuts the body on the `##` and `###` headings and writes each large section to a per-topic file **as a literal copy**.
2. The new hub keeps the frontmatter, the always-on sections and an index with one line per file.
3. Verify against a backup: every original paragraph over 120 characters must still be present, in the hub or in some reference file.

Verification drops: **paragraphs** (every original paragraph over 120 characters present); **lines** (every original paragraph over 40 characters present — normalise whitespace and strip backticks before comparing, or a reference that rewrites `` `x` `` as `x` reports a false loss); **headings** (every original `##` and `###` heading present); **numbering** (copy the original numbers literally — some rule lists have duplicate numbers and gaps, and renumbering breaks cross-references); **cross-references** (if one skill points at a file that lives in another skill, create a symbolic link rather than duplicating it, and verify it resolves).

## Latency and parallelism
- Correlation between output tokens and latency: strong. Between context tokens and latency: negligible. Lowering the compression threshold therefore does not speed anything up; it only reduces cache-read spend.
- What helps: fewer tool calls per turn, shorter outputs, no long serial command queues.
- Write-capable tools are not marked parallel-safe, so two of them in the same turn serialise. Measured parallelism per session ranged from 0% to 91% with the same runtime and model — the limit is the working style, not the infrastructure.
- The lever is instruction ("batch the read-only calls"), not configuration. On one host, 89% of shell calls were read-only and batchable; grouping them four at a time saved roughly 950 turns.
