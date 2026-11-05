---
name: dns-and-cgnat
description: "DNS-01 challenges, the correct vantage point for WAN tests, the three CGNAT probes with their footguns, and the carrier escape procedures."
---

# DNS and CGNAT

Two questions live here: is the hostname actually published, and can an inbound port reach this host at all. Both are routinely answered wrong from the wrong vantage point.

## DNS-01 challenges

For a wildcard hostname, the certificate is issued with a DNS-01 challenge: the proxy writes a TXT record through the provider's API, then polls until it appears.

- The API token lives in an **environment file** referenced by the unit, not in the config. Load it before validating or the token reads as empty.
- The DNS module caches the token at first load. Restart, do not reload.
- The challenge takes 30-60 seconds. Do not restart the proxy repeatedly while waiting for one challenge — you will start a new one each time.

## Test from the right vantage point

A WAN test run from the host itself is a **false positive** when it uses the local resolver. Local DNS rewrites map every internal hostname to the proxy's LAN address, so a request returns 200 for a hostname that is not published at all.

```bash
# 1. Resolve with an external resolver first
dig +short <host> @1.1.1.1
# NXDOMAIN -> the hostname is not published. The audit stops here.

# 2. If it resolves to a CDN address, pin the request to that address
curl --resolve "<host>:443:<cdn-ip>" -o /dev/null -w 'HTTP=%{http_code}\n' https://<host>/
```

The proxy binds to the LAN address, not to loopback, so a local vhost check must pin the LAN address. A loopback pin returns connection refused, which is not a service-down signal.

## CGNAT detection: three probes, all must agree

Tunnel-based HTTP and WebSocket services work fine behind carrier-grade NAT, because the tunnel agent is a **client**. UDP-raw services — game servers, voice, mesh VPN inbound, tunnel funnels — need inbound ports that CGNAT denies. Before promising to expose any non-HTTP service, run all three probes. Each catches a failure the others miss.

**Probe 1 — router WAN address vs the address the internet sees.** Read the router's WAN address from its admin UI and compare it against the address an external service reports. Beware: a local "what is my IP" service queried from the LAN usually returns the NAT egress address, not the carrier's external address. Query from a different network. The same value twice means there is no CGNAT in this section.

**Probe 2 — traceroute to a public address.** `traceroute -n -w 1 -q 1 -m 6 1.1.1.1`. Carrier-grade NAT always shows up as a run of RFC1918 or RFC6598 hops between the router and the first public hop. Common private ranges on residential networks: `10.0.0.0/8`, `172.16.0.0/12`, `100.64.0.0/10`. If hops 2-5 are all private, the path traverses carrier NAT and **port forwarding on the LAN router cannot reach the internet, period**.

**Probe 3 — the router's own external-IP answer over UPnP.** Send the SOAP body to the router's `WANIPConnection:1` control URL. A correct router returns the assigned IPv4 in `<NewExternalIPAddress>`. An empty string, `0.0.0.0` or a private address means the router is telling you it has no public address of its own, because the carrier is NATing it.

*Footgun A — pick the LAST control URL.* The device description lists several control URL siblings, typically a layer-3-forwarding one and a WAN-IP-connection one. Picking the first returns an authorization error and an empty address even on a router with a valid public IP. Always select the last `<controlURL>` in the description XML.

*Footgun B — never hardcode the control path.* The path changes when the router re-provisions itself after a firmware update or a carrier reconnection. Re-run the SSDP discovery on every invocation and re-parse the description for the device location and the last control URL.

**NAT-PMP opcode validation.** Routers that do not implement NAT-PMP sometimes answer on UDP/5351 with a 12-byte reply whose first byte is `0x00` — glibc noise, or a stray UPnP packet leaking back on the same socket. **Do not parse that as an address.** The only valid NAT-PMP reply starts with `0x80` (result code: success). Drop everything else and rely on the UPnP probe. This is not a theoretical guard: a script once parsed `0x00 0x53 0x4e 0x4d 0x50` as a public IP and nearly reported a CGNAT escape that had not happened.

## Confirming an escape, positively

When all three probes agree on CGNAT, say it plainly: port forwarding on the router cannot work for UDP-raw inbound. Then offer the real options: a tunnel-based or mesh peer-to-peer direct path for UDP where it exists; a relay service with a UDP-capable tier (check that the paid tier actually includes UDP — free tiers commonly disable it); a small VPS acting as a jump host; or a public address from the carrier.

**Do not** spin up a tunnel and call it "exposed to the internet" for a UDP-raw service. Tunnels do not proxy UDP.

Whichever path the operator takes, the only way to claim the escape worked is to compare the router's reported WAN address against the address the rest of the internet sees — **the two must match exactly**. A successful escape also shows up as: the traceroute no longer has carrier-private hops between the router and the first public hop; the UPnP answer is a non-empty, non-private address equal to the internet-visible one; and an external TCP connect to the forwarded port reports it open. All three must agree. The rule is symmetric: a positive detection can lie too.

## Carrier escape procedures

Prices and procedures change. Verify against the provider's own self-care portal before quoting anything to the operator.

**Public IPv4 add-on.** The cheap pattern is universal across residential providers: a small monthly fee unlocks a public IPv4 on the WAN side of the router, instead of renting a whole VPS. From there, UPnP or manual port forwarding work normally. What it does **not** do: change the line speed, provide a static IPv6, or give you a domain. It gives exactly one public IPv4.

**Procedure.** Find the add-on in the provider's self-care portal, under the line's additional-services section. If it is not offered there, the provider's customer service line provisions it after an identity check; expect a stated delay of hours to a day. Then **power-cycle the user-side router**: the change is applied at the DHCP level, a configuration reload leaves the old WAN state in place, and only a power cycle picks it up. Verify with the confirmation probes above — expect the public address to change out of the carrier's private range.

**Fast-fail diagnostic.** When the operator says "I already requested it, is it active yet": (1) compare the public address with the previous one — if it has not changed, the service is not active yet, and provisioning windows run from hours to a day; (2) trace the route — if hops 2-5 still show carrier-private ranges, the router is still routed through carrier NAT, so power-cycle it; (3) query the router's external-IP answer — if it still returns a private address or `0.0.0.0`, the WAN side has not re-DHCP'd, so power-cycle rather than reload the configuration; (4) stop polling. After the provider's stated window with no change, tell the operator to escalate the request — provisioning can fail silently on lines with pinned leases or prior static routes from the provider.
