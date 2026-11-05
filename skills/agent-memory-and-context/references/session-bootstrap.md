---
name: session-bootstrap
description: "Trigger families, parallel probe routine, auto-write rule and citation format for a proactive session bootstrap."
---

# Session bootstrap

A bootstrap is a **proactive** context load. It runs before the first answer of a turn, in silence, when the request implies prior context.

## Trigger families

### A. Infrastructure

Fires when the request contains any of:

- A host or machine name, an operating system name or version.
- A storage path, a project directory, a private address, a mesh-network name.
- A service, container, repository, proxy, tunnel or domain name.
- Any project slug the operator owns.

Typical wording: "the server", "the host", "the box", "the cluster", "the dashboard", "the tunnel".

### B. Biography

Fires when the user states, or is about to be asked about:

- Employer, role, craft, years of experience.
- Residence, city, household, family, age.
- Hobbies, ongoing projects, colleagues, school.

Also fires on prose that asks the agent to summarise what it knows about the user ("what do you know about me", "remind me who I am").

### C. Meta

Fires when the agent is about to answer a personal question and its memory of the user is thin.

## Probe routine

Run it; do not narrate it.

1. **Probe entities in parallel**, based on which trigger fired.
   - Infrastructure trigger: the infrastructure namespaces (host, services, network, repository host, tunnel) plus any noun taken from the message.
   - Biography trigger: the biography namespaces (identity, work, place, family) plus any name, city or role in the message.
2. **Search by message keywords.** Extract the nouns and query the store once per keyword. Skip any keyword the preloaded block already answers.
3. **Load only the procedures the probed entities imply.** An infrastructure probe loads the administration and orchestration procedures. A biography probe loads nothing but the store queries.
4. **Do it in parallel.** One round of concurrent probes, not a sequential walk.

## Auto-write rule

When the user shares a new durable fact about themselves — name, role, employer, residence, family, hobby, colleague, school — **write it without asking for confirmation**. The store extracts facts on its own as well, so this explicit write is the reliable path for the important ones.

Never copy the fact into the always-on profile file.

## A correction is a new write, not an edit

When the user corrects a fact ("I live in X, not in Y"):

1. Update every layer that carries the wrong value, in the same pass — the store entry and any document page.
2. Then add the corrected value as its own fact, so a direct keyword lookup finds it.

Editing only the entry the user happened to quote leaves the stale copy in the layer that gets read most.

## Never infer an attribute from a keyword

A place name inside a hobby fact is usually the project's location — a marina, a workshop, a route — not the user's home. When a biography term is ambiguous across contexts, ask which one it is instead of picking one and propagating it into always-on memory.

## Citation format

When a reply states a probed fact, cite it with its identifier and its trust score:

```
[fact_id 12, trust 0.5]
```

This lets the operator audit where the claim came from and lets the trust score be tuned over time.

## Anti-patterns

- Do not probe on creative or research work unrelated to the operator's infrastructure or biography.
- Do not narrate the probe — it is a silent warm-up.
- Do not probe twice for the same entity.
- Do not load procedures the probed entities do not imply.

## Failure mode this closes

Without an auto-write rule, biographical facts only land in memory when the user explicitly asks for it to be remembered. "What do you know about me" then becomes a recurring manual bug: the user repeats themselves every few months. The auto-write rule closes the loop.

## Retired approaches

Do not revive a capture path that depends on a module the runtime no longer exposes as importable. A cron-based ambient capture built on such a module was retired for exactly this reason; capture now happens through the store's own extraction plus the explicit auto-write above.
