---
name: cloudflare-tunnel-and-access
description: "Token scopes, tunnel ingress endpoints, access policy enforcement, the four reach patterns with their bind rules, and the public-surface audit."
---

# Cloudflare Tunnel and Access

The tunnel edge and the proxy are two separate ingress systems on the same host. Most incidents come from a hostname existing in both, a token that cannot read the endpoints the audit needs, or an access policy that exists in the dashboard but never intercepts traffic.

## Token scopes

Two token families exist and they are **not** interchangeable:

| Token | Prefix | Scope | Reaches |
|---|---|---|---|
| User service token | `cfut_` | Zone read, DNS record edit | `/zones`, `/user`, DNS record editing |
| Account API token | `cfat_` | Account-level resources | `/accounts/...`, access applications, tunnel configurations |

A `cfut_` token returns **403** on every `/access/apps` and `/cfd_tunnel/<id>/configurations` call. A "token is valid" check passes because `/zones` and `/user` answer 200 — the scope matrix excludes the rest. Confirm the scope mismatch from the response, not from a guess.

For any audit beyond DNS records, use the account token. Search the host before asking the operator for one:

```bash
grep -rIEh 'cfat_[A-Za-z0-9_-]{20,}' /home /etc /opt 2>/dev/null \
  | sed -E 's/.*(cfat_[A-Za-z0-9_-]+).*/\1/' | sort -u
```

Persist the chosen token in the proxy's environment file (mode 0640, owner `root:<proxy-group>`) or a separate audit-only environment file, so the audit script does not have to re-read shell history.

## Tunnel ingress configuration lives in the API

For tunnels created from the dashboard, the config source is the provider, and the ingress rules exist **only** in the account API. The local agent config file on disk is a stub, and reading it gives a false inventory.

Verified endpoint: `GET /accounts/<account-id>/cfd_tunnel/<tunnel-id>/configurations`, returning `{config: {ingress: [...]}, tunnel_id, version}`. A `PUT` to the same URL with `{config: {ingress: [...]}}` replaces the ingress atomically.

Endpoints that look similar and return **404**: `/tunnels/<id>/configurations` and `/cfd_tunnels/<id>/configurations` (plural on `cfd_tunnels`). The warp-routing block lives in the same `config` object.

Required scopes: `Account: Cloudflare Tunnel: Edit` for the ingress PUT, `Account: Zero Trust: Edit` for access applications.

## An access application that exists is not one that enforces

An application created in the dashboard with auto-redirect disabled and eager cookie setting enabled, whose domain is not referenced by the tunnel ingress, is visually configured and **inert**. Requests reach the origin without a challenge.

The ingress rule has no field that names the access application. Setting such a field is silently ignored and the bypass remains. Real enforcement flows through access policies attached to the application's domain; the ingress only provides reach.

To close a bypass: the policy decision must be `allow` (not `deny`, not `none`); the `include:` clause must match identities actually in use; and the policy must be attached at the scope where the traffic flows (account vs application).

**The verification that distinguishes configured from enforced is an empirical probe.** Send an unauthenticated request from outside the private network against a sensitive endpoint:

```bash
curl -X POST https://<app-host>/<sensitive-endpoint> \
  -d '<anything>' -H 'content-type: application/json' -i
```

A 302 to the access provider, or a `cf-mitigated` header, means **enforced**. A 200 from the origin means **not enforced**, and the request reached a backend that may now be writable from the public internet. The provider's documentation is thin on this: trust the probe.

## The four reach patterns and their bind rules

| # | Pattern | Reach path | Bind rule |
|---|---|---|---|
| 1 | Proxy vhost | Tunnel → proxy:443 → backend on `127.0.0.1:<port>` | Backend binds `127.0.0.1` |
| 2 | Tunnel direct to the host address | Tunnel → `http://<host-lan-ip>:<port>` | Backend binds the **host LAN address**, not loopback |
| 3 | Tunnel direct to loopback | Tunnel → `http://127.0.0.1:<port>` | Backend binds `127.0.0.1`; requires a tunnel ingress edit |
| 4 | Wildcard apex abort | The apex does not resolve through the proxy at all | Backend binds the host LAN address |

**The self-connection trap.** The kernel does **not** redirect a TCP connection aimed at the host's own LAN address to a loopback listener, even though the route lookup shows the loopback device. Verified empirically: a listener bound to `127.0.0.1:<port>` accepts a request to `127.0.0.1:<port>` and refuses a request to the host's LAN address from the same host.

Consequence: pattern 2 with a loopback bind produces a silent 502 from the tunnel while every local loopback test passes. Pattern 2 requires the host LAN address bind. Pattern 3 is the only way to remove LAN exposure entirely for a direct service, and it requires editing the tunnel ingress.

**Never both.** A hostname in the proxy config *and* in the tunnel ingress produces ghost 502s, two certificates for the same name in the transparency log, and certificate-authority rate-limit trips. Audit before adding a vhost.

## Verify-after-bind-change order

1. Edit the unit or compose file to the new bind address.
2. Reload the daemon and restart the service (or `docker compose up -d`).
3. `ss -tlnpH sport = :<port>` must show the intended local address.
4. **If the service is behind a proxy vhost:** update the proxy config and repo to the new upstream, restart the proxy. Smoke test LAN and WAN.
5. **If the service is tunnel-direct:** also update the tunnel ingress URL to `http://127.0.0.1:<port>` **before** the bind change. Order matters — ingress first, wait for edge propagation, then bind. Binding first breaks the tunnel during the gap. Always have a rollback plan.
6. After every change, smoke the hostname from outside the LAN. A local DNS rewrite that maps internal names to the LAN address makes LAN-side smoke tests meaningless for tunnel verification.

## The public-surface audit

Four API calls, in this order, with the account token:

```bash
# 1. Proxied DNS CNAMEs — the published list
curl -s -H "Authorization: Bearer <token>" \
  "https://api.cloudflare.com/client/v4/zones/<zone-id>/dns_records?type=CNAME&per_page=100" \
  | jq -r '.result[] | "\(.name) -> \(.content) proxied=\(.proxied)"'

# 2. Tunnel ingress rules
curl -s -H "Authorization: Bearer <token>" \
  "https://api.cloudflare.com/client/v4/accounts/<account-id>/cfd_tunnel/<tunnel-id>/configurations" \
  | jq '.result.config.ingress[] | {hostname, service}'

# 3. Access applications
curl -s -H "Authorization: Bearer <token>" \
  "https://api.cloudflare.com/client/v4/accounts/<account-id>/access/apps" \
  | jq '.result[] | {name, domain, policies: [.policy[].name]}'
```

Differences to investigate: an ingress rule with no matching DNS record is a zombie tunnel; a DNS record with no matching ingress rule is orphaned DNS; an access application whose domain is not referenced by any ingress rule is cosmetically configured, not enforced. **The audit must end with the bypass probe** for every public hostname with sensitive backend endpoints — the API call alone is not evidence of enforcement.

## Cross-check three inventories

LAN-only hostnames live in the proxy config and resolve through a local DNS rewrite. Private-network hostnames appear in the mesh network's status output as raw addresses with no DNS. Internet-exposed hostnames live in the tunnel ingress and the DNS records. The same fully qualified name can appear in any combination. The deliverable is a per-hostname table with three columns (LAN, private network, internet) and a column naming the source of each answer. Claiming "X is public" without the cross-check hands the operator half the truth.

**When the API path is blocked and the operator pastes the dashboard table as text, that text IS the source of truth, not a fallback.** Parse it into a structured list, diff it against the previous canonical list, and update the inventory with it. Do not keep the pasted text as a separate artifact.
