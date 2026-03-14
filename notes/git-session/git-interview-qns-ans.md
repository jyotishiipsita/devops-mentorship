# Git — Interview Questions and Answers
> Phase 1 | Week 1 | Session 2
> Date: 2026-03-15
> 20 questions — Beginner to Advanced + Scenario-based

---

## Beginner Level

**Q1. What is Git and how is it different from other version control systems?**

Git is a distributed version control system, but more fundamentally it is a
content-addressable filesystem. Unlike centralized systems (SVN, CVS) where
history lives on a central server, every Git developer has a complete copy
of the repository including full history locally. Git identifies and stores
all data by the SHA-1 hash of its content, making it extremely reliable,
fast, and tamper-proof. Operations like log, diff, and branch are all local
— no network required.

---

**Q2. What are the four Git object types?**

- **Blob** — stores raw file contents (no filename, no metadata)
- **Tree** — stores directory listing (permissions + type + hash + filename)
- **Commit** — stores snapshot pointer (tree hash), parent hash, author, message
- **Tag** — named pointer to a specific commit

Everything in Git is stored as one of these four types, all identified by
their SHA-1 content hash.

---

**Q3. What is the difference between `git merge` and `git rebase`?**

Both integrate changes from one branch into another, but differently:

| | Merge | Rebase |
|-|-------|--------|
| History | Preserves full history, adds merge commit | Rewrites history, produces linear result |
| Commits | Original commits unchanged, new merge commit | New commit objects created with new hashes |
| Use case | Public/shared branches, preserving context | Private feature branches before PR |
| Safety | Always safe — non-destructive | Never on shared branches — rewrites history |

Rebase replays commits as new objects on top of the target branch tip.
Merge creates a new merge commit tying two branch histories together.

---

**Q4. What is HEAD in Git?**

HEAD is a special pointer that tells Git where you currently are.
It lives in `.git/HEAD` and normally contains a branch reference:
```
ref: refs/heads/main
```
When you checkout a specific commit hash instead of a branch name,
HEAD becomes "detached" — it points directly to a commit hash.
Any new commits in detached HEAD state are not on any branch and
will eventually be garbage collected unless saved to a branch.

---

**Q5. What is the staging area (index) in Git?**

The staging area (also called the index) is a middle layer between
the working directory and the repository object store.
`git add` moves changes into staging. `git commit` takes everything
currently staged and creates a commit object from it.
This design allows crafting precise commits — staging only the changes
relevant to one logical unit of work, even if multiple files were modified.

---

**Q6. What happens when you run `git add`?**

`git add` does two things:
1. Creates a **blob object** in `.git/objects/` for the file content
2. Updates the **index** (staging area) to reference that blob

The blob is created immediately on `git add` — not on `git commit`.
This is why Git can detect staged vs unstaged changes separately.

---

## Intermediate Level

**Q7. What does `git rebase -i` do and when would you use it?**

Interactive rebase lets you modify commits before they become part of
shared history. It opens an editor showing recent commits with options:
- **squash** — merge multiple WIP commits into one clean commit
- **reword** — fix a poorly written commit message
- **drop** — remove a commit that shouldn't exist
- **reorder** — rearrange commits for logical grouping
- **edit** — pause at a commit to amend it

Primary use case: clean up messy development commits before raising
a Pull Request so the team sees a clean, professional history.
Rule: always run on your own feature branches, never on main.

---

**Q8. A developer accidentally deleted a branch. How do you recover it?**

Use `git reflog` — it records every HEAD movement for at least 30 days:
```bash
git reflog
# Find the hash of the last commit on the deleted branch
git checkout -b recovered-branch <hash>
```
Nothing in Git is truly deleted immediately. Objects remain in the store
until garbage collection runs (default: unreachable objects kept 30+ days).
The reflog itself is kept for 90 days by default.

---

**Q9. What is the difference between `git reset`, `git revert`, and `git restore`?**

| Command | What it does | Rewrites history? | Safe for shared branches? |
|---------|-------------|-------------------|--------------------------|
| git reset | Moves branch pointer backward | Yes | No |
| git revert | Creates new commit undoing a previous one | No | Yes |
| git restore | Discards working directory changes | No (local only) | N/A |

In production: always use `git revert` to undo changes on shared branches.
`git reset` is for local/feature branches only.
`git restore` is purely for discarding uncommitted local changes.

---

**Q10. What is a Git hook and how are they used in DevOps pipelines?**

Git hooks are executable scripts in `.git/hooks/` that run automatically
at specific points in the Git workflow. In DevOps they enforce quality gates:

- **pre-commit** — run linters, block TODO/FIXME, check secrets, enforce formatting
- **commit-msg** — enforce conventional commit format (feat:/fix:/chore:)
- **pre-push** — run unit tests before allowing push to remote
- **post-merge** — auto-install dependencies, rebuild assets after merge

Important: `.git/hooks/` is not tracked by Git itself.
For team-wide enforcement use Husky (Node.js projects) or the
pre-commit framework (Python/any language).

---

**Q11. Explain what happens internally when you run `git commit`.**

1. Git reads all staged files from the index
2. Creates **blob objects** for any new/changed file content
3. Creates a **tree object** representing the full directory snapshot
4. Creates a **commit object** containing:
   - Hash of the tree
   - Hash of the parent commit
   - Author and committer (name, email, Unix timestamp, timezone)
   - Commit message
5. Updates the current branch file in `.git/refs/heads/` to the new commit hash
6. HEAD continues pointing to the same branch name (which now has a new hash)

---

## Advanced Level

**Q12. Why does rebasing change commit hashes even when the content is identical?**

A Git commit hash (SHA-1) is computed from ALL of its contents combined:
tree hash + parent hash + author + committer + timestamp + message.

When you rebase, the parent pointer changes to the new base commit.
Even if the file changes are byte-for-byte identical, the different
parent hash produces a completely different SHA-1 output.
This is fundamental to how cryptographic hashing works — any change
to the input, no matter how small, produces a completely different hash.
This is why rebasing is considered "rewriting history."

---

**Q13. What is the difference between `git fetch` and `git pull`?**

`git fetch` downloads changes from the remote into remote-tracking branches
(origin/main, origin/feature) but does NOT modify your local branches.
It is always safe and non-destructive — you can inspect before integrating.

`git pull` is `git fetch` + `git merge` (or `git fetch` + `git rebase` with --rebase).
It immediately integrates remote changes into your current branch.

Best practice in teams:
```bash
git fetch origin
git log origin/main..main      # see what's different
git merge origin/main          # integrate when ready
```
Never `git pull` blindly on a branch others are actively pushing to.

---

**Q14. What is cherry-pick and what are its risks?**

Cherry-pick applies a specific commit onto the current branch,
creating a new commit object with the same diff but a new hash
(because the parent pointer is different).

Use cases:
- Hotfix on feature branch needs to go to production immediately
- Port a specific bug fix to multiple release branches (v1.0, v1.1, v2.0)

Risks:
- Creates duplicate history — same logical change appears multiple times
- Conflicts likely if the surrounding code context differs between branches
- Overusing cherry-pick signals a branching strategy problem
- If the original branch is later merged, the cherry-picked commit may conflict

---

**Q15. What is a detached HEAD state and how do you get out of it?**

Detached HEAD occurs when HEAD points directly to a commit hash
instead of a branch name — typically from `git checkout <hash>`.

```
Normal:    HEAD → refs/heads/main → a542aac
Detached:  HEAD → a542aac (directly, no branch)
```

Any commits made in detached HEAD are not on any branch.
They will be garbage collected eventually (after 30 days from reflog expiry).

Recovery:
```bash
# Save work by creating a branch at current position
git checkout -b save-my-work

# Or go back without saving
git checkout main
```

---

**Q16. How does Git store history efficiently without full copies of every version?**

Git uses multiple strategies:

1. **Content addressing** — identical content shares one blob object
   (a file unchanged across 100 commits = 1 blob, not 100)

2. **Packfiles** — Git periodically packs loose objects into `.pack` files
   using delta compression (stores diffs between similar objects)

3. **Lazy storage** — objects are created only when needed (`git add` creates
   blobs, `git commit` creates trees and commits — not before)

4. **Shallow clones** — `git clone --depth=1` fetches only recent history,
   critical for CI/CD pipeline performance

---

## Tricky / Scenario Based

**Q17. Your team member pushed directly to main by mistake. How do you fix it?**

Never use `git reset --hard` on a shared branch — it rewrites history
and breaks everyone else's local copy when they pull.

Correct approach — use `git revert`:
```bash
git revert <bad-commit-hash>
git push origin main
```
This creates a new commit that undoes the bad commit, preserving full history.
Everyone on the team can safely pull without any conflict or force-push.

---

**Q18. You need to apply a hotfix to 3 different release branches. What is your approach?**

1. Create the fix on a hotfix branch off main, test thoroughly
2. Cherry-pick the single fix commit onto each release branch:
```bash
git checkout release/v1.0
git cherry-pick <fix-hash>
git push origin release/v1.0

git checkout release/v1.1
git cherry-pick <fix-hash>
git push origin release/v1.1

git checkout release/v2.0
git cherry-pick <fix-hash>
git push origin release/v2.0
```
3. Tag each release after applying: `git tag v1.0.1 && git push --tags`
4. Document the cherry-pick in your incident log

---

**Q19. What is `git bisect` and when would a DevOps engineer use it?**

`git bisect` performs a binary search through commit history to find
the exact commit that introduced a regression. It halves the search
space with each step — finds the bad commit in O(log n) steps.

```bash
git bisect start
git bisect bad HEAD           # current HEAD is broken
git bisect good v2.3.0        # this release was working fine

# Git checks out the middle commit automatically
# You test: does the bug exist?
git bisect good    # or: git bisect bad

# Repeat until Git identifies the exact breaking commit
# Output: "a1b2c3d is the first bad commit"

git bisect reset              # return to original HEAD
```

DevOps use case: production regression where you have 200 commits
between the last good release and current broken state.
Bisect finds it in ~8 steps instead of checking all 200.

---

**Q20. What is a shallow clone and why is it important in CI/CD?**

A shallow clone fetches only a limited commit history:
```bash
git clone --depth=1 https://github.com/org/repo.git
```

Why it matters in CI/CD:
- Build agents are ephemeral — they clone fresh on every pipeline run
- Full clone of a large repo (Linux kernel = 4GB+) would be very slow
- CI only needs the code, not 10 years of history
- GitHub Actions uses `fetch-depth: 1` by default for this reason

```yaml
# GitHub Actions example
- uses: actions/checkout@v4
  with:
    fetch-depth: 1    # shallow clone — fastest option
    # fetch-depth: 0  # full clone — needed for git log, versioning tools
```

Limitations of shallow clones:
- `git bisect` doesn't work (needs full history)
- `git log --follow` for renamed files may be inaccurate
- Some versioning tools (semantic-release, gitversion) need full history
  — use `fetch-depth: 0` for those specific jobs
