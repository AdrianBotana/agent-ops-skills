---
name: dependents-and-data
description: "Finding every dependent across the service layers, proving data preservation with blob-hash coverage, and recovering a broken index or a divergent clone."
---

# Dependents and data

## Finding the dependents

```bash
grep -rIln "<slug>" <projects-dir> --exclude-dir=node_modules --exclude-dir=.git --exclude-dir=dist
grep -rIln "<projects-dir>/<slug>" /etc/systemd /etc/<service> ~/.config/systemd
```

Typical hits: a unit pointing at a project path, a note linking to a service URL, a panel calling a retired endpoint, a proxy vhost, a cron entry.

## The six-site orphan sweep
When the operator demands a single canonical instance of a domain, run all six in one command:

1. **Units** — `systemctl list-units --type=service --all | grep -iE "<term>"`. Exactly one active result expected.
2. **Listening ports** — `ss -tlnp | grep -E ":<port-1>|:<port-2>"`. Only the canonical port binds.
3. **Service catalogue** — `grep -E "name: '<term>'|slug: '<term>'" <catalog-file>`. Exactly one row, no tombstone pointing at it.
4. **Proxy configuration** — `grep -E "^@<slug>" <proxy-file>`. Exactly one host matcher, or a documented alias.
5. **Code host** — `curl -s -H "Authorization: token <token>" "<code-host>/api/v1/repos/search?q=<term>"`. At most one live result; archived ones are history.
6. **Filesystem and archive** — `find <archive-dir>/ -iname "*<term>*"` and `find <projects-dir>/ -maxdepth 2 -iname "*<term>*"`. Only archived snapshots, no live directory.

More than one match on sites 1 to 4 is real drift. Sites 5 and 6 are intentional history and are marked as such.

## Layer coverage for "not deployed"
A static site served by a separate web server appears in neither the container list nor the unit list. Before writing "not deployed", check in this order: (1) listeners and web-server roots (`ss -tlnp`, plus the `root` directives in the site files); (2) proxy vhosts in the repository; (3) tunnel ingress entries in the provider's dashboard; (4) compose files and units.

Real consequence of getting this wrong: a sweep deleted the working copy of a site that was being served as static files, leaving the web-server vhost, the proxy vhost and the tunnel hostname pointing at nothing. The site answered 404 publicly for three days and no alert fired, because the health checks watched a different port.

## Blob-hash coverage: the only proof the data is safe

```python
import hashlib, os, subprocess

def blobs(ref):  # blob hashes present in a tree
    out = subprocess.run(["git", "-C", repo, "ls-tree", "-r", ref],
                         capture_output=True, text=True).stdout
    return {l.split("\t")[0].split()[2] for l in out.splitlines() if l}

def blob_hash(data):
    h = hashlib.sha1(); h.update(b"blob %d\0" % len(data)); h.update(data)
    return h.hexdigest()

cover = blobs("origin/<branch>")
missing = []
for root, dirs, files in os.walk(repo):
    rel = os.path.relpath(root, repo).split(os.sep)
    if ".git" in rel or any(p in rel for p in scratch_dirs):
        continue
    for f in files:
        try:
            data = open(os.path.join(root, f), "rb").read()
        except OSError:
            continue
        if blob_hash(data) not in cover:
            missing.append(os.path.join(root, f))
```

Check the full history too (`git cat-file --batch-all-objects`) — a file deleted months ago is still recoverable from an old commit.

**Classify what is missing.** *Regenerable*: build bundles, compiled binaries whose source is present, generated project files whose generator config is present, derived databases whose source data is present. *Real and actionable*: state data, personal data, source files, assets. Upload the real ones to the correct branch before deleting.

**The scratch-directory filter is subtle:** a directory is excluded only if the exclusion is checked against **every** ancestor of the relative path; a naive prefix check on the immediate parent directory misses nested scratch directories.

## The ignored-file trap
```bash
git status --ignored --short
git ls-files --others --ignored --exclude-standard
```

Anything listed there is outside version control and dies with the tree. Real examples: a state JSON file with personal data, a scraped-data database, a generated listing. A repository with code but no data is nearly useless for restoring.

## Broken index: not lost work
Symptom: `git status` marks dozens of files as deleted while they are intact on disk.

1. **The index is missing.** A repository with `.git/` but no `.git/index` rebuilds its state against the head commit; if that fails it marks everything as deleted.
2. **The files are intact.** Verify on disk, not through `git status`: list the files, list the tree of the head commit, read the last commit.
3. **Scope it cheaply** across the whole project tree by counting repositories with and without an index file.

The repair, which does not touch the working tree: `cd <repo> && git reset --mixed -q`. Never the hard reset — that deletes modified files.

**Two failures that look the same.** "cannot read tree" → the tree object is missing: real corruption, the reset does not fix it; diagnose with the integrity check and inspect the refs. "index file smaller than expected" or "index.lock" → the index is corrupt: remove it and reset. Measured on a real host: 38 of 44 repositories were fixed by the mixed reset; 6 failed with "cannot read tree" and had broken objects and refs.

## A fetch failure does not mean the remote is broken

```
error: <remote> did not send all necessary objects
fatal: bad object <sha>
```

That error is about the **local** repository. The remote may clone perfectly:

```bash
git clone --no-checkout --depth 1 <remote> /tmp/check
cd /tmp/check && git fsck --no-progress
```

Real case: the remote cloned with 878 commits and zero integrity errors while the local clone failed both the fetch and the mixed reset. When the clean clone is fine, the problem is local: remove the repository directory and clone again at the same path, or force-fetch after clearing the broken refs.

## A clone that diverged: merge, never rebase
Symptoms: a pull with rebase fails on content conflicts; the working tree has uncommitted changes and the remote has commits the clone lacks; the common ancestor is neither head.
```bash
git -C <repo> log --oneline -3
git -C <repo> log --oneline origin/main -3
git -C <repo> merge-base HEAD origin/main
```

If the common ancestor is neither head, the lineages differ.

1. **Commit the local state first** as `wip:`. It preserves the work and unblocks the merge, because the merge aborts when untracked files collide with files the remote checks out. A `wip:` commit is better than a stash: it stays in the history as proof the state existed, and a stash discarded by a later conflict loses the work silently.
2. **Merge, do not rebase.** A rebase rewrites published commits; a merge preserves them. When the local clone is a restored copy whose history you do not control, rebasing is incorrect by definition.
3. **Resolve by size and recency**, not by eye. For a file the remote deleted and the local copy modified, the local copy is usually the older version — resolve towards the remote and keep the local value in the `wip:` commit.
4. **Verify blob-hash coverage before committing.** A merge that resolves in the remote's favour can drop files that existed only locally.
5. **Commit, push, verify.** `git rev-list --count HEAD..origin/main` must be zero and `git status --porcelain` must be empty.

If the clone was already restored from a mirror, the remaining uncommitted changes are the **originals**, not new work. A blanket add would upload things that were never meant to be published. Compare against the state before the restore.
