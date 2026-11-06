# Agent Ops Skills

Production runbooks for AI agents that operate real infrastructure.

These skills encode what actually breaks when an agent changes a running
host: the firewall flush that takes Docker NAT down, the systemd drop-in
that only `systemctl show` can see, the service that is three places
instead of one, the backup that was never restored.

They are not tutorials. Every rule states the root cause, the exact symptom,
the command that proves it, and what not to do — because each one is a
failure that already happened on a real machine.

## Install

```bash
npx skills add AdrianBotana/agent-ops-skills
```

Or a single skill:

```bash
npx skills add AdrianBotana/agent-ops-skills --skill server-ops-safety
```

## What's in it

| Skill | Use when |
|---|---|
| `server-ops-safety` | Changing a running Linux host: Docker networking, firewall rules, systemd units, ports, backup timers, credential files |
| `reverse-proxy-tunnel-ops` | Publishing or auditing a service behind Caddy, Cloudflare Tunnel, Access or DNS |
| `incident-triage` | A host, container or service is failing, or you are diagnosing a post-outage state |
| `infra-audit-checklist` | Auditing an environment before touching it: third-party software, bots, cron jobs, credentials, public surface |
| `service-retirement` | Retiring a service, project, unit, container or hostname without breaking what depends on it |
| `agent-memory-and-context` | An agent must remember facts across sessions without bloating or losing them |
| `watchdog-operations` | Writing, reviewing or debugging a watchdog, health-check timer or retry loop that restarts services |

Seven skills, 301 rules, plus 19 reference files that load on demand.

## Who it is for

Anyone running Claude Code, OpenClaw, Codex or a similar agent against a
machine that matters: a home server, a VPS, a NAS, a CI runner. The rules are
cross-industry on purpose — they apply to any host, not to one particular
setup.

## Format

Standard [Agent Skills](https://agentskills.io) layout: `SKILL.md` with
YAML frontmatter (`name`, `description`) plus `references/` for the detail
that should load on demand.

Each skill's `description` starts with `Use when ...` so the agent knows
when to load it.

## License

MIT.
