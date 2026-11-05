---
name: service-retirement
description: "Use when retiring a service, project, unit, container, hostname or note: find dependents, preserve the data, tear down in order, and prove the removal."
version: 0.1.0
author: Adrián Botana
license: MIT
platforms: [linux]
---

# Service Retirement

Retiring a service means removing it without breaking whatever depends on it and
without losing data that only exists locally. The workflow is ordered so that the
cheap reversible checks happen first, the preservation happens before the
deletion, and the removal is proven afterwards rather than assumed.

## When to Use

- The user says "retire", "shut down", "remove", "decommission", "stop using" a service, project, unit, container, hostname or note.
- A service has been inactive for a long time and needs to be cleaned up.
- A proxy vhost points at a port that no longer has a backend.
- A catalog entry has no live service behind it.
- A public hostname is no longer used.

Do not use for: "why is this down" or "restart this" — those are diagnostics.

## The Rules

### Tier of autonomy

**1. Retirement is a confirm-required action.** Read-only audits are autonomous; teardown is not.
Retire a service from the catalog only, keeping the backend alive: explain and act. Retire a project fully (stop the unit, remove the vhost, sweep the catalog, delete the tree): explain what and why in one line, act, and be ready to undo. Retire a failed unit, bulk-retire more than a few projects, or delete a repository from the code host: diagnose first, present the plan, wait for approval.

**2. A generic "do it all" does not cover irreversible steps.** After a blanket order, run the first recognisable step, then pause and ask before the irreversible one. An emphatic order after a scare does not raise the tier either.

### Establishing the facts

**3. Confirm end-of-life intent before touching anything.** "It does not work" is not "it is finished". Read the request and the history; if the intent is ambiguous, investigate first — the unit status, the last fifty journal lines, the listener on the port. Do not retire a service that is mid-recovery.

**4. A failed unit is a bug until proven otherwise.** Read the journal before deciding. A unit that keeps restarting because of a configuration error is not end-of-life; it is a bug that will bite again once the service is redeployed.

**5. Capture a baseline before the first deletion.** Without it, the question "did you delete it?" has no honest answer: you cannot distinguish what your session removed from what was already gone, and you end up either claiming deletions you did not make or denying ones you did.
Baseline source: an independent mirror of the filesystem, or a listing taken before you touched anything.
```bash
find <projects-dir> -maxdepth 1 -mindepth 1 -type d -printf '%f\n' | sort > /tmp/baseline.txt
comm -23 /tmp/baseline.txt /tmp/final.txt    # run again at the end: what disappeared
```
Attribution rules: something that disappeared and is present in the code host with verified content was preserved, not lost. Something that disappeared with no record in your own deletion list was **not** deleted by you — report it as "no record of deleting it, it was in the mirror at HH:MM, its repositories are intact" and ask whether it rings a bell. Something in your own deletion list is yours, with its name and destination.

**6. The baseline also catches what you missed.** In a 103-directory audit, nine directories had disappeared without a record. Without the baseline those nine are reported as deleted by the audit (right by luck) or as untouched (wrong).

**7. Find every dependent before you break it.** Search the whole project tree for references to the project path, the slug, the unit name and the port, excluding dependency and version-control directories. A system unit pointing at a project path, a note linking to a service URL, and a panel calling a retired endpoint are the usual three.
Also check the layers a name sweep misses: a static site served by a separate web server does not appear in the container list or the unit list.

**8. Check the deployment footprint in every layer.** Containers, units, proxy vhosts, tunnel ingress entries, web-server roots, cron entries, and the catalog. Judging only on containers and units once deleted a live site whose web-server root pointed at a missing directory, and it stayed broken for three days with no alert firing. Before writing "not deployed", check in this order: listeners and web-server roots, proxy vhosts, tunnel ingress, then compose files and units.
If the site is served as static files, its "deployment" is a clone plus a pull — there is no build step and no artefact to regenerate. The repository *is* the artefact.

### Preservation

**9. The code host is a safety net, not the source of truth — the flow is one way.** The server pushes to the code host; the code host is where deleted work is stored. The only exception is an explicit restore requested by the operator.
Do not clone from the code host back to the server to "rebuild" a broken local clone. If the local clone is corrupt, push its content up and leave it as it was. If you need a clean clone to push from, clone into a scratch directory and delete it afterwards.

**10. "The repository exists" is not proof that the content is safe.** A repository can exist and be empty, hold the code without the data, or hold the content on a branch you are not looking at. The only proof is blob-hash coverage: compute the hash of every file on disk and check it against the tree of the remote on the correct branch, including the full history (`git cat-file --batch-all-objects`). A deleted file is still recoverable from an old commit.
```python
import hashlib
def blob_hash(data: bytes) -> str:
    h = hashlib.sha1(); h.update(b"blob %d\0" % len(data)); h.update(data)
    return h.hexdigest()
```
Also check `default_branch` — many repositories use `master`, not `main`, and a renamed project changes its slug, so a 404 on the old name does not mean it is missing.

**11. `git add -A` does not upload what the ignore file excludes.** The commit-push-delete flow only protects what version control tracks. A data file, a state database or a generated listing marked as ignored enters no commit, appears in no push, and dies with the tree.
```bash
git status --ignored --short
git ls-files --others --ignored --exclude-standard
```
Then upload what is missing explicitly with `git add -f <path>`, or decide consciously that it is not preserved. The tell is a project with code but no data — a repository of that shape is nearly useless for restoring.

**12. Commit before you delete, and verify the push.** A new commit hash is not proof of a push and not proof that the content landed. Check the local head against the remote ref:
```bash
L=$(git rev-parse HEAD); R=$(git ls-remote origin refs/heads/main | cut -f1)
[ "$L" = "$R" ] && echo SYNC || echo DESYNC
```
On-disk ref files lag behind after a push and are never proof. Verify the API endpoint, which reflects in-memory state.

**13. The tombstone commit comes first.** Before deleting the working copy, commit the retirement notice — a README with the retirement date, the reason and where the code lives. If you delete first and the push fails, the repository has no retirement marker and the next operator cannot tell retirement from abandonment.

**14. Never hard-reset a clone with uncommitted work.** That work exists nowhere else. Commit it first — a `wip:` commit is fine.

**15. A local clone that diverged from the remote: merge, never rebase.** If the common ancestor is neither head, the lineages differ and a rebase rewrites published history. Commit the local state as `wip:` first — that also unblocks the merge when untracked files collide — then merge, resolve by size and recency rather than by eye, and verify blob-hash coverage before committing.

### Teardown

**16. Stop, disable, remove, reload.** In that order, for the unit or the timer, then reload the manager. Confirm the unit is inactive and the port is free afterwards.

**17. Retire every unit that serves the stack, not only the config files.** The obvious targets are the compose file, the proxy block and the catalog entry. The non-obvious target is every unit that starts, runs or proxies for the project — including dedicated sub-servers, timers, path units, and units created in a side project that were never wired into the main repository. A dedicated proxy unit that survives its project enters a restart loop forever, spamming the journal at the alerting threshold and hiding real failures.
```bash
for u in $(systemctl list-unit-files --state=enabled --no-legend | awk '{print $1}' | grep -E '<slug>'); do
  printf '%-40s %s\n' "$u" "$(systemctl is-active $u)"
done
```

**18. Patch the proxy configuration in the repository, never in place.** The repository file is the source of truth; the deployed file is the target. Edit the source, format it, validate it, and let the deploy path copy it and restart the server.
```bash
caddy fmt --overwrite <repo>/Caddyfile
sudo -E sh -c 'set -a; . /etc/caddy/cloudflare.env; set +a; caddy validate --config <repo>/Caddyfile'
```
Formatting is not optional — it normalises indentation, and a non-zero exit means the file is unrecoverable as configuration. A misplaced tab inside a nested block breaks the parent handler for every hostname it contains.

**19. Restart the proxy, do not reload it.** The proxy caches its upstream token at boot, so a reload can fail silently with a connection error on the local admin endpoint when the server has been idle. Restart is the safe path.

**20. A validation failure under a non-root user can be a permission artefact, not a syntax error.** Validation can fail on log-file permissions rather than on the configuration. Judge the change by the deploy log, not by a failed non-root validation.

**21. A proxy vhost whose document root was deleted answers 404, not 502 — which makes it look like a proxy problem.** Diagnose in three steps: request the root directly and look at the status (404 means the proxy is fine and the directory is missing), list the root directory to confirm, then find the vhost whose `root` points at it. A 502 means the backend is not listening; a 404 with the web server's own header means the root does not exist.

**22. A deploy script that generates configuration is a second source of truth, and it rots.** An unused generator drifts from the installed state and reintroduces bugs when it finally runs. Before executing one, diff what it generates against what is installed (the listen directives and the roots). A syntax check does not detect bind conflicts or duplicate server names — those only appear when the server actually starts. A wildcard bind on one address family silently absorbs a specific bind on the same port in the other family.

**23. Keep the catalog, the proxy configuration and the documentation in sync.** Three places change when a service is added, retired or moved: the service catalog in code, the proxy vhost, and the human-readable service table. Miss one and the drift surfaces a week later.
```bash
diff <(grep -oE "port: [0-9]+" <catalog-file> | sort -u) \
     <(ss -tlnp | awk '/LISTEN/ {split($4,a,":"); print "port: " a[2]}' | sort -u)
```
Mismatches are drift. Note that a tombstoned catalog entry with a surviving backend produces an expected mismatch — report it as expected and documented, not as a bug.

**24. A service entry is five places; a panel is seven.** A catalog entry needs: the backend route file removed, the import and registration removed from the server file, the catalog entry replaced by a tombstone, the slug removed from the launcher set, and the frontend component and its section removed.
A panel or tile with its own route adds: the slug removed from the shared API client's type union and its typed fetch helper deleted, and the tab-bar entry removed together with its icon import.
Order matters: delete the type union entry **before** the component, and remove the icon import **with** the tab-bar entry, otherwise the type checker reports errors against the wrong file.

**25. One tombstone per retired entity, appended over time.** If a tombstone already exists from a previous session, do not add a second one — find it and append a dated line describing what changed since, or rewrite the reason if the context materially differs. The same applies to historical comments: append a dated line, never rewrite the original note.
Tombstone shape: `// { slug: '<slug>', ... } — RETIRED <YYYY-MM-DD>. <reason>. Repo: <code-host>/<user>/<slug>.git`. Copy the shape of an existing tombstone rather than inventing a new one.

**26. Purge the documentation that points at the dead local path.** A retired project leaves pointers to its filesystem path in places outside the repository you deleted, and they mislead the next operator.
```bash
grep -rn "<projects-dir>/<slug>" <projects-dir> --exclude-dir=node_modules --exclude-dir=.git --exclude-dir=dist
grep -rn "<slug>" <docs-dir>/
```
The usual offenders: reference files in procedure libraries that say "the full code lives at ..." — rewrite the paths as repository-relative and add a short retirement note pointing at the code host. If the procedure exists in a live copy and a versioned copy, **patch both and verify they are identical**; editing only the versioned copy leaves the live copy serving stale text. Also add a dated entry to the operation log — the log is the audit trail, and a retirement that is not in the log did not happen as far as the documentation is concerned.

**27. "Fix what you flagged" does not mean "restore".** Every finding from a diagnostic carries an explicit intent label: `FIX` (something broken that must be put back) or `RETIRE` (something withdrawn that must be cleaned up). Execute according to the label.
Before investing work in a restoration, contrast the repository with the signs of life: recent commits, a working copy, a unit or container, a catalog entry, real traffic. A repository with no commits for months, no working copy, no unit, no container, no catalog entry and no traffic is retired even if its vhost is still on disk. What needs cleaning is the vhost, not the service. When in doubt, ask: the irreversible action is worth a question, and so is the reversible one.

**28. A static site with a single-page fallback keeps answering 200 for retired paths.** The fallback serves the root document for any path that matches no file, so a retired section keeps returning the home page. After removing the routes, rebuild, confirm the output no longer contains them, and add an explicit `410 Gone` for the retired paths **before** the catch-all in the web-server configuration. 410 is the semantically correct status: the resource was intentionally retired, not temporarily missing. Skip this and users hit the home page for months and crawlers keep indexing a phantom path.

**29. A retired subcommand stays in the dispatcher with a retired handler.** Do not delete the entry from the case statement or the function — callers with the argument hardcoded then fail with a confusing usage message or a silent 404. Replace the body with a handler that returns a clear error and a non-zero exit code. The usage text keeps listing the subcommand, cached callers fail loudly with a useful message instead of "command not found", and reactivating later is a one-line restore. For an HTTP endpoint, return a JSON body with an error field and a 200 status — the endpoint exists, it is retired.

**30. Retiring a whole stack crosses more surfaces than one workflow covers.** A named application can touch: the home-automation entities, the root-owned helper scripts, the units those scripts drive but never enabled, and the critical scripts that still reference the retired stack in their dead branches. Map every surface with one grep over the script directories, the unit directories and the optional install directories, then classify each hit: script to delete, script to patch, unit to retire, configuration to remove, comment-only (do not touch). Execute in that order and write a manifest listing what was deleted versus what was patched, with rollback instructions.
Do not touch a retired stack again unless the operator asks.

**31. A cold mirror is a staging area, not an archive.** Snapshot to the cold area before deleting, push, verify the remote has the history, then delete the cold copy — it is a duplicate at that point. Leaving it "just in case" is not a strategy; a long-term archive belongs in the code host or in a dedicated archive area. The cold area mirrors without deleting, so it accumulates: it is expected to hold more than the live tree, and a difference in that direction is not a finding. The reverse — something live with no cold presence at all — means the mirror never copied it.

**32. Two projects covering the same domain need disambiguation before any deletion.** Confirm with the operator which one is the real one, and identify it by inspecting the brand fingerprint or the hardcoded name in the source: the active project has the current brand, the retired one has a stale one. Default when the operator says "get rid of the old one": delete the older generation and archive the snapshot. The intent is to clean up the mess, not to delete history. The code host is preserved either way; the decision is whether to keep the cold mirror and the live reference.

**33. Never retire SSH keys without identifying the active one.** An existing shell survives the change to the authorised-keys file, but if you disconnect and the active key was removed, you cannot get back in.
Identify the key in use before touching anything: read the current user, the host and the SSH connection environment, then for each private key extract its public part and look for it in the authorised-keys file. Do not retire a key you have not identified as inactive — keys labelled as generated or as belonging to retired machines are the tempting ones, and they can be the live one. If you do retire one: edit the file leaving the active key last, keep a timestamped backup, diff before and after, verify the new file still contains the public key of the live session, then fix the mode and reload.

**34. Deleting root-owned files needs an escalation path that does not prompt for a password.** If the removal command is not on the non-interactive allowlist, `sudo` will prompt and the agent must not satisfy it. A privileged container joined to the host namespaces reaches the real filesystem as root:
```bash
docker run --rm --privileged --pid=host alpine:latest \
  nsenter -t 1 -m -u -i -n -p -- rm -f /etc/systemd/system/<unit>.service
systemctl daemon-reload
systemctl list-unit-files <unit>     # expect not-found
```
The same wrapper is the only way to delete other root-owned files. An in-place write tool serves for writing root files, not for deleting them.

**35. Never `rm -rf` the directory the shell is sitting in.** Removing the current working directory breaks every later command in the session, because the shell tries to restore the working directory before running the next line. Change directory first, or set the working directory of the next command explicitly.

**36. A bulk delete script that pushes and then removes hides what it touches.** Run it in dry-run mode first and read the output as a checklist of every hook it fires — package registry publishing, homepage configuration, web-server vhosts, note archives, code-host indexing. Every hook that targets a file you did not know about is a place where the script can write outside the project. Hooks that are no-ops are fine; hooks that point at live infrastructure need a decision first.

**37. A project created and retired in the same session leaves more behind than its directory.** The footprint includes the units it created (disabling is not enough — the file stays and keeps being listed), the scratch and workspace directories, the memory entries about the topic, the catalog panel and the note. Walk an explicit layer list — project, units, ports, processes, scratch, memory, catalog, notes, code host — and report each layer's verified state. "I removed everything" after deleting only the directory is false.

### Proving it

**38. Prove the removal with a diff against the previous state, not with a grep.** A zero-hit grep on the working tree is not proof: the tree and the committed state can differ, and what ships is the committed state.
```bash
PRE=$(git rev-list --max-count=1 --skip=1 HEAD)
git diff <PRE> HEAD --stat          # must be empty
```
If the diff shows a lone added blank line, the file has a trailing newline residue — normalise it.

**39. Never silence the error output of a staging command.** `git add -A <path-list>` aborts the **whole** command when any listed path is absent from the working tree, so nothing after the missing path is staged and the commit lands half of the intent while claiming to be a complete revert. Stage modified files and deleted files separately, or use `git add -u` for tracked files plus an explicit add for new ones. Before committing, `git status --short` must list exactly what you intended; after committing, `git show --stat HEAD` must show only the commit's scope.

**40. Verify each layer with its own probe.** Unit inactive, port free, unit files report not-found, the vhost answers 404 or refuses the connection, the catalogue endpoint no longer lists the slug, the route endpoint answers 404, the type checker exits zero, the build succeeds, the remote refs are unchanged. Report the codes.

**41. The final report has three parts.** What changed, one bullet per file or service. What was verified, with the actual codes and refs. What is pending, including anything the operator must decide.

**42. Report progress during a long retirement.** The operator sees the progress line of each command, not the reasoning. A chain of thirty commands with no commentary reads as a hang and produces repeated "what have you done?" questions. One concrete line every three to six commands — "37 candidates, verifying the 5 large ones" — and announce each phase change. If asked, answer with a chronological summary grouped by phase and the current state at the end, not a replay of the commands.

## Reference Files

- `references/retirement-checklist.md` — the ordered checklist from baseline to final report, the tier matrix, the "never do" list, and the verification probes per layer.
- `references/dependents-and-data.md` — how to find every dependent across the layers, how to prove the data is preserved with blob-hash coverage, the ignored-file trap, and the recovery recipes for a broken index and a divergent clone.

Load with `skill_view(name='service-retirement', file_path='references/retirement-checklist.md')`.
Load with `skill_view(name='service-retirement', file_path='references/dependents-and-data.md')`.
