---
name: caddy-and-https
description: "Caddyfile structure, env-file validation, restart versus reload, matcher placement, vhost retirement and credential injection at the proxy."
---

# Caddy and HTTPS

A reverse proxy is the single point where most "the site is down" reports land. The rules here separate the four causes: the request never arrived, the proxy rejected it, the proxy routed it wrong, or the origin is down.

## Two files

The **repo copy** is the source of truth and version-controlled. The **deployed copy** under the system config directory is what the process reads. Always edit the source and let the deploy pipeline copy it. If you must edit the deployed copy, reformat the source afterwards and push it, or the next deploy reverts you.

The deploy may be driven by a systemd path unit watching the repo, and that unit is sometimes inactive. After editing the source, check the deployed file actually changed. If it did not, copy it manually and restart the proxy.

## Validate with the service's environment

Secrets referenced by the config live in an environment file read by the systemd unit, not in the config itself. Validating from a shell without that environment produces this false error:

```
loading module 'cloudflare': API token '' appears invalid;
ensure it's correctly entered and not wrapped in braces nor quotes
```

The token is fine; the proxy just cannot see it from your shell. Always pre-flight with the environment loaded:

```bash
sudo -E sh -c 'set -a; . /etc/caddy/cloudflare.env; set +a; caddy validate --config /etc/caddy/Caddyfile'
```

Trust the exit code. Some Caddy builds have no `--quiet` flag; passing it fails with an unknown-flag error.

## Restart, not reload

The DNS module caches its token at first load. `systemctl reload caddy` keeps the in-memory cache from the previous configuration, so new hostnames do not activate even though validation passed. Use `systemctl restart caddy` to clear the cache after any change involving DNS challenge settings.

Restarting also costs 30-60 seconds while the DNS-01 challenge runs. If you only need to confirm a hostname exists, grep the config instead of restarting.

## Matcher placement

A named matcher is a site-level directive, not a sub-directive of `header`. Writing it inside a `header` block is a parse error, the process exits, and the unit restart-loops with `adapting config using caddyfile: Unexpected next token after '{' on same line`.

```caddy
:80 {
    file_server
    header /sw.js { Cache-Control "no-cache" }   # inline path argument: works
    @manifest path *.webmanifest                  # named matcher: site scope only
    header @manifest Content-Type "application/manifest+json"
}
```

The correct shape: named matchers at site scope, then the directive that consumes them.

## Diagnosing "the new vhost does not work"

Run this checklist in order.

0. **Is the hostname spelled exactly right?** A typo sends the request to public DNS, which returns NXDOMAIN, and the proxy never sees it. Grep the exact hostname across the proxy config, the application's own public-URL setting and the application source. `getent ahostsv4 <host>` returning empty while an external resolver answers is the tell.
1. Is the handle actually in the **deployed** file, not only in the repo?
2. Did validation pass **with the environment loaded**?
3. Did you **restart** rather than reload?
4. Does the backend listen? `curl http://127.0.0.1:<port>/`
5. Is the certificate from the expected wildcard? Check the subject and issuer with `openssl s_client`.

**An empty access log for a hostname means the request never reached the proxy.** The fault is in DNS or in the hostname. Do not edit matchers, server names or upstreams until the request is arriving.

## The proxy binds to the LAN address, not loopback

The proxy listens on the LAN address and on the IPv6 wildcard — not on `127.0.0.1`. A local vhost check must therefore pin the LAN address:

```bash
curl -sS --resolve <host>:443:<lan-ip> -o /dev/null -w 'HTTP=%{http_code}\n' https://<host>/ -k
```

Using `--resolve ...:127.0.0.1` returns connection refused. That is not a service-down signal. Check the actual listeners with `sudo ss -tlnp | grep ':443'` before interpreting a refused connection.

If the direct backend call fails too, the service is down. If the backend answers and the vhost does not, the problem is in the proxy path — certificate, matcher or upstream.

## Retiring a vhost

Commenting the inner handle is not always enough. If the hostname also appears in a parent's host list, requests still match the parent, fall through every child handle, and get answered by the next matching handle — often a catch-all — producing a 200 from an unrelated service.

1. Remove the hostname from the parent's host list **and** disable the inner handle.
2. Validate with the environment loaded, then restart the proxy.
3. Probe every state explicitly, expecting 404 — not 200.
4. Spot-check that the parent's other hostnames still answer.

The same shape applies to any grouped matcher. Check who consumes the matcher, not only who the matcher targets.

## Sweep after every change

```bash
for h in $(grep -oE '@[a-z-]+ host' /etc/caddy/Caddyfile | sort -u | sed 's/@\(.*\) host/\1/'); do
  echo "$(curl -sS --resolve ${h}.example.com:443:<lan-ip> -o /dev/null -w '%{http_code}' https://${h}.example.com/) ${h}"
done
```

One row with the wrong status is the smoking gun: 200 on a retired hostname, 404 on a live one.

## Injecting an application credential at the proxy

When a self-hosted app demands an API key from the browser but the key already lives on the server, the proxy can supply it so the user never pastes it:

```caddy
@<slug> host <slug>.example.com
handle @<slug> {
    reverse_proxy 127.0.0.1:<port> {
        header_up Authorization "Bearer {$<APP>_API_KEY}"
    }
}
```

- `header_up` is a sub-directive of `reverse_proxy`. At the top level the config aborts with `unrecognized directive: header_up`.
- Load-time substitution (`{$VAR}`) fails the config when the variable is missing, so the vhost does not come up. Runtime substitution (`{env.VAR}`) silently yields an empty value. For a mandatory credential, prefer failing loudly.
- The variable arrives through the unit's environment file, so validation needs it in the environment: `sudo -E sh -c 'set -a; . /etc/caddy/*.env; set +a; caddy validate --config /etc/caddy/Caddyfile'`.
- **Security consequence to report:** from that moment on, every client that reaches the vhost uses the credential without authenticating. If the vhost is LAN-only, the exposure is the LAN; if it is public, it is the internet. If the key must remain mandatory, use `basic_auth` instead — the browser remembers it once.
