---
name: infra-audit-checklist
description: "Use when auditing an environment before touching it: third-party software, deployed bots, schedulers, credentials, public exposure, and orphaned projects."
version: 0.1.0
author: Adrián Botana
license: MIT
platforms: [linux]
---

# Infra Audit Checklist

An audit answers one question per surface: *what is actually there, what does it
expose, and what will break if I touch it*. Every rule below is ordered so a
read-only sweep comes first, a measured number replaces an opinion, and every
finding is verified against a primary source before it is reported.

## When to Use

- The user asks "is this safe", "does it leak my data", "should I install it".
- The user asks for a full audit of a host, a stack, or a deployed bot.
- The user asks what runs on a schedule, what watches what, or where the spending goes.
- The user asks whether the credentials are still valid, or what is exposed to the internet.
- Before a bulk cleanup, to know which projects are genuinely orphaned.
- After a dependency upgrade of the agent runtime or a plugin.

Do not use for: reviewing your own diff before a commit, debugging a concrete
stack trace, or a feature comparison.

## The Rules

### Standing rules for every audit

**1. Verdict first, one line, then evidence.** The strongest reason goes in sentence one. Feature lists, controls and caveats go below the verdict. Burying the answer inside an options list gets rejected every time.
**2. Code, not copy.** Evidence comes from what the binary or the source actually does. Store listings, project READMEs and developer-written privacy manifests are checked *after* the code-level signals. A pattern in the source that hits a third-party endpoint beats a paragraph claiming "no tracking".
**3. "Safe" means "will not leak data".** Not "well engineered", not "stable", not "fast". Every check focuses on outbound traffic, default-on third-party services, what gets logged, and what travels unencrypted.
**4. Every quantitative claim carries its number.** "1 of 6 responses empty", "9 831 of 10 266 characters are reasoning". Without a number it is an opinion.
**5. Separate what you measured from what you inferred.** "Measured 6 times" and "probably" are different classes of statement and must read differently.
**6. Deliver the artifact, not the transcript.** If the audit produced a script or a report, save it to a re-runnable location and link it. Audit output that lives only in chat is work thrown away.
**7. Do not leave changes behind.** The user asked for an audit, not a fix. Delete or commit separately any test or probe script you created, and leave other people's uncommitted work untouched.

### Third-party software

**8. State the analysis depth before anything else.** Open source means you clone and read the code. Closed source means metadata only — the privacy manifest, the dependency list, the store listing. Saying "closed source, I can only see metadata and the manifest" is fair; pretending to have read source you did not is not.
**9. Enumerate every external host before reading anything else.** Telemetry, analytics, relays, payment SDKs, font CDNs, push relays. Zero to three entries is clean; four or more is a smell. For each one: where it is called from, what it carries, whether the user can turn it off.
**10. Classify each external host by who controls it.** A server the user runs is expected. The author's own cloud is acceptable when the project is open and auditable, suspicious when it is closed. A third-party relay, CDN or analytics endpoint is the red flag — and if it is not user-toggleable, the verdict is no.
**11. Treat the privacy manifest as a claim, not a finding.** If the manifest declares no tracking but the dependency list contains an analytics or crash-reporting SDK, the manifest is wrong. The binary still leaks.
**12. DEBUG-guarded logging does not ship.** Logging compiled only in debug builds is stripped from release builds. Confirm the build guard before flagging it; persisted release logging is what matters.
**13. Distinguish a relay from a reader.** A privacy-preserving push relay encrypts the payload on the server with a per-install key the device holds, so the relay routes ciphertext plus a device token but cannot read the content. Verify the encryption is actually there before calling it a leak. Do not conflate "third-party relay" with "third-party can read my messages".
**14. A plaintext-HTTP override for arbitrary hosts is a red flag; a local-networking override is not.** Permitting plain HTTP to local-network hosts is normal for a self-hosted client. Permitting it to any host is not.
**15. Closed-source means saying so.** Suggest building from source or requesting a pre-release build if that is what the user needs. Do not present "maintain a custom fork" as the fix without naming the cost: signing, no auto-updates, maintenance.
**16. Store ratings validate nothing about security.** They say "does not crash and works as advertised". Do not cite them either way.

### Deployed bots and agents

**17. Audit against the user, not against the code.** The code can be perfect and the bot useless. The acceptance criterion is: does the user reliably get what they came for? Order the findings by impact on the user, not by technical severity — a concurrency bug that corrupts the history outranks a stray log line.
**18. Start with real usage.** Count the messages in the store and the number of distinct users. A bot with a handful of messages in its whole history is validated by nothing; there is no evidence its architecture works. Say that in the report — it is the context that makes the other findings matter.
**19. Measure the empty-response rate; the model answering empty is the bug that hides best.** Run the real prompt (persona plus mode plus retrieval) several times and count empty responses and latency. Any rate above zero is a critical finding even when the code looks clean. The usual root cause is an output-token cap too small for the model's reasoning: the budget is spent on internal reasoning and the final answer comes back at zero characters. Confirm it in the response token counts (completion versus reasoning tokens) instead of assuming it.
**20. Follow the empty response to the end.** An empty answer usually breaks a database constraint such as `CHECK(length(content) > 0)`; the exception leaves the handler, the framework swallows it, and the user's turn ends with nothing delivered. A quality defect becomes a delivery defect.
**21. Measure what the user actually sees, not what the model returns.** If the provider returns reasoning blocks and the code only escapes HTML, the user sees raw reasoning. Measure the ratio of reasoning characters to total characters; above 50% is critical. Also check chunking for empty messages and orphaned chunks, and check that escaping does not break the surrounding quotes.
**22. Audit every entry path separately.** Text, document, voice and command paths can each have their own persona, corpus and rules. If the document path loads a different corpus than the text path, the bot grades against two different standards and the user notices when they switch format.
**23. Measure domain coverage against the primary source.** Extract the official topic list and count how many entries the corpus actually covers. Zero of N means the bot answers from general knowledge, cannot cite where it got an answer, and cannot tell which topics it has failed. Also check what the corpus claims is unavailable — if it says "this is not published yet", verify that it really is not published, because otherwise half the syllabus is excluded.
**24. Verify claims about a catalogue or a regulation against the primary source.** Download the official document, extract the data, and compare it with what the code and the corpus assert. Extract counts and enumerations — how many items, how many sections, which date, which table — because that is where the error hides. The real bug is almost always that the code and the corpus say different things.
**25. A sub-agent's report is a self-declaration.** Verify every finding you are about to publish yourself. If a sub-agent stalls, stop it and deliver what you have verified; a report with confirmed findings beats a complete report half-finished.
**26. Report shape: verdict, criticals with measured numbers, medium findings in a table, what is working, and a phased fix plan.** The "what is working" section is part of the report — it tells the operator what not to touch. Put the phase that makes the bot respond first, then content, then features.

### Schedulers, cron and agent activity

**27. Inventory every scheduler in parallel.** The agent's own jobs, the root crontab, the user crontab, `/etc/cron.d/`, the periodic cron directories, the system timers and the user timers. For each: name, schedule, last and next run, type, one-line purpose. Do not read whole scripts — metadata only.
**28. Reconcile the inventory.** Find duplicates (the same script driven by two schedulers), deaths (an entry whose target script no longer exists), and silent jobs (an active job with no watched log, which fails without a trace).
**29. Build a scheduled job with the real CLI shape, then confirm it exists.** The schedule is positional; passing it as a named flag creates nothing while still printing a success message. The script argument wants a bare filename, not an absolute path. The "no agent" flag makes the script the entire job and needs a script argument. Empty stdout means nothing is delivered, which is exactly the alert-only semantics a watchdog wants. A green CLI message is not proof the job exists — list the jobs afterwards.
**30. Verify a script before wiring it to a schedule.** Run it directly and check the exit code. Watch for `set -u` interacting with positional parameters: a function using `$1` inside a loop or before assignment aborts the whole script.
**31. Do not migrate a working scheduler just to standardise it.** Check what the script needs (bash-only is fine, anything that touches the init system or its cgroups belongs to the host side) and which paths and credentials it touches. Migration that adds a layer is debt. Migrate only jobs whose failure mode is currently silent.
**32. Measure fresh tokens and cached tokens separately.** Usage audits usually report prompt tokens that *include* cache reads. Counting cache as real consumption overstates the problem by roughly an order of magnitude, and cache reads are far cheaper. The actionable metric is fresh tokens per firing, not the total. A watchdog that reads a diff and sends one message should sit in the low hundreds of thousands.
**33. Watch the unit of the timestamps before writing a window filter.** Session timestamps are stored as epoch *seconds*. Comparing them against a millisecond clock makes every window filter return zero, silently, with no error — the report looks empty while the data is correct. The same applies to day keys: a local-time formatter and a UTC formatter disagree across midnight and shift the bucket by a day.
**34. Unclosed scheduled sessions are pure noise and cannot be pruned.** Sessions the scheduler opens and never closes — the process died before finalising — stay open forever: zero messages, no title, zero tokens. The prune command reports that it skips them. Deleting them is destructive and needs explicit approval.
**35. A watchdog that only checks state must never carry an LLM.** A scheduled job with a long prompt and a large accumulated context re-sends the whole context on every firing and the cache is invalidated between runs. Measured: one such job burned 16.9 million prompt tokens in six days, with 3.5 million in a single 16-minute run. A pure-script job with no agent costs zero tokens. If the watchdog needs to reason, make it reason over a small delta, not over the history.
**36. Split large procedure files.** A loaded procedure stays in the context for the rest of the session and is re-paid on every later call. Measured on a real host: 105 loads accounted for 39% of all tool output, and loading one 170 KB file four times in the same session added about 176 000 tokens. Splitting the always-on rules into a hub plus per-topic files cut per-load cost by 73% to 86%.
**37. Latency is driven by generated output, not by context size.** Across 2 603 API calls the correlation between output tokens and latency was strong while the correlation with context tokens was negligible. Lowering the compression threshold does not speed anything up; it only reduces cache-read spend. What helps: fewer tool calls per turn, shorter outputs, and no long serial command queues.
**38. Parallelism is limited by the work style, not by the runtime.** The runtime does execute tool calls from the same turn concurrently, but write-capable tools are not marked parallel-safe and therefore serialise. Measured parallelism per session ranged from 0% to 91% with the same runtime and the same model. The lever is instruction — batch the read-only calls — not configuration.

### Credentials

**39. Search the host before asking the user for a credential.** Search order, broadest and fastest first: project env files and secret directories; user dotfiles and netrc; shell history (a token pasted once for a one-off export often ends up there); the password manager CLI; the code-hosting API; the agent's own memory. Each step takes under two seconds. Asking for a value that is already on disk wastes round-trips and breaks autonomy.
**40. Never read or echo a secret value.** Output paths, lengths, owner and mode, login round-trips and drift. Round-trip a token with one call using a deliberately wrong credential (expect 401 or 403, so the auth path is live) and one call using the correct one read from the environment at runtime (expect 200). Never print the value.
**41. Expected permissions: 600 for user-owned files, 640 for system-wide.** Anything world-readable is a finding. Every running daemon should map to at least one credential file; a daemon with no credential file is either not using credentials or storing them somewhere you have not looked.
**42. Detect drift between duplicates with a hash-only comparison.** Some credentials legitimately live in several places. When one is rotated the others silently drift. Compare lengths and short head/tail fragments, or compare hashes — never the full value — and confirm with a live request: a stale token answers 401, a current one answers 200.
**43. Naming is not consistent across env files.** The agent's env file may call it `HASS_TOKEN` while the project's env file calls it `HA_TOKEN`. Enumerate every `*_TOKEN` name and path in one pass, then probe each candidate against the upstream. Escalate to the user only when every candidate answers 401.
**44. A locked password manager means "cannot audit from this session", not "everything is fine".** The master password cannot be requested and the headless client usually cannot unlock. Say exactly that, give the operator the unlock recipe to run locally, and mark the result unknown.
**45. Report credential findings in three buckets — working, stale or drifted, unknown — and do not propose the fix in the same reply.** The operator decides per case whether to copy a working token over or to rotate and update everywhere. Default to the copy when it is cheaper and no rotation is needed.

### Public exposure

**46. Audit the public surface through the provider's API, not from inside the network.** A request from the host itself resolves hostnames through local DNS rewrites and will happily answer for names that are never public. The provider API is the only source that tells you what is published.
**47. Token scope decides what the audit can see.** A zone-scoped token lists DNS records but cannot read tunnel configuration or access applications; the account-scoped token can. Stopping after DNS and reporting "audit done" is the most common error in this whole area — it misses the applications that are configured but not enforced. Use the account-scoped token for anything beyond DNS.
**48. A configured access application is not an enforced one.** The access application has to be referenced by the tunnel ingress rule for the edge to intercept traffic. The API shows that the application exists; it does not show that it is enforced. The verification is a programmatic request from outside the private network with no cookies: a redirect to the access provider means it is enforced, a 200 means it is not and the request reached the backend — which may now be writable from the public internet. Every public hostname with sensitive endpoints needs this probe.
**49. Cross-check three inventories.** LAN-only names, private-network-only names and internet-exposed names live in three different places — the proxy configuration, the mesh-network status, and the provider's tunnel and DNS records. The same fully qualified name can appear in any combination. The deliverable is a per-hostname table with a column for each network and a column naming the source of each answer. Claiming "X is public" without the cross-check hands the operator half the truth.

### Tooling and reporting traps

**57. An audit tool can lie about the system it inspects.** A data hook that skips the fetch while the browser tab is in the background makes a live panel render an empty skeleton while the API answers 200 with correct data. Check the tab's visibility before debugging the backend.
**58. A linter's findings depend on the interpreter that runs it.** When the YAML parser is missing, the linter falls back to a line-by-line parse and reports phantom findings — a real count of 131 against 310 with the fallback, with hundreds of invented problems about missing metadata and malformed platform lists. Probe the parser before trusting the output; report "the linter is not available here" rather than inventing a hundred findings.
**59. The zero-failure line counts as a line.** The literal "0 loaded units listed." is a line, so a naive line count reports one failure when there are none. Use the no-legend form and count the lines above zero.
**60. An accumulating mirror only drifts in one direction.** A mirror that copies without deleting grows forever, so it normally holds more than the live tree. That difference is expected. The finding is the reverse: something live that the mirror never copied.
**61. Pending package updates are not a coherence finding.** Report them separately from configuration drift, and defer the kernel.
**62. Read the activity database read-only, and only the table you need.** A read-only connection works against a large write-ahead log without blocking the running agent (about 15 ms). The session table is enough for a panorama; the message table is large and its full-text indexes are unnecessary.
**63. Snapshot the inventory before repeating the audit.** A scheduled job that writes the inventory to a cache file means the next audit starts from the previous result instead of rebuilding it.
**64. Split the audit into non-overlapping scopes and measure while they work.** One pass over the code for concurrency, persistence and security; one over the domain for coverage and primary sources; one over the live behaviour for usage, reliability and filtering. Do not wait for the sub-audits to finish before you start measuring yourself, and use their findings to corroborate or contradict yours.

### Autonomy for audits

**65. An audit is autonomous; a fix is not.** Read-only sweeps, probes and reports run and are delivered without asking. Any change the audit suggests is a separate decision the operator makes — including the small ones. When the audit comes back clean and the operator did not ask for improvements, deliver the table and stop.
**66. Do not invent checks the operator did not ask for.** The deliverable is a table of hard checks with verified status, plus what is genuinely imperfect. A list of possible enhancements is noise, and it reopens a decision the operator has already closed.

### Orphaned projects and broken repositories

**50. A project is orphaned only when three conditions hold at once.** No deployment footprint in any service layer, no live reference from anywhere, and no substitute. Add a fourth check: no uncommitted work. Also keep an explicit list of protected infrastructure that is excluded by name even when it has no footprint — an exclusion that is applied silently is not auditable, and the next operator repeats the mistake.
**51. Check every service layer before writing "not deployed".** A static site served by a separate web server, a proxy vhost, a tunnel ingress entry, a compose file and a service unit are five different layers. Judging only on containers and units once deleted a working site that stayed served from a missing directory, and it stayed broken for days with no alert firing. Check the listeners, the proxy vhosts, the tunnel ingress, then the compose files and units.
**52. Know the false positives of a name sweep.** Word matches in unrelated files, projects referenced by name rather than by path, commented-out vhost blocks that are tombstones from earlier retirements, and dependency directories that show up as thousands of deletions. A name match is a candidate, not a verdict.
**53. A broken index is not lost work.** A repository whose index file is missing can report every file as deleted while the files are intact on disk. Diagnose in three steps: check whether the index exists, verify the files and the head commit are on disk, then rebuild the index with a mixed reset, which realigns the index without touching the working tree. Never use the hard reset — it deletes modified files.
**54. Two different failures look the same.** A reset that fails with "cannot read tree" means the objects are missing: real corruption that a reset does not fix. A reset that fails with "index file smaller than expected" means the index is corrupt: remove it and reset. Report them separately.
**55. "The fetch failed" does not mean the remote is broken.** A local repository can fail to fetch while the remote clones cleanly. Verify with a shallow clone into a scratch directory and run the integrity check there. If the clone is fine, the problem is local: clean the repository directory and clone again, or force-fetch after clearing the broken refs.
**56. A repository may have several remotes.** Read the complete remote list before concluding that the history is backed up. The upstream might be one remote and the internal host another.

### Drift and coherence between layers

**67. Read-only coherence is a separate audit from the per-component ones.** Component audits catch failures; the coherence audit catches components that work in isolation but contradict each other. Plan for fifteen to thirty minutes of parallel read-only calls and run the independent steps concurrently.
**68. Three sources must agree about a proxy configuration: the repository, the deployed file and the service catalogue.** A non-empty diff between the repository and the deployed file means the sync path is broken — re-sync by hand. A catalogue entry with no vhost, a vhost with no catalogue entry, and a catalogue that names a different backend port than the vhost are three distinct failure modes, and the third one is a real bug that was live in production.
**69. A grep for host matchers produces false positives.** One matcher line can list several hostnames while the individual handlers live inside a parent block. Parse the file as a state machine: find the matcher, walk forward to its handler block, extract the backend from inside. A naive line grep reports several phantom services and hides the one that actually moved.
**70. A declared-but-inactive matcher is a partial retirement.** A host matcher that is not commented out, with no live handler behind it, still routes traffic — usually to the next handler as a fallback, which returns whatever that block serves. Test each public hostname and read the status code: 200 means the matcher still routes, 502 or a refused connection means it matched a dead backend.
**71. A static site served as plain files has the repository as its deploy artefact.** There is no build step and nothing to regenerate: a clone plus a pull is the deployment. A change to the proxy configuration does not need a rebuild, and a rebuild does not deploy.
**72. Compare the catalogue's ports against the live listeners.** A mismatch is drift. Decide which side is wrong before fixing it — a tombstoned catalogue entry whose backend still runs is expected drift, documented, not a bug to fix.
**73. Check the catalogue repository's own state.** A dirty catalogue file means the running system has the change and the code host does not. A diff on that file tells you whether the change is a retirement marker or a real mutation — confirm the second one before committing it.
**74. A repository can be committed locally and never pushed, and that is a drift category of its own.** Sort the project tree into dirty, ahead, and untracked-upstream. Push the ahead ones that are actively used; ignore the ahead ones that are mirrors of upstream projects; a branch with no upstream configured is cosmetic and not a finding.
**75. The code host may hold repositories that are not on disk, and that is normal.** The operator clones only what they use. The finding is the reverse: a local project whose remote points at a repository the host answers 404 for — the remote was deleted and the local copy is now the only one.
**76. Match the local directory to the remote by the remote URL, not by name.** Renamed projects and legacy branches break name-based matching: a directory whose remote slug differs from its own name, a project whose branch is `master` when you assumed `main`.
**77. A script with no consumer is an orphan, and there are four places to look.** Systemd units, other scripts, cron entries, and running processes. Zero matches across all four means the script is dead. The common causes: the service was retired and the script stayed; the service moved from a unit to manual invocation and the script was never rewired; a one-shot experiment was left behind.
**78. A monitoring check that only tests "is the process running" misses the failures users actually notice.** Three layers per critical service: the process layer (unit active or container running), the wire layer (the port listens, or a request returns a valid status), and the content layer (a sample operation against the data path returns the expected result). The content layer is the one most often missing, and it is the one that catches the bugs that survive a green board: bind divergence, stale cached state, silently broken routing.

### Monitoring and alerting

**79. Audit the monitors, not only the services.** A watchdog that fires every cycle on the same failure trains the operator to ignore it. Findings in this area: a permanent failure with no rate limit (identical entries repeated every cycle — real case: seventeen identical entries in two hours), a threshold baked in as a constant instead of an environment override (device readings vary by ±1 on re-read, so the alert fires every cycle on a stable disk), and a watcher whose escalation path is a destructive action taken automatically.
**80. Never auto-apply a destructive fix from a monitor.** Killing processes, restarting critical services and deleting data all need explicit approval. A monitor may apply a local, reversible fix; anything destructive escalates to a human with a tagged alert.
**81. Rate-limit on state change, not on every cycle.** Persist a consecutive-failure counter per service and alert only when the service is new to the counter or the state flips. Log the repetitions as information. If the notification path itself can be down, fall back to the system log so the failure is at least recorded somewhere.
**82. A watcher cannot drive user-scope units from a system-scope service without an explicit bus.** The environment variables for the user bus are not set in that context and the call always fails. The working form routes to the user manager by identifier instead.
**83. Do not test a monitor's failure path by causing a real failure.** The monitor will fire the real alert for the problem you created. Restore the broken state first, trace the failure-handling block by hand against a synthetic input, and if you must run it end to end, use a dry-run switch that skips the notification call — add that switch the first time you need it, not after you have already notified the operator. Confirm afterwards that no new notification rows were created during the test window.

## Reference Files

- `references/third-party-and-bot-audit.md` — external-host enumeration commands, the privacy manifest field reference, network-behaviour deep read, the five layers of a bot audit with measured commands, and the report templates.
- `references/credential-hygiene.md` — the credential search order, the never-echo round-trip recipe, the drift-detection snippet, the provider-token scope table, and the public-surface audit recipe.
- `references/scheduler-and-cron-audit.md` — the parallel scheduler inventory, the reconciliation checklist, the job-creation CLI shape, the token-accounting queries, and the context-bloat arithmetic.

Load with `skill_view(name='infra-audit-checklist', file_path='references/third-party-and-bot-audit.md')`.
Load with `skill_view(name='infra-audit-checklist', file_path='references/credential-hygiene.md')`.
Load with `skill_view(name='infra-audit-checklist', file_path='references/scheduler-and-cron-audit.md')`.
