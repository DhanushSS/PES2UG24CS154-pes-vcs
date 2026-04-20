# PES-VCS Lab Report (Code + Analysis)

## Phase 5: Branching and Checkout (Analysis)

### Q5.1
A branch is a file under `.pes/refs/heads/` that stores a commit hash. To implement `pes checkout <branch>`:

1. Validate that `.pes/refs/heads/<branch>` exists.
2. Read target commit hash from that branch file.
3. Resolve target commit -> root tree from object store.
4. Compare current working tree state vs index/HEAD for safety (dirty-check).
5. If safe, rewrite working directory to match target tree:
   - Create missing files/directories from target tree blobs/trees.
   - Remove tracked files/directories that do not exist in target tree.
   - Apply file modes (100644/100755).
6. Rewrite index to match checked-out tree snapshot metadata.
7. Update `.pes/HEAD` to `ref: refs/heads/<branch>`.

Why this is complex:
- Requires full tree materialization from object store into filesystem.
- Must detect and prevent data loss on local edits.
- Needs careful handling of deletes, nested directories, and file mode changes.
- Must keep HEAD, index, and working directory mutually consistent.

### Q5.2
Dirty conflict detection (using index + object store):

1. For each tracked file in index:
   - `stat()` working file.
   - If missing -> unstaged deletion.
   - If mtime/size changed, re-hash file as blob and compare with index hash.
   - If hash differs, file is dirty.
2. Resolve current HEAD tree and target branch tree.
3. For each dirty tracked file, check whether the file differs between current HEAD tree and target tree:
   - If target blob hash != current HEAD blob hash (or file added/removed in target), switching would overwrite semantic content.
4. If any such path exists, abort checkout and print conflicting paths.

This avoids false positives from timestamp-only changes and catches real overwrite risks.

### Q5.3
Detached HEAD means `.pes/HEAD` stores a commit hash directly (not `ref: ...`). New commits in this state:

- Still get created normally with parent links.
- Advance HEAD to newer detached commits.
- Are not referenced by any branch name.

Risk: user can "lose" them after moving HEAD elsewhere, because no branch keeps them reachable.

Recovery:
- If user still knows commit hash (or from reflog-like history if implemented), create a new branch file under `.pes/refs/heads/<name>` pointing to that commit.
- Then switch HEAD to that branch (`ref: refs/heads/<name>`).

## Phase 6: Garbage Collection and Reclamation (Analysis)

### Q6.1
Safe mark-and-sweep algorithm:

1. Build root set = all refs under `.pes/refs/heads/*` (and tags if present), plus detached HEAD hash if HEAD is direct.
2. Mark phase (graph traversal):
   - Use DFS/BFS from each root commit.
   - For each commit: mark commit object, then visit its tree and parent commit.
   - For each tree: mark tree object, then visit each entry hash (blob/tree).
   - Blobs are terminal nodes.
3. Sweep phase:
   - Enumerate all files under `.pes/objects/*/*`.
   - Delete objects not in marked set.

Efficient structure: hash set keyed by 32-byte hash (or hex string) for O(1)-average membership checks.

Scale estimate (100,000 commits, 50 branches):
- Traversal visits all reachable commits at least once, but shared history prevents multiplicative blow-up.
- Roughly O(commits + trees + blobs) reachable objects; commonly a few hundred thousand to a few million objects depending on churn and repository size.

### Q6.2
Concurrent GC vs commit race:

- Commit process may write new tree/blob/commit objects, then update branch ref at the end.
- If GC snapshots refs before branch update, new objects are not yet reachable from any ref.
- GC could sweep those just-written objects.
- Commit then updates ref to point to a commit whose subtree objects were deleted -> broken history.

How real Git avoids this (conceptually):
- Uses reachability protections and grace windows (recent objects are preserved).
- Uses lockfiles/coordination around ref updates and maintenance.
- Packs/prunes with safety constraints so objects concurrently being written are not immediately pruned.
- Reflogs and conservative prune horizons reduce accidental loss.

## Verification Notes

In this Windows workspace session, `make` and OpenSSL headers are not available in the active terminal, so full local execution of required test binaries could not be completed here. The code changes were implemented according to the assignment format/contracts and validated for static editor errors.

Recommended verification on Ubuntu 22.04 (as required by assignment):

```bash
make clean
make all
./test_objects
./test_tree
make test-integration
```

Then capture all required screenshots listed in README.
