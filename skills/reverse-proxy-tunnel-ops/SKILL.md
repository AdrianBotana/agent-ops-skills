---
name: reverse-proxy-tunnel-ops
description: "Use when an agent publishes or audits a service behind Caddy, Cloudflare Tunnel or a DNS-01 challenge: vhost routing, bind addresses, tunnel ingress, Access policies, DNS and CGNAT reachability."
version: 0.1.0
author: Adrián Botana
license: MIT
platforms: [linux]
---

# Reverse Proxy and Tunnel Ops

Rules for putting a service on a hostname and for proving it is reachable
from the right side of the network. The recurring failure in this area is not
a broken proxy — it is a request that never reaches the proxy at all, tested
from the wrong vantage point, with a token that cannot see what you think it
sees.

Every rule here is about one of three questions: does the request reach the
edge, does the edge know where to send it, and does the origin answer.

## When to Use

- Adding or removing a hostname in the proxy or in the tunnel ingress.
- Changing a service's bind address or upstream port.
- Auditing what is exposed to the internet and what is only on the LAN.
- Verifying that an access policy is actually enforced, not just configured.
- Deciding whether an inbound port is reachable from outside at all.
- Diagnosing a vhost that returns 404, 502, or refuses the connection.

Do not use for: application-level bugs behind a working vhost, or general DNS
administration for a domain you do not serve.

## The Rules

### Routing and configuration

**1. There are two configuration files, and only one is the source of truth.**
The repo copy is what gets version-controlled; the deployed copy under the
system config directory is what the process reads. Edit the source, then
deploy it. If you ever edit the deployed copy directly, reformat the source
afterwards and push it, or the next deploy silently reverts you.

**2. The proxy's source-of-truth file may be synced by a path unit that is
not always active.** If the config lives in a repo and a systemd path unit
copies it on change, that unit can be inactive. When the deployed file does
not match your edit, deploy it manually and restart the proxy.

**3. Validate the config before deploying it, with the same environment the
service runs with.** Secrets referenced by the config (an API token for a
DNS-01 challenge, an application credential) live in an environment file, not
in the config. Validating from a shell that has not loaded it produces a false
error — the token reads as empty and the config looks broken when it is not.
Load the environment file first and check the exit code.

**4. Restart the proxy, do not reload it, after changing DNS challenge
settings.** The DNS module caches its token at first load. A reload keeps the
in-memory cache from the previous configuration, so new hostnames do not
activate even though validation passed. A restart clears the cache.

**5. Named matchers are site-level directives; they cannot be nested inside
another directive's block.** Writing a matcher inside a `header` block is a
parse error, and the proxy exits and restart-loops. The shape is: named
matchers at site scope, then the directive that consumes them.

**6. An empty access log for a hostname means the request never reached the
proxy.** The fault is in DNS or in the hostname itself, not in the routing
config. Do not start editing matchers, server names or upstreams until the
request is arriving.

**7. A typo in the hostname looks exactly like a proxy or TLS problem.** The
request leaves to public DNS, which returns NXDOMAIN, and the proxy never sees
it. When the resolver answers but the request fails and the access log is
empty, grep the exact hostname across the proxy config, the application's own
public-URL setting and the application source. One typo in any one of them
breaks the chain.

**8. Handles are evaluated top to bottom; keep them alphabetical.** The proxy
does not care, but reviewers and diffs do.

**9. A retired hostname under a parent matcher still serves traffic.** If the
hostname appears in a parent's host list, requests still match the parent,
fall through every child handle, and get answered by the next matching
handle — which produces a 200 from an unrelated service. To truly retire a
host: remove it from the parent's host list *and* disable the inner handle,
then probe expecting 404, not 200. The same shape applies to any grouped
matcher: check who consumes the matcher, not only who the matcher targets.

**10. Sweep every hostname after any config change.** A per-hostname probe
loop catches silent fall-throughs that a single curl never will: one row
showing 200 on a retired hostname, or 404 on a live one, is the smoking gun.

### Bind addresses and reach paths

**11. A service reached through a proxy vhost must bind loopback.** The proxy
runs on the same host and connects to the upstream over loopback. Binding the
LAN address leaves the port exposed to every device on the network for no
benefit.

**12. A service reached directly by the tunnel must bind the address the
tunnel ingress points at.** The kernel does not redirect a connection aimed at
the host's LAN address to a loopback listener. If the ingress URL is
`http://<host-lan-ip>:<port>` and the service listens only on `127.0.0.1`, the
tunnel returns 502 while every local test against loopback succeeds.

**13. Edit the tunnel ingress before you flip the bind address.** Changing the
ingress first, waiting for the edge to propagate, and then binding to loopback
is the only safe order. Binding first breaks the tunnel for the length of the
propagation gap, with no automatic recovery. Keep a rollback plan.

**14. A hostname must never be served by both the proxy and the tunnel.** The
result is ghost 502s, two certificates for the same name in the transparency
log, and rate-limit trips on the certificate authority. Audit before adding a
vhost: if the name appears in the proxy config and in the tunnel ingress, one
of them has to go.

**15. Smoke-test tunneled hostnames from outside the LAN.** A local DNS
rewrite that maps every internal name to the proxy's LAN address turns a
LAN-side smoke test into a false positive. Test from a different network.

### Testing from the right vantage point

**16. A WAN test from the host itself is a false positive if it uses the local
resolver.** Local DNS rewrites resolve every internal hostname to the LAN
address, so a request returns 200 even for hostnames that are not published at
all. For a real exposure audit: resolve with an external resolver first. If
the name does not exist publicly, the audit stops there. If it resolves to a
CDN address, then pin the request to that address with `--resolve` and check
the backend response.

**17. The proxy binds to the LAN address, not to loopback.** A local vhost
check must therefore resolve to the LAN address. Using `--resolve ...:127.0.0.1`
returns connection refused and is not a service-down signal. Check the actual
listeners with `ss -tlnp` before interpreting a refused connection.

**18. Test the backend directly when the proxy is the suspect.** If the
backend answers on loopback and the vhost does not, the problem is in the
proxy path — certificate, matcher or upstream. If both fail, the service
itself is down.

**19. Test the backend's real data path, not just its root.** A 200 on `/`
proves the process answers, not that the feature works. For file-serving
services, compare a file count on the host against the count inside the
container or behind the proxy.

### Credentials and secrets at the edge

**20. A credential injected by the proxy makes the endpoint unauthenticated to
everyone who can reach the vhost.** The proxy can add an `Authorization`
header so the browser never has to hold a key. That is a real pattern with a
real consequence: if the vhost is LAN-only the exposure is the LAN; if it is
public it is the internet. Report the consequence. If the key must stay
mandatory, use browser basic authentication instead.

**21. `header_up` is a sub-directive of the reverse proxy.** Written at the
top level it aborts the config with an unrecognized-directive error.

**22. Use load-time substitution for a mandatory credential.** Load-time
substitution fails the config when the variable is missing, so the vhost does
not come up unauthenticated. Runtime substitution silently yields an empty
value. For a credential that is required, prefer failing loudly.

### Cloudflare API and access policies

**23. Token types are not interchangeable.** A user-scoped token works for
DNS record editing and returns 403 on account-level endpoints — access
applications, tunnel configurations, most account routes. A "token is valid"
check passes while the scope matrix excludes the call. Confirm scope mismatch
from the response, not from a guess.

**24. The tunnel ingress configuration lives in the provider's API, not on
disk.** For tunnels whose config source is the provider, the ingress rules
exist only in the account API. The local agent config file is a stub. The
verified endpoint is `/accounts/<account-id>/cfd_tunnel/<tunnel-id>/configurations`;
similar-looking paths with a different plural form return 404. A PUT to the
same URL replaces the ingress atomically.

**25. An access application that exists is not an access application that
enforces.** The policy has to be attached where the traffic actually flows.
Two fields silently do nothing: the policy must be an allow decision, and the
included identities must match the identities actually in use.

**26. The only trustworthy enforcement check is an unauthenticated probe.**
Send a request from outside the private network with no cookies, against a
sensitive endpoint. A redirect to the access provider, or a `cf-mitigated`
header, means it is enforced. A 200 means it is not: the request reached the
origin, and the origin may now be writable from the public internet. Cloud
provider documentation is thin here; trust the empirical probe.

**27. An access policy in the wrong scope does nothing.** Account-level versus
application-level scope matters, and so does whether the tunnel ingress is the
public ingress for that hostname. Verify with the probe from rule 26 after
every policy change.

**28. Cross-check three inventories before claiming a hostname is public.**
LAN-only names live in the proxy config, private-network names show up in the
mesh network's status output, and internet-exposed names live in the tunnel
ingress and the DNS records. The same fully qualified name can appear in any
combination. The deliverable is a per-hostname table with a column for each
network and a column naming where each answer came from.

**29. When the API path is blocked, the operator's pasted dashboard table is
the source of truth, not a fallback.** Parse it into a structured list and
update the canonical inventory with it. Do not keep both the pasted text and
the table.

### Public address and reachability

**30. A tunnel is a client, not a server, so HTTP works behind CGNAT and
UDP does not.** Tunnel-based HTTP and WebSocket services work from behind
carrier-grade NAT. UDP-raw services — game servers, voice, mesh VPN inbound,
funnel — need inbound ports that CGNAT denies. Do not publish a tunnel and
call it "exposed to the internet" for a UDP service.

**31. Detect CGNAT with three independent probes, and require all three to
agree.** The probes are: the router's WAN address against the address the
internet sees; a traceroute showing a run of carrier-private hops between the
router and the first public hop; and the router's own external-IP answer over
UPnP. A single probe can lie.

**32. The UPnP external-IP call has two footguns.** The device description
lists several control URLs, and the useful one — the WAN IP connection — is
typically the last of them, not the first; picking the first returns an
authorization error and an empty address even on a router with a valid public
IP. And the control URL path changes when the router re-provisions itself, so
re-run the discovery on every invocation instead of hardcoding the path.

**33. Validate the NAT-PMP opcode before parsing the reply.** Routers that do
not implement NAT-PMP sometimes emit a 12-byte UDP reply whose first byte is
zero. Parsing that as an address produces a plausible-looking public IP and
can report a CGNAT escape that never happened. Only a success opcode counts.

**34. Confirm a CGNAT escape positively.** The router's reported WAN address
must match the address the rest of the internet sees, exactly. The traceroute
must no longer show carrier-private hops, the UPnP answer must be a
non-private address equal to the internet-visible one, and an external TCP
connect to the forwarded port must report it open. All three must agree —
the detection rule is symmetric, because a positive detection can lie too.

**35. The cheapest escape is a carrier add-on, not a VPS.** Residential
providers commonly offer a public IPv4 for a small monthly fee, and some
fiber providers offer a fixed public address free of charge. Ask the operator
to check the provider's own self-care portal. Prices and procedures change:
verify before quoting.

**36. After the carrier re-provisions the line, the router must be
power-cycled.** The change is applied at the DHCP level, so a configuration
reload leaves the old WAN state in place. A power cycle, not a service
restart.

**37. Fail fast on the escape check instead of polling for hours.** Compare
the public address with the previous one, trace the path, and query the
router's external-IP answer. If the address has not changed after the
provider's stated provisioning window, tell the operator to escalate the
request: provisioning can fail silently on lines with pinned leases.

### Identity of hosts

**38. The MAC OUI identifies the manufacturer; the TTL and the open ports do
not.** A TTL of 128 covers Windows, macOS and many BSDs — it is not an
Apple-specific signal. The first three bytes of the MAC address, read from
the neighbour table after a single ping, give the vendor. Getting the identity
wrong propagates into every hostname, vhost and catalogue entry that names
the machine, and the rename has to be redone everywhere.

**39. Read the OUI before you name the host.** When there is no neighbour
entry because the machine is off or just connected, a host sweep or a manual
ping populates it.

**40. Check where your shell already is before you SSH to another machine.**
If the shell is already running on the target — the user name matches, the
host name matches, the connection environment shows an incoming session — then
an SSH hop to that same machine is wasted and often broken. Talk to it
directly. When the shell is genuinely elsewhere, accept the new host key on
first contact so the prompt does not abort the command.

## Reference Files

- `references/caddy-and-https.md` — Caddyfile structure, env-file validation,
  the restart-versus-reload rule, matcher placement, vhost retirement and the
  credential-injection pattern.
- `references/cloudflare-tunnel-and-access.md` — token scopes, the tunnel
  ingress endpoints, access policy enforcement, the four reach patterns and
  their bind rules, and the public-surface audit.
- `references/dns-and-cgnat.md` — DNS-01 challenges, the WAN-test vantage
  point, the three CGNAT probes with their footguns, and the carrier escape
  procedures.

Load with `skill_view(name='reverse-proxy-tunnel-ops', file_path='references/caddy-and-https.md')`.
Load with `skill_view(name='reverse-proxy-tunnel-ops', file_path='references/cloudflare-tunnel-and-access.md')`.
Load with `skill_view(name='reverse-proxy-tunnel-ops', file_path='references/dns-and-cgnat.md')`.
