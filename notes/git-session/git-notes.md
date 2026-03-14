# Git — Complete Session Notes
> Phase 1 | Week 1 | Session 2
> Date: 2026-03-15
> Environment: Windows laptop (Git Bash) + EC2 Ubuntu 24.04 (AWS Mumbai)

---

## What is Git?

Git is NOT just a version control system.
Git is a **content-addressable filesystem** with version control built on top.

Every piece of data Git stores is an **object**, identified by its SHA-1 hash.
The hash is computed from the content itself — same content always produces
the same hash. This makes Git reliable, tamper-proof, and efficient.

---

## The 4 Git Object Types

### 1. Blob
- Stores raw file contents — nothing else
- No filename, no permissions — just the bytes
- Same content across different files = same blob hash (automatic deduplication)

```bash
git cat-file -p <blob-hash>
# → Hello DevOps
```

### 2. Tree
- Stores a directory listing
- Points to blobs (files) and other trees (subdirectories)
- Each entry: permissions + object type + hash + filename

```bash
git cat-file -p <tree-hash>
# → 100644 blob 0f4ea472...  hello.txt
```

File permission codes:
| Code   | Meaning          |
|--------|-----------------|
| 100644 | Regular file     |
| 100755 | Executable file  |
| 040000 | Directory (tree) |
| 120000 | Symbolic link    |

### 3. Commit
- Points to exactly one tree (the snapshot)
- Contains: tree hash, parent hash, author, committer, timestamp, message
- Parent pointer is how Git builds history — a linked chain of commits
- Root commit has no parent

```bash
git cat-file -p <commit-hash>
# → tree 80706c4e...
# → parent b8fee2f2...   ← previous commit (absent in root commit)
# → author jyotishiipsita <email> 1773513842 +0530
# → committer jyotishiipsita <email> 1773513842 +0530
# → commit message here
```

### 4. Tag
- Named pointer to a specific commit
- Annotated tags = tag object with message + signature
- Lightweight tags = just a ref pointer (no object created)

---

## The Object Chain — Visual

```
COMMIT (b8fee2f)  ← "first commit"
│  tree → 80706c4
│  parent → (none, this is root)
│
└──▶ TREE (80706c4)
     │  100644 blob 0f4ea47  hello.txt
     │
     └──▶ BLOB (0f4ea47)
               "Hello DevOps"
```

After second commit — Git creates NEW objects, reuses unchanged blobs:

```
COMMIT (ee43164)  ← "second commit"
│  tree → 65417f7   ← NEW tree object
│  parent → b8fee2f  ← points back to first commit
│
└──▶ TREE (65417f7)
     │  100644 blob f4b2c94  hello.txt  ← NEW blob (content changed)
     │
     └──▶ BLOB (f4b2c94)
               "Hello DevOps\nSecond line"

Old BLOB (0f4ea47) → "Hello DevOps"  ← still exists, never deleted!
```

**Critical insight:** Git is append-only. Old objects are never overwritten.
Every version of every file ever committed is preserved in the object store.

---

## Branches — Just Pointer Files

A branch is literally a 40-character text file. Nothing more. No magic.

```
.git/
├── HEAD                       ← which branch am I on right now?
└── refs/
    └── heads/
        ├── main               ← contains latest commit hash of main
        └── feature/xyz        ← contains latest commit hash of feature
```

```bash
cat .git/HEAD
# → ref: refs/heads/main

cat .git/refs/heads/main
# → a542aac7f3b...   (40-char SHA-1)
```

### Two levels of indirection

```
HEAD  →  refs/heads/main  →  a542aac (commit hash)
 │               │                │
 │               │                └── the actual snapshot
 │               └── the branch file (lives in .git/refs/heads/)
 └── always = where you are right now
```

**When you `git commit`:**
- Branch file updates to new commit hash
- HEAD stays pointing to the same branch name

**When you `git checkout other-branch`:**
- HEAD updates to point to other branch
- Branch files do not change

---

## Rebase — How it Actually Works

### Standard Rebase

Replays your branch commits on top of another branch tip.

```
BEFORE rebase:                    AFTER rebase:

* fb5479f  feature commit         * ccef211  feature (NEW hash!)
|                                 |          parent: a542aac
| * a542aac  main v2              * a542aac  main v2
| * 82e2510  main v1              * 82e2510  main v1
|/                                * ee43164  second commit
* ee43164  base commit            * b8fee2f  first commit
```

**What Git actually does step by step:**
1. Finds the common ancestor of both branches
2. Saves your commits as patches (diffs)
3. Moves to the target branch tip
4. Replays each patch as a brand new commit object
5. Each replayed commit gets a NEW hash (same content, new parent = new hash)

**The golden rule:**
> Never rebase a branch that others are working on.
> Rebase rewrites hashes. Anyone with old hashes gets conflicts.
> Safe: your own private feature branches only.

### Interactive Rebase

Lets you edit, squash, reorder, or drop commits before sharing.

```bash
git rebase -i HEAD~3
```

Options in the editor:
| Command      | What it does                              |
|-------------|------------------------------------------|
| pick         | Keep commit exactly as-is                 |
| squash / s   | Merge into previous commit                |
| reword / r   | Keep changes, edit message only           |
| edit / e     | Pause to amend the commit                 |
| drop / d     | Delete this commit entirely               |
| fixup / f    | Like squash but discard commit message    |

**Real use case:** Clean up WIP commits before raising a Pull Request.
3 messy commits → 1 clean professional commit.

---

## reflog — Your Complete Safety Net

Records every single HEAD movement ever made in the repository.
Nothing is truly deleted for at least 30 days (default GC period).

```bash
git reflog | head -20
# → a542aac HEAD@{0}: checkout: moving from feature to main
# → 6d46bea HEAD@{1}: rebase (finish): returning to feature/add-logging
# → 6d46bea HEAD@{2}: rebase (squash): clean commit message
# → fb5479f HEAD@{11}: commit: original feature commit (before rebase!)
```

**Recovery workflow:**
```bash
# Find the hash of what you lost
git reflog

# Recover a deleted branch
git checkout -b recovered-branch <hash>

# Recover a dropped commit
git cherry-pick <hash>
```

---

## Git Hooks

Scripts in `.git/hooks/` that run automatically on Git events.
Not tracked by Git — each developer's machine runs its own hooks.
For team-wide hooks use Husky (Node.js) or pre-commit framework (Python).

Common hooks:
| Hook         | When it runs                              |
|-------------|------------------------------------------|
| pre-commit   | Before commit object is created           |
| commit-msg   | After commit message is written           |
| pre-push     | Before push to remote                     |
| post-merge   | After a merge completes                   |
| post-checkout| After switching branches                  |

**Pre-commit hook to block TODO/FIXME:**
```bash
#!/bin/bash
if git diff --cached | grep -E '^\+.*TODO|^\+.*FIXME' > /dev/null; then
    echo "BLOCKED: Remove TODO/FIXME before committing"
    exit 1
fi
echo "Pre-commit check passed ✓"
exit 0
```

**Important gotcha:** Hook scans ALL staged content, not just new lines.
If TODO exists anywhere in any staged file → blocked.
You must remove TODO from the entire file, not just avoid adding it.

---

## Cherry-pick

Applies a single specific commit onto the current branch.
Creates a new commit object with a new hash (same changes, new parent pointer).

```bash
git cherry-pick <commit-hash>
```

**Conflict scenario:**
Cherry-pick conflicts when the file being modified doesn't exist on target branch.

```bash
# Resolution
git add <conflicted-file>
git cherry-pick --continue    # write commit message and save

# Or abort entirely
git cherry-pick --abort
```

**When to use:**
- Apply a hotfix from feature branch to production immediately
- Port a specific fix to multiple release branches
- Selectively bring one commit from an abandoned branch

---

## Important Gotchas

### 1. Git does not track empty folders
Only files are tracked. Use `.gitkeep` as a placeholder.
```bash
touch folder/.gitkeep
git add folder/.gitkeep
```

### 2. Windows line endings (CRLF warning)
Linux uses LF, Windows uses CRLF.
Git on Windows auto-converts and shows this warning.
In DevOps (Docker/Linux containers) this matters — configure:
```bash
git config --global core.autocrlf true    # on Windows (convert on checkout)
git config --global core.autocrlf input   # on Linux/Mac (convert on add)
```

### 3. Rebase changes commit hashes
Even if content is identical, a rebased commit gets a new hash
because its parent pointer changed. Never force-push rebased
commits to shared branches.

### 4. Default branch name
Older Git versions default to `master`. Modern standard is `main`.
```bash
git config --global init.defaultBranch main
```

### 5. Per-repo vs global Git config
```bash
git config --global user.name "name"    # applies to all repos
git config user.name "name"             # applies to this repo only
git config --list                       # see all active config
```

---

## Essential Commands Reference

```bash
# Object inspection
git cat-file -t <hash>              # type of object (blob/tree/commit/tag)
git cat-file -p <hash>              # contents of object
find .git/objects -type f           # all stored objects

# Branch internals
cat .git/HEAD                       # current branch pointer
cat .git/refs/heads/<branch>        # latest commit on branch
git log --oneline --graph --all     # visualize full repo history

# Rebase
git rebase <branch>                 # rebase current branch onto branch
git rebase -i HEAD~N                # interactive rebase last N commits

# Recovery
git reflog                          # complete HEAD movement history
git checkout -b <branch> <hash>     # recover deleted branch from hash

# Hooks
ls .git/hooks/                      # see available hook scripts
chmod +x .git/hooks/<hook>          # make hook executable

# Cherry-pick
git cherry-pick <hash>              # apply specific commit to current branch
git cherry-pick --continue          # continue after resolving conflict
git cherry-pick --abort             # cancel cherry-pick entirely

# Config
git config --global user.name "x"
git config --global user.email "x"
git config --global init.defaultBranch main
git config --list
```
