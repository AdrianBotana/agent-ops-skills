---
name: credential-hygiene
description: "Search order, never-echo round trips, drift detection, provider token scopes and the public-surface audit recipe."
---

# Credential hygiene and public surface

## Never read or echo a secret value
Output paths, lengths, owner and mode, login round-trips and drift. Nothing else.

## Search order before asking the user
Broadest and fastest first; each step is under two seconds.

1. **Project files** — `.env`, `*.env`, `secrets/*`, `config/*.env`, the application config directory. Read the ignore file first to know which paths are meant to be secret.
2. **User dotfiles** — the shell rc files, `.netrc`, `.git-credentials`.
3. **Shell history** — a token pasted once for a one-off export often ends up here and never reaches an env file. Match narrowly on the provider's known token prefix.
4. **Password manager CLI** — list items by metadata when unlocked; a headless session usually cannot unlock it.
5. **Code-hosting API** and **the agent's own memory** — repository descriptions occasionally mention token identifiers; memory holds prior references.

Ask only after the search comes up empty, and save the answer so you do not ask again.

**Must ask anyway:** master passwords, device passcodes, two-factor codes, one-time tokens — the operator reads them from their authenticator, never pasted into a chat.

**Must not ask:** anything found above (use it and cite the path so it can be audited), anything the agent's memory already holds, anything derivable from something you already have (a key identifier encoded in a filename, a team identifier in a certificate subject).

## Round-trip verification without exposing the value
- **API tokens** — one call with a deliberately wrong credential (expect 401/403, so the auth path is live), then one call with the correct token read from the environment at runtime (expect 200).
- **Basic-auth services** — the daemon logs successful and failed logins; check the log for recent successful logins from the expected client.
- **OAuth and session-cookie services** — skip the round trip; document only that the service is reachable.

## Permissions
User-owned credential files: 600, system-wide: 640; anything world-readable is a finding. Every running daemon should map to at least one credential file — a daemon with none is either not using credentials or storing them where you have not looked.

## Drift detection between duplicates
Some credentials live in several places by design. When one is rotated the others drift silently. Compare without printing:

```bash
H1=$(awk -F= '/^HASS_TOKEN=/{print $2}' ~/.agent/.env)
H2=$(awk -F= '/^HA_TOKEN=/{print $2}'  ~/.config/<app>/<app>.env)
H3=$(sudo -n grep '^HA_TOKEN=' /etc/<service>/config.env | cut -d= -f2-)
echo "agent len=${#H1} head=${H1:0:8}...tail=${H1: -8}"
echo "app   len=${#H2} head=${H2:0:8}...tail=${H2: -8}"
echo "svc   len=${#H3} head=${H3:0:8}...tail=${H3: -8}"
[ "$H1" = "$H2" ] && echo "agent==app" || echo "agent!=app"
```

Confirm with a live request: a stale token answers 401, a current one answers 200.

## Naming is not consistent
The agent's env file may call it `HASS_TOKEN` while the project's env file calls it `HA_TOKEN`. Enumerate all of them in one pass:

```bash
grep -rEn "^[A-Z_]*_TOKEN(=| )" ~/.agent/.env ~/.local/bin/*.env \
  /etc/<service>*/config.env <projects-dir>/*/.env 2>/dev/null \
  | awk -F= '{print $1}' | sort -u
```

Then probe each candidate against the upstream. Escalate only when every candidate answers 401.

## A locked password manager is an unknown, not a pass
The headless client reports "enabled but locked" and the unlock prompt needs an interactive terminal. Report exactly that, give the operator the local unlock recipe, and mark the result unknown — never claim it is fine.

## Report format
Three buckets, no fixes in the same reply: **working**, **stale or drifted**, **unknown (locked)**. The operator decides per case whether to copy a working token over or to rotate everywhere; default to the copy when it is cheaper and no rotation is needed.

## Public-surface audit: the provider API, never the local network
A request from inside the network resolves names through local DNS rewrites and will answer for hostnames that were never published. Only the provider API knows what is actually public.

### Four API calls, in order
1. **Account-scoped token.** A zone-scoped token can list DNS records but cannot read the tunnel configuration or the access applications — it answers 403 and 404 on those endpoints. With only a zone-scoped token, stop after DNS and say the audit is incomplete.
2. **DNS records** — the CNAMEs with proxying enabled in the zone.
3. **Tunnel ingress rules** — the hostname-to-service mapping.
4. **Access applications** — configured, with their policies.

```bash
curl -s -H "Authorization: Bearer <token>" \
  "https://api.cloudflare.com/client/v4/zones/<zone-id>/dns_records?type=CNAME&per_page=100" \
  | jq -r '.result[] | "\(.name) -> \(.content) proxied=\(.proxied)"'
curl -s -H "Authorization: Bearer <token>" \
  "https://api.cloudflare.com/client/v4/accounts/<account-id>/cfd_tunnel/<tunnel-id>/configurations" \
  | jq '.result.config.ingress[] | {hostname, service}'
curl -s -H "Authorization: Bearer <token>" \
  "https://api.cloudflare.com/client/v4/accounts/<account-id>/access/apps" \
  | jq '.result[] | {name, domain, policies: [.policy[].name]}'
```

**What to look for:** an ingress rule with no matching DNS record (a zombie tunnel); a DNS record with no matching ingress rule (orphaned DNS); an access application whose domain is not referenced by any ingress rule (configured but not enforced).

### The enforcement probe
For every public hostname with sensitive endpoints, send one unauthenticated request from outside the private network:

```bash
curl -X POST -s -o /dev/null -w '%{http_code}\n' https://<host>/api/<sensitive>
```

A 302 to the access provider, or a body containing the interception marker, means access is enforced. A 200 means it is **not** enforced and the request reached the backend, which may now be writable from the public internet. The API call alone does not prove enforcement.

### Three inventories to cross-check

| Network | Source of truth |
|---|---|
| LAN only | the proxy configuration in the repository |
| Private network only | mesh-network status output (raw addresses, no DNS) |
| Internet exposed | the provider's tunnel ingress plus DNS records |

Deliverable: one row per hostname, one column per network, one column naming the source of each answer. The same name can appear in any combination, so claiming "X is public" without the cross-check hands over half the truth.

### When the API is unavailable
If the operator pastes the dashboard table as text, that text **is** the source of truth, not a fallback: the dashboard row is the live configuration. Parse it into a structured list, diff it against the previous canonical list, update the stored table, and do not keep the pasted text as a separate transcript.
