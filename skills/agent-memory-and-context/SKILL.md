---
name: agent-memory-and-context
description: "Use when an agent must remember facts across sessions, size its always-on context, boot a session with the right context, or keep a long-term store clean and secret-free."
version: 0.1.0
author: Adrián Botana
license: MIT
platforms: [linux]
---

# Agent Memory and Context

Long-term memory for an agent is a pipeline, not a file: an always-on block that
must stay tiny, a semantic store that absorbs facts automatically, long documents
that load on demand, and procedures that load only when the task calls for them.
This skill covers the layer boundaries, the noise control that keeps a store
trustworthy, the session bootstrap that loads context before answering, and the
secret-handling rules that must never be violated.

## When to Use

- The user refers to something from an earlier session ("the thing we set up last week").
- The user states a durable fact about themselves, their work, or their infrastructure.
- The user asks the agent to forget, correct, or re-check a remembered fact.
- A session must start with the right context loaded (host names, service names, personal facts).
- The always-on context file is growing, or the semantic store is returning junk.
- A secret may have leaked into a memory layer or a prompt.

Do not use for: code review, debugging a stack trace, or building a retrieval
system for a product that is not an agent runtime.

## The Rules

### Always-on context

**1. A new fact never goes into the always-on file — the semantic store extracts it.** The always-on block is re-read on every turn, so every line in it is paid for on every turn. Writing a fact there duplicates the store and grows the prompt for no benefit.
Symptom: the always-on file creeps past its cap while the store stays empty.
Do: write the fact once into the semantic store, and only for facts that are important and explicitly stated.
Do not: write the same fact into both layers.

**2. The always-on file changes only to add a hard rule, and only while it stays under its cap.** Hard rules are conventions that change how the agent behaves every single turn. Facts, state and history belong in the store.
Symptom: the file holds user biography, service inventories or one-off decisions.
Rule of thumb from a live deployment: keep the pair of always-on files under 60% of their character budget, and re-check the ratio after every addition.

**3. Never store a secret in a memory layer.** Memory blocks are plain text injected into prompts; a token written there is leaked on every subsequent turn and ends up in logs, snapshots and backups.
Symptom: a token, password or API key appears in a memory file, a store entry, or a judge log.
Do: use a secret manager or a vault, and read the value at runtime with a short-lived command.
Do not: treat a memory layer as a credential vault. If a secret already landed there, see rule 16.

**4. The preloaded block is already a search result.** If the answer is in the block that the runtime injects at session start, searching again is pure latency.
Do: search the store only when the block does not answer and the user referenced the past.
Do not: search on every turn "to be safe".

**5. Point-in-time state is not a memory fact.** Uptime, swap usage, which containers are running, free disk, current load — all of it is stale within minutes.
Symptom: the store fills with "the server was at 40% memory" type entries that are wrong on the next read.
Do not: store live state. Query it when it is needed.

**6. Test data must be labelled and deleted.** A plausible invented fact gets stored and later returned as if it were real.
Symptom: the store reports a hobby, a date or a location that the user never mentioned.
Do: prefix every test fact with a `TEST DATA (delete)` marker, use a throwaway collection and a throwaway history database for experiments, and purge the entries afterwards.

### Noise control in the semantic store

**7. Extraction noise comes from what the extractor is fed, not from the model being weak.** Store implementations typically inject the last N messages of the scope plus the N most similar existing memories into every extraction call. A small model re-extracts paraphrases of both, and turns state and requests into durable facts.
Measured on a live store with the same corpus: 1 of 3 clean rounds with no defences, 2 of 6 with extraction instructions only, 6 of 6 with instructions plus a fact judge.
Do: add extraction instructions, then add a judge.

**8. A fact judge needs a closed label set, and it must delete aggressively.** The judge sees the source turn next to each candidate fact and returns one letter: N (the turn does not state it), S (state), Q (a request or an action taken), T (test data), P (preference), F (durable fact). Everything that is not P or F is deleted.
Symptom: the store fills with "the user asked about X" and "the server had 4 GB free".
Do: keep the judge running on the same model as the extractor, and log every rejected fact with its source turn.
Do not: let the judge keep S and Q. State and requests are not durable facts.

**9. Similarity-based filtering is the wrong defence.** Paraphrases of a legitimate update score 0.80–0.97 against each other and overlap with real changes, so a similarity threshold deletes legitimate facts. Removing the existing memories from the extraction prompt is worse: the extractor then re-extracts what is already stored.
Do: rely on exact-hash de-duplication plus the judge.
Do not: add a similarity cutoff, and do not strip existing memories from the prompt.

**10. Extraction latency is a fixed cost of the prompt, not of the store size.** The extractor sends a fixed prompt of roughly nine thousand tokens, so an extraction takes about 25 seconds warm and about 60 seconds cold, and a turn is skipped if the previous extraction is still running.
Design consequence: never block a turn on extraction. The preloader waits at most three seconds, and the backend needs a moment to come up at session start.

**11. Cap the extractor's generation length or it loops.** At low temperature the extractor sometimes repeats indefinitely; one run produced about sixteen thousand tokens and pinned the accelerator for ten minutes. A cap of 1024 predicted tokens is enough — a normal extraction produces about 750.
Symptom: the accelerator is saturated with no user activity, and the turn's facts are missing because the extraction was truncated.
Do: set the prediction cap, and treat a truncated extraction as "that turn was not extracted".

**12. A store outage at session start silently disables memory for that session.** The plugin does not retry, so sessions opened before the store is ready run with no memory at all until the next session starts.
Symptom: the agent has no recall in the first session after a host reboot or a store restart.
Do: pause the health-watchdog timer before store maintenance, verify the store answers before starting real work, and treat "no memory this session" as a known failure mode, not a bug in the store.

**13. Back up the store natively and drill the restore.** A file-level copy of the store directory is not a backup if the store has its own consistency requirements.
Recipe: a daily native snapshot into a timestamped directory, keep the last fourteen, and a restore drill that proves the newest snapshot is readable and is not older than two days.
Restore: `curl -X POST '<store>/collections/<collection>/snapshots/upload?priority=snapshot' -F snapshot=@<snapshot-file>`.

### Corrections, deletions and leaks

**14. A correction is a new write, not an edit of the old entry.** Editing only the entry the user happened to quote leaves the stale copy in the layer that gets read most.
Symptom: the agent keeps repeating the old value even though the user corrected it once.
Do: update every layer that carries the wrong value in the same pass, then add the corrected value as its own fact so a direct keyword lookup finds it.

**15. Never infer an attribute from a keyword.** A place name inside a hobby fact is usually the project's location — a marina, a workshop, a route — not the user's home. Propagating it into always-on memory is how a wrong attribute becomes permanent.
Do: when a term is ambiguous across contexts, ask which one it is.
Do not: pick one reading and store it.

**16. A leaked secret: rotate first, then clean.** Copies cannot be reliably scrubbed, so the credential must be considered compromised before any cleanup runs.
Order: rotate the secret; find the store entry and delete it; purge the message history with a pattern-limited purge command (dry run first); then account for the remaining copies — recent snapshots, cold mirrors, offsite copies, and the judge rejection log.
If the secret cannot be rotated, delete the affected snapshots instead.
Do not: start by cleaning and hope rotation is unnecessary.

**17. Rejected facts are recoverable from the judge log.** When the user says the agent forgot something it was told, the store may be missing it because the judge rejected it.
Do: check the judge rejection log, and re-add the fact explicitly if it was a good one.
Do not: assume the user never said it.

### Session bootstrap

**18. Probe the store before answering, and do it silently.** The bootstrap is a warm-up, not an announcement.
Routine: derive entity names from the request, probe them in parallel, load only the procedures the probed entities imply, and skip any lookup the preloaded block already answers.
Do not: narrate the probe, probe twice, or load unrelated procedures.

**19. A newly stated biographical fact is written immediately, without asking for confirmation.** The user should not have to say "remember that".
Symptom: facts only land when the user explicitly demands it, and "what do you know about me" becomes a recurring manual bug.

**20. Trigger the bootstrap on infrastructure words and on biography, not on every turn.** Two trigger families matter: infrastructure vocabulary (a host name, a storage path, a private address, a service name, a project name) and biographical statements (role, employer, residence, family, age).
Do not: probe on unrelated creative or research work — the warm-up costs tokens for nothing.

**21. Cite a probed fact with its identifier and its trust score.** A reply that states a remembered fact should carry the reference so the user can audit where it came from, for example `[fact_id 12, trust 0.5]`.

### Long-form knowledge and procedures

**22. A remembered fact about running state is not evidence — query the runtime.** Memory and notes are the last source in the truth order: runtime first, then the declared configuration, then the human documentation, then notes and memory. Anything below the runtime can drift.
Symptom: the agent answers "the service is up" from a stored fact while the unit is actually failed.
Do: for anything about current state, check the runtime and say what you checked.
Do not: treat a stored fact about a service's status as current.

**23. Export the old store before you migrate the memory backend, with secrets redacted.** The export is the only way back to the previous facts, and a secret that was stored by mistake has to be redacted in the export too.
Do: keep the export, note the date, and confirm the file is readable before switching.
Do not: switch backends in place and hope the old data is still reachable.

**24. When a procedure or note exists in two places, patch both and verify they are identical.** A live copy and a versioned copy drift apart, and the live copy is the one that gets served.
Symptom: a change appears to have been saved but the live copy still serves the old text.
Do: patch the versioned copy, copy it across, and compare the two files byte for byte.

**25. Keep the fallback path working.** The always-on files must stay valid with the semantic store disabled, because that is what an operator falls back to when the store is broken.
Do: keep the always-on rules self-sufficient for the behaviour that matters most.

## Verification

- [ ] The always-on files parse, contain only hard rules, and sit under their character budget.
- [ ] No secret appears in any memory layer, snapshot, or judge log.
- [ ] The store returns the fact the user stated, from the layer that was supposed to hold it.
- [ ] A recall check passes: preload injects a known fact, a preference extracts, a state report leaves nothing behind, and the check cleans up after itself.
- [ ] The newest snapshot is readable and is not older than the drill's freshness limit.
- [ ] Every layer of the bootstrap that fired has a matching procedure loaded, and no others.
- [ ] A corrected fact returns the new value from a direct keyword lookup, not only from the entry that was edited.

## Reference Files

- `references/memory-architecture.md` — the layer table, the store stack (extractor, embedder, vector store), the measured performance envelope, backup and restore procedure, the post-upgrade re-patch procedure, and the full secret-leak runbook.
- `references/session-bootstrap.md` — the trigger families, the parallel probe routine, the auto-write rule, the citation format, the anti-pattern list, and the failure mode this closes.

Load with `skill_view(name='agent-memory-and-context', file_path='references/memory-architecture.md')`.
Load with `skill_view(name='agent-memory-and-context', file_path='references/session-bootstrap.md')`.
