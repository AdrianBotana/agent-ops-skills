---
name: retirement-checklist
description: "Ordered checklist from baseline to final report, the autonomy tier matrix, the never-do list and the per-layer verification probes."
---

# Retirement checklist

## Tier matrix

| Action | Tier | Behaviour |
|---|---|---|
| Read-only audit of the candidate | A | Run it, report it. |
| Retire from the catalog only, keep the backend alive | B | One line of what and why, act, undo on objection. |
| Retire a project: stop the unit, remove the vhost, sweep the catalog, delete the tree (code host preserved) | B | One line of what and why, act, undo on objection. |
| Retire a failed unit | C | Diagnose first, then ask. |
| Delete the repository from the code host | C | Almost never. Preservation is the default. |
| Bulk retire (more than three projects in one session) | C | Present the plan, ask. |
| Retire a public hostname | C | Confirm zero traffic for 24 hours first. |

## Ordered checklist

### Phase 0 — baseline

- [ ] Listing of the directory names, from an independent source, before anything is deleted.
- [ ] Timestamp of the snapshot recorded.

### Phase 1 — facts

- [ ] End-of-life intent confirmed (not just "it is broken").
- [ ] Unit status, last fifty journal lines, listener on the port.
- [ ] Working copy state: uncommitted changes, last commit.
- [ ] Cross-references: `grep -rIln <slug> <projects-dir> --exclude-dir=node_modules --exclude-dir=.git --exclude-dir=dist`.
- [ ] Deployment footprint in every layer: containers, units, proxy vhosts, tunnel ingress, web-server roots, cron entries, catalogue.
- [ ] Protected list: infrastructure excluded by name even with no footprint.

### Phase 2 — preservation

- [ ] Code host API check: repository exists and is not empty.
- [ ] `default_branch` checked (may be `master`, not `main`); alternative branches covered.
- [ ] Blob-hash coverage of every file on disk against the remote tree and the full history.
- [ ] Ignored files enumerated (`git status --ignored --short`, `git ls-files --others --ignored --exclude-standard`) and uploaded explicitly with `git add -f`, or consciously dropped.
- [ ] Tombstone commit made first, with the date, the reason and the repository location.
- [ ] Push verified: local head equals remote ref.

### Phase 3 — teardown

- [ ] Unit stopped, disabled, file removed, manager reloaded. Confirmed inactive, port free.
- [ ] Every unit serving the stack retired, not only the configuration files.
- [ ] Proxy configuration patched in the repository, formatted, validated. Deploy path owns the copy and the restart.
- [ ] Catalogue, proxy and documentation updated together.
- [ ] Panel sweep: five places for a service, seven for a panel.
- [ ] Tombstone appended or updated — never a second tombstone for the same entity.
- [ ] Retired subcommand replaced with a retired handler, not deleted from the dispatcher.
- [ ] Working copy deleted from a directory that is not the shell's working directory.
- [ ] Cold snapshot removed after the push is verified.

### Phase 4 — purge references

- [ ] `grep -rn "<projects-dir>/<slug>" <projects-dir> --exclude-dir=node_modules --exclude-dir=.git --exclude-dir=dist`.
- [ ] `grep -rn "<slug>" <docs-dir>/`.
- [ ] Reference files rewritten to repository-relative paths, with a retirement note.
- [ ] Live and versioned copies of the procedure both patched and verified identical.
- [ ] Dated entry in the operation log.

### Phase 5 — verification

- [ ] `git diff <PRE> HEAD --stat` empty.
- [ ] Unit reports not-found; port free.
- [ ] Vhost answers 404 or refuses the connection.
- [ ] Catalogue endpoint no longer lists the slug; route endpoint answers 404.
- [ ] Type checker exits zero; build succeeds.
- [ ] Remote refs unchanged.
- [ ] Diff against the baseline: what disappeared, and how each item is attributed.

### Phase 6 — report

- [ ] What changed: one bullet per file or service.
- [ ] What was verified: the actual codes, refs and counts.
- [ ] What is pending: decisions for the operator.

## Verification probes per layer

```bash
# Unit
systemctl is-active <unit>                       # inactive
systemctl list-unit-files <unit>                 # not-found
ss -tlnp | grep :<port>                          # empty

# Proxy vhost
curl -sS --resolve <host>:443:<addr> -o /dev/null -w '%{http_code}\n' https://<host>/

# Catalogue and route
curl -s http://127.0.0.1:<port>/api/launcher | jq '.data[].slug' | grep -w <slug>
curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:<port>/api/<slug>

# Code host
git ls-remote <code-host>/<user>/<slug>.git refs/heads/*

# Baseline diff
comm -23 /tmp/baseline.txt /tmp/final.txt
```

## Never do

- Never delete a project tree without confirming the code host has it, verified by content and not by the existence of the repository.
- Never edit the deployed proxy configuration without editing the repository copy and pushing the source.
- Never retire a failed unit without reading the journal.
- Never commit unrelated modifications belonging to someone else.
- Never skip the catalogue sweep on the grounds that the catalogue file is enough — every place is mandatory.
- Never delete the shell's working directory.
- Never delete a repository from the code host without explicit approval.
- Never rewrite a historical comment — append a dated line.
- Never retire a public hostname without checking 24 hours of traffic.
- Never clone from the code host back to the server to rebuild a broken local clone.
- Never hard-reset a clone whose working tree has uncommitted work.
- Never delete without a baseline taken before the first deletion.
- Never claim you deleted something you have no record of deleting.
- Never silence the error output of a staging command.
- Never call a revert done without an empty diff against the previous state.
- Never report "removed everything" after deleting only the project directory.
