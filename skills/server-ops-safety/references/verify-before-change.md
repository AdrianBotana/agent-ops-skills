---
name: verify-before-change
description: "Idempotent verification commands, the three-place service edit, drift detection and the safe procedure for retiring SSH keys."
---

# Verify before change

Read-only checks you can run twice with the same result. Use them to pick the right command, to prove a change landed, and to detect drift.

## Idempotent command table

| Want | Command |
|---|---|
| Unit is active / enabled | `systemctl is-active <unit>` / `systemctl is-enabled <unit>` |
| Full unit definition including drop-ins | `systemctl cat <unit>` |
| Merged environment of a unit | `systemctl show <unit> -p Environment` |
| Unit is failed (clean count) | `systemctl --failed --no-legend \| wc -l` |
| Port listening / port owner | `ss -tlnH sport = :<port>` / `ss -tlnpH sport = :<port>` |
| All listeners on the host | `ss -tlnp` |
| External references to a path | `grep -rIln <path> /etc/systemd /etc/cron.d ~/.config/systemd` |
| Recent unit log | `journalctl -u <unit> -n 50 --no-pager` |
| Errors since a timestamp (no boot filter) | `journalctl -p err --since "<ISO timestamp>" --no-pager` |
| Scheduled jobs and next run | `systemctl list-timers --all` |
| Container state / restart count | `docker ps -a --format '{{.Names}}\t{{.Status}}'` / `docker inspect --format '{{.State.Status}} {{.RestartCount}}' <container>` |
| Disk usage and inodes | `df -h` and `df -i` |
| SMART summary | `smartctl -H -A /dev/<device>` |
| Filesystem state without repairing | `tune2fs -l /dev/<partition> \| grep 'Filesystem state'` |
| Write test inside a mounted filesystem | `dd if=/dev/urandom of=<mount>/X bs=1M count=100 && md5sum <mount>/X` (then remove it) |
| DNS resolution | `getent hosts <name>` |
| External resolution | `dig +short <name> @1.1.1.1` |
| Backend directly | `curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:<port>/` |
| Vhost through the proxy | `curl -sS --resolve <host>:443:<lan-ip> -o /dev/null -w 'HTTP=%{http_code}\n' https://<host>/` |

**Never write to a raw block device to "test the disk".** A write to `/dev/<device>` destroys the filesystem metadata (primary superblock, group descriptor table, early inodes) and leaves the filesystem unmountable. Write a file inside the mounted filesystem instead. To measure the link without writing anything, read from the device to `/dev/null`.

**Never force a filesystem repair on a filesystem that is already writing without I/O errors.** The residual bitmap errors are counters, not corruption, and forcing the repair over hundreds of gigabytes of production data carries more risk than it removes.

## The three-place service edit

A service is described in three places. Change all three or the drift resurfaces as a 502 or a stale entry.

1. **The catalogue** — the array the launch panel renders. Schema is roughly `{ slug, name, hostname, port, category, icon, desc }`.
2. **The proxy** — the host matcher plus the reverse-proxy directive. Keep the handles alphabetical: the proxy does not care, reviewers do.
3. **The documentation** — the canonical table the operator reads.

**Adding a service:** verify it is running (`systemctl is-active <unit>`, `ss -tlnp | grep :<port>`) → append to the catalogue → add the proxy handle → update the documentation row → validate the proxy config, restart the proxy, rebuild the panel → verify with `curl --resolve <host>:443:<lan-ip> -sI https://<host>/`.

**Retiring a service:** confirm it is down first — diagnose first, "failed" is not "retired" (`systemctl is-active <unit>` inactive, `ss -tlnp | grep :<port>` empty) → remove the catalogue entry (do not keep history in the catalogue) → remove the proxy handle and push → mark the documentation row retired and archive the note → rebuild the panel and confirm the entry is gone.

**Moving a port:** update the catalogue port, the proxy upstream and the documentation, then restart the service and the proxy, then re-run the drift diff.

### The retirement must actually kill the path

A static site with a single-page fallback serves its home page for **any** path that no longer matches a file, so a retired path keeps answering 200 and users see the home page instead of a 404.

```nginx
location ~* ^/(retired-path|retired-file\.xml)(/|$) { return 410; }
```

`410 Gone` is the semantically correct status: the resource was intentionally retired, not temporarily missing. Search engines understand the difference. Place the rule **before** the catch-all, reload the web server, then probe: the retired path must return 410 and the root must still return 200.

## Drift detection

Compare what the catalogue declares against what the host actually listens on:

```bash
diff <(grep -oE "port: [0-9]+" <catalogue-file> | sort -u) \
     <(ss -tlnH | awk '/LISTEN/ {split($4,a,":"); print "port: " a[2]}' | sort -u)
```

Mismatches are drift. Read the direction: a catalogue port with no listener means the service is down or the catalogue is wrong; a listener with no catalogue entry means an undocumented service. Fix the catalogue or restart the listener, depending on which side is wrong.

## Repository drift

Uncommitted work and unpushed commits are invisible to every port and catalogue check. A service can be "running" while the code on disk is older than the committed version, or the other way round — and a restore from the remote then silently loses the last hours of work.

```bash
for d in <projects-dir>/*/; do
  name=$(basename "$d"); [ -d "$d.git" ] || continue; cd "$d" || continue
  git fetch --quiet --no-tags 2>/dev/null
  local=$(git rev-parse HEAD 2>/dev/null); remote=$(git rev-parse '@{u}' 2>/dev/null)
  dirty=$(git status --porcelain 2>/dev/null | head -1)
  [ "$local" = "$remote" ] && [ -z "$dirty" ] && continue
  printf '  %-30s local=%s remote=%s %s\n' "$name" "${local:0:8}" "${remote:0:8}" \
    "$([ -n "$dirty" ] && echo DIRTY; [ "$local" != "$remote" ] && echo AHEAD)"
done
```

**DIRTY** — uncommitted changes on disk: review and commit or revert. **AHEAD** — committed locally, not pushed: push it. **No upstream tracking** — cosmetic, set the upstream. Caveat: a bind-mounted runtime config that a running container rewrites is runtime state, not source — surface the diff and ask before committing it.

## Retiring SSH keys safely

Retiring a key you did not identify first is a guaranteed lockout.

**1. Identify the key the live session is using.** The connection environment shows the client, and the fingerprint of the active key can be derived from the running process tree. For each candidate private key, extract the public key and look for it in the authorized-keys file: `ssh-keygen -y -f ~/.ssh/<key> | awk '{print $2}' | grep -f ~/.ssh/authorized_keys`.

**2. Build the whitelist**: the key matching the live session, plus the keys of the machines currently in use. The comment field is not proof — a key labelled with a retired machine can still be the active one.

**3. Back up before editing**: `cp authorized_keys authorized_keys.bak.$(date +%Y%m%d-%H%M%S)` and `chmod 600 authorized_keys`.

**4. Filter, then diff** the before and after, then confirm the new file still contains the public key of the live session.

**5. Do not retire keys at all** until the operator has explicitly authorised it and the active fingerprint has been verified. Recovery requires physical or console access to the machine, which is rarely available.
