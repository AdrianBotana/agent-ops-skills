---
name: memory-architecture
description: "Layer table, store stack, performance envelope, backup and secret-leak runbook for a long-term agent memory pipeline."
---

# Memory architecture

## Layers

| Layer | Holds | How it is used |
|---|---|---|
| Always-on notes (`MEMORY.md`, `USER.md`) | Hard rules, conventions, identity only | Injected in every prompt. Keep under 60% of the character budget. |
| Semantic store (facts) | Loose facts: services, decisions, preferences, biography | Extracted automatically after each turn; preloaded before the next one |
| Long-form documents | Incidents, migrations, guides | Loaded on demand |
| Procedures (skills) | Step-by-step knowledge | Loaded on demand |
| Historical archive | Read-only record | Never read for live answers |

## What goes where

- A **hard rule** changes behaviour every turn ("never commit directly to the main branch"). Always-on file.
- A **fact** is true and durable ("the build server lives in the internal network"). Semantic store.
- A **procedure** is a sequence of commands. A skill file, not a fact.
- A **document** is longer than a paragraph. A document store, not a fact.

## Store stack

A production deployment usually has four moving parts:

1. **Extractor** — a small local model behind an OpenAI-compatible endpoint, at low temperature, with a prediction cap (1024 tokens is enough; a normal extraction returns about 750).
2. **Embedding model** — a separate small model, 768 dimensions is typical. It and the extractor must both fit in memory; the accelerator was pinned for over a hundred seconds per extraction when only one model could be resident at a time.
3. **Vector store** — an external service, never the embedded single-process build, because the agent opens several connections at once.
4. **Judge** — the same extractor model, called once per candidate fact with the source turn, returning a label (N/S/Q/T/P/F). Everything that is not P or F is deleted.

Two local patches are usually needed on the store plugin:

- **Custom extraction instructions.** Without them the plugin never forwards the instructions to the store, and the extraction stays noisy.
- **The fact judge.** Installed as a local module and called after every extraction.

Both patches are re-applied after the agent runtime is upgraded, because the plugin source changes under them.

## Performance envelope

| Operation | Measured cost |
|---|---|
| Extraction, warm | ~25 s |
| Extraction, cold | ~60 s |
| Preload wait | ≤ 3 s |
| One fact judge call | ~0.2 s |
| One embedding (extractor busy) | ~0.7 s |
| Normal extraction output | ~750 tokens |

The fixed extraction prompt is roughly nine thousand tokens, which is where the time goes. Store size barely matters.

## Vector store version pinning

Pin the store to a minor version and upgrade **minor by minor** (1.20 → 1.21 → 1.22). Never track `latest`: only consecutive minors are guaranteed to interoperate on disk. Put the upgrade order in a comment at the top of the compose file.

After a restart the collection may report `grey` because an optimisation is pending. That is not an error. A `PATCH` to `/collections/<collection>` with `{"optimizers_config": {}}` returns it to `green`.

## Backups

- A daily job takes a native snapshot into `.../memory_snapshots/memory-<timestamp>.snapshot` and keeps the last fourteen.
- Status is written to a state file; the restore drill asserts the newest snapshot is readable and is not more than two days old.
- Restore: `curl -X POST '<store>/collections/<collection>/snapshots/upload?priority=snapshot' -F snapshot=@<snapshot-file>`.
- Snapshot copies also live in the cold mirror and in the offsite. A secret purge must account for all three.

## Post-upgrade re-patch procedure

```bash
PY=$(ls -td ~/.agent/installs/*/environments/*/venv | head -1)/bin/python
cd <scripts-dir>
$PY patch_plugin.py
$PY check_recall.py --with-extraction
```

- `patch_plugin.py` re-applies both patches and syncs the instructions. It is idempotent. Exit code 1 means the plugin source changed and the patch has to be redone by hand at the new location.
- `check_recall.py` loads the plugin the way the runtime does and asserts four things: initialisation, preload injection of a test fact, extraction of a preference, and no fact left behind by a state report. It must print `check_recall: PASS`. If the preference appears on a second attempt, that is the known intermittent extractor failure, not a broken store.
- On failure: inspect the agent's memory status and the gateway journal.
- If the instructions or the judge were touched, run the judge's unit tests and a noise run (about one minute per round) against a **copy** of the collection.

## Secret-leak runbook

1. **Rotate the secret first.** Copies cannot be scrubbed reliably.
2. Find the store entry and delete it by id.
3. Purge the message history with a pattern-limited purge command, using the dry-run flag first.
4. Account for the remaining copies: recent snapshots, the cold mirror and its versioned subdirectory, the offsite, and the judge rejection log.
5. If the secret cannot be rotated, delete the affected snapshots.

## Experiments must not touch production scope

Every turn stays in the store's message history and the next real extraction sees it. A recall check cleans up after itself; for experiments use a separate collection and a temporary history database, then delete both.
