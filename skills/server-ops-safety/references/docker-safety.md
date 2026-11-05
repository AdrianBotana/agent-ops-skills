---
name: docker-safety
description: "Container networking rules: the NAT-flush trap, the firewall diagnosis ladder, bind-address checks and the post-change verification."
---

# Docker safety

Container networking is where a host-level edit takes down everything else. These rules assume a host where Docker owns the NAT and masquerading tables and a firewall service manages the host's own filter table.

## The NAT flush trap

A firewall config that runs `flush ruleset` removes **every** nft/iptables table in the kernel, including the ones Docker created: `table ip nat` (chains `DOCKER`, `DOCKER-USER`, rules `MASQUERADE`) and `table ip filter` (chains `DOCKER`, `DOCKER-FORWARD`, `DOCKER-ISOLATION-*`). Docker does not recreate them by itself — only a daemon restart does. From the flush until that restart, every container has no outbound connectivity. If the DNS resolver runs in a container and the host's resolver points at it, the host loses DNS and so does every device that uses it as its resolver. `ping` to a public address still works, because ping does not use DNS.

1. **Never write `flush ruleset` on a host with Docker.** Reload only your own table: `flush table inet filter`.
2. **Never leave an `ExecStop=` or `ExecStopPost=` that flushes the ruleset.** It is inherited from the packaged unit on Debian-family hosts. Read the base unit *and* the drop-ins, not just the config file.
3. **Check Docker's NAT before blaming your change.** If the masquerade count is already 0, the breakage predates your edit. Count before and after.
4. **Use `systemctl reload nftables`, not `restart`.** With a flushing config both verbs break connectivity for every container.
5. **Restarting the container daemon fixes the NAT but takes everything down with it.** It is the emergency escape hatch, not a repair. Containers with `restart: no` do not come back on their own — start them by hand.
6. **Validate without applying.** `nft -c -f <file>` parses and checks without installing. To test for real, copy the set into a scratch table (`sed 's/inet filter/inet filtortest/g'` plus `nft -f`), then `nft delete table inet filtortest`. Never test by applying to production and rolling back.

## Diagnosis ladder

Run in order. The first three separate "no IP" from "no DNS" from "no NAT".

```bash
ping -c 2 1.1.1.1                            # 1. IP-level egress?
getent hosts example.com                    # 2. resolves? (rc=2 = broken)
dig +short @127.0.0.1 example.com            # 3. local resolver answers? (port up, upstream dead)
docker exec <container> ping -c 2 1.1.1.1   # 4. container egress? ("Network unreachable" = no NAT)
sudo iptables -t nat -S | grep -c MASQUERADE  # 5. DECISIVE: 0 = NAT is gone
sudo iptables -S | grep -c DOCKER              # 5b. chains still there?
```

Step 5 returning 0 while step 1 works means "the firewall flush removed Docker's NAT" — not "the router died" and not "DNS is broken".

## The fix, in three pieces

1. **The config** stops flushing the whole ruleset. Declare the table first (so the flush cannot fail on a missing table), then flush only that table: `table inet filter` / `flush table inet filter` / `table inet filter { chain input {...} chain forward {...} chain output {...} }` / `include "/etc/nftables.d/*.nft"`.
2. **A unit drop-in** neutralises the packaged `ExecStop=`: `/etc/systemd/system/nftables.service.d/override.conf` with an empty `ExecStop=`. Without this, stopping or restarting the firewall service re-breaks the NAT even though the config is fixed.
3. **A watchdog** as a safety net: a timer that checks every few minutes that the `DOCKER` chain and at least one `MASQUERADE` rule exist, and restarts the container daemon to recreate them if they do not.

## Bind addresses

A service consumed by other machines on the network must bind the wildcard address, not loopback: `listen(PORT, '0.0.0.0')` in Node, `app.run(host='0.0.0.0')` in Flask, `TCPServer(('0.0.0.0', PORT), Handler)` in the Python standard library, `http.ListenAndServe("0.0.0.0:"+port, nil)` in Go.

Symptom when bound to loopback: `curl http://127.0.0.1:<port>/` works on the server, and a client on the network gets an immediate connection refused. Confirm with `ss -tlnp | grep <port>` — the local address column must show `0.0.0.0:<port>` or the LAN address, not `127.0.0.1:<port>`.

## Post-change check

```bash
sudo iptables -t nat -S | grep -c MASQUERADE   # > 0
sudo nft list table inet filter                 # allowlist present
getent hosts example.com                        # rc=0
docker exec <container> ping -c 2 1.1.1.1       # packets received
curl -s -o /dev/null -w '%{http_code}\n' https://example.com
```

**Do not** treat `ping` success as proof of connectivity. It is proof of IP egress only.
