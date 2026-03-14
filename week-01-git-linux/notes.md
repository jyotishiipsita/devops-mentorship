# Week 1 — Git Internals + Linux Networking

## Date started: 2026-03-15

---

## Session 1 — Environment Setup

### What we did
- Generated SSH key pair on EC2 using `ssh-keygen -t ed25519`
- Added EC2 public key to GitHub (Settings → Access → SSH and GPG keys)
- Generated a separate SSH key pair on local laptop
- Added laptop public key to GitHub as a second authentication key
- Configured `~/.ssh/config` on both machines to point to correct key
- Cloned repo via SSH on both EC2 and laptop

### Key commands
```bash
ssh-keygen -t ed25519 -C "label"     # generate key pair
cat ~/.ssh/key.pub                    # view public key to add to GitHub
ssh -T git@github.com                 # test GitHub authentication
git remote set-url origin git@github.com:user/repo.git  # switch to SSH remote
git remote -v                         # verify remote URL
```

### Takeaways
- Every machine gets its OWN key pair — never copy private keys between machines
- Private key stays on your machine, public key goes to GitHub
- SSH is always preferred over HTTPS for DevOps work — no password prompts, works in automation
- `~/.ssh/config` is where you map which key to use for which host

---

## Session 2 — Git Internals

### What we did
- Explored Git's object model by inspecting `.git/objects`
- Understood that Git is a content-addressable filesystem, not just version control
- Learned the 4 Git object types

### The 4 Git object types
| Object | What it stores |
|--------|---------------|
| blob   | File contents |
| tree   | Directory listing (points to blobs + trees) |
| commit | Snapshot + parent hash + author + message |
| tag    | Named pointer to a commit |

### Key commands
```bash
find .git/objects -type f            # see all stored objects
git cat-file -t <hash>               # what TYPE is this object?
git cat-file -p <hash>               # what is INSIDE this object?
git log --oneline --graph --all      # visualize repo history
cat .git/refs/heads/main             # a branch is just a 40-char SHA file
cat .git/HEAD                        # points to current branch
```

### Takeaways
- A branch is literally a text file containing a 40-character commit hash — nothing more
- Every file version, directory, commit is stored as a SHA-1 hashed object
- Understanding this makes rebasing, merging and cherry-picking logical, not magical

---

## Session 3 — Git Advanced Operations

### What we did
- Practiced interactive rebase to clean up messy commits
- Simulated accidental branch deletion and recovered using reflog
- Wrote a pre-commit hook to block TODO/FIXME from being committed
- Practiced cherry-pick to apply a single commit across branches

### Key commands
```bash
git rebase main                      # replay current branch on top of main
git rebase -i HEAD~3                 # interactive rebase — squash/edit last 3 commits
git reflog                           # every HEAD movement ever — your undo history
git cherry-pick <hash>               # apply one specific commit to current branch
```

### Takeaways
- `git rebase -i` is how professionals keep clean, readable Git history
- Nothing in Git is truly deleted for 30 days — `reflog` is your safety net
- Git hooks are scripts in `.git/hooks/` that run automatically on Git events
- Pre-commit hooks are used in real teams to enforce code quality before anything is committed

---

## Session 4 — GitHub Integration + Important Git Gotcha

### What we did
- Created `devops-mentorship` public repo on GitHub (our portfolio repo)
- Set up folder structure: `week-01-git-linux/{labs,scripts,notes}`
- Made first commit and push from EC2
- Pulled repo on local laptop
- Discovered empty folders don't appear in GitHub

### The `.gitkeep` lesson — IMPORTANT
Git does NOT track empty folders. Only files are tracked.

**The fix:** Place an empty `.gitkeep` file inside any folder you want Git to track.
```bash
touch week-01-git-linux/labs/.gitkeep
touch week-01-git-linux/scripts/.gitkeep
git add .
git commit -m "chore: track empty folders with .gitkeep"
git push origin main
```

### Our workflow every session
```bash
# Start of session
git pull origin main

# End of session
git add .
git commit -m "week-01: describe what you did"
git push origin main
```

### Takeaways
- Git tracks FILES not FOLDERS — empty directories are invisible to Git
- `.gitkeep` is the standard DevOps convention to force-track empty folders
- Public repos = your portfolio — every commit tells your story to future employers
- Always pull before you start, always push before you stop

---

## Linux Networking
*(coming next)*

