# PES-VCS Lab Report
**Name:** Tanmayee Kashyap
**SRN:** PES2UG24CS552

---

## Phase 1 Screenshots
- Screenshot 1A: ./test_objects output — all tests passing
- Screenshot 1B: find .pes/objects -type f — sharded directory structure

## Phase 2 Screenshots
- Screenshot 2A: ./test_tree output — all tests passing
- Screenshot 2B: xxd of raw tree object

## Phase 3 Screenshots
- Screenshot 3A: pes init → pes add → pes status sequence
- Screenshot 3B: cat .pes/index showing text-format index

## Phase 4 Screenshots
- Screenshot 4A: pes log output with three commits
- Screenshot 4B: find .pes -type f | sort showing object growth
- Screenshot 4C: cat .pes/refs/heads/main and cat .pes/HEAD

## Final Integration Test
- All integration tests completed successfully

---

## Phase 5: Branching and Checkout

### Q5.1: How would you implement `pes checkout <branch>`?

To implement `pes checkout <branch>`, the following changes need to happen:

1. **Read the target branch file** at `.pes/refs/heads/<branch>` to get the commit hash.
2. **Read the commit object** to get its tree hash.
3. **Walk the tree recursively** and restore every file in the working directory to match the tree's blobs.
4. **Update `.pes/HEAD`** to contain `ref: refs/heads/<branch>`.

This operation is complex because:
- Files that exist in the current branch but not in the target must be deleted.
- Files modified in the working directory could be overwritten and lost.
- Nested subdirectories must be created or removed as needed.
- The index must also be updated to reflect the new branch's tree.

### Q5.2: How would you detect a "dirty working directory" conflict?

To detect conflicts when switching branches:
1. For each file in the current index, compare its `mtime` and `size` against the actual file on disk.
2. If any file has been modified (metadata differs), the working directory is "dirty".
3. Additionally, check if the file differs between the two branches by comparing blob hashes in their respective trees.
4. If a file is dirty AND differs between branches, refuse the checkout and print an error.

This uses only the index and object store — no external diff tools needed.

### Q5.3: What happens in "detached HEAD" state?

In detached HEAD state, `.pes/HEAD` contains a raw commit hash instead of a branch reference like `ref: refs/heads/main`. If you make commits in this state, `head_update` writes the new commit hash directly into `HEAD`. However, no branch pointer is updated, so these commits are not reachable from any branch. To recover them, the user can create a new branch pointing to the detached HEAD commit: `pes branch <new-branch> <hash>`, which creates `.pes/refs/heads/<new-branch>` containing that hash.

---

## Phase 6: Garbage Collection

### Q6.1: Algorithm to find and delete unreachable objects

**Algorithm:**
1. Start from all branch refs in `.pes/refs/heads/` — collect every commit hash.
2. For each commit, add its hash to a `reachable` set, then follow its `parent` pointer.
3. For each commit's tree, recursively walk all tree entries — add every tree and blob hash to the `reachable` set.
4. List all files under `.pes/objects/` — any hash NOT in the `reachable` set is garbage and can be deleted.

**Data structure:** A hash set (e.g. a hash table or sorted array) to track reachable hashes efficiently — O(1) lookup per object.

**Estimate for 100,000 commits, 50 branches:**
- Each commit references ~10-50 objects on average (blobs + trees).
- Total objects to visit: roughly 100,000 × 20 = ~2 million objects.
- With 50 branch starting points, the full reachability traversal visits all of them.

### Q6.2: Race condition between GC and commit

**Race condition:**
1. GC scans the object store and builds the reachable set — at this moment, a new commit is being created.
2. The new commit's `object_write` writes a new blob to the store.
3. GC hasn't seen this blob yet (it scanned before the write), so it marks it as unreachable.
4. GC deletes the blob.
5. The commit finishes and references the now-deleted blob — the repository is corrupted.

**How Git avoids this:**
- Git uses a "grace period" — objects newer than a certain age (default 2 weeks) are never deleted by GC, giving in-progress operations time to complete.
- Git also uses lock files to prevent concurrent GC runs.
- The temp-file-then-rename pattern ensures partially written objects are never visible to GC.
