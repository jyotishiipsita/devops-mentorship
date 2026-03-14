# Git Lab — Commands and Outputs
> Phase 1 | Week 1 | Session 2
> Date: 2026-03-15
> Environment: Windows laptop (Git Bash) + EC2 Ubuntu 24.04

---

## Lab Setup — SSH + GitHub Integration

```bash
# On EC2 — generate SSH key
ssh-keygen -t ed25519 -C "jyotishiipsita@github" -f ~/.ssh/github_devops

# View public key and add to GitHub (github.com/settings/keys)
cat ~/.ssh/github_devops.pub

# Configure SSH
cat >> ~/.ssh/config << 'EOF'
Host github.com
    HostName github.com
    User git
    IdentityFile ~/.ssh/github_devops
EOF

# Test authentication
ssh -T git@github.com
# → Hi jyotishiipsita! You've successfully authenticated,
#   but GitHub does not provide shell access.

# On Laptop (Git Bash) — generate separate key
ssh-keygen -t ed25519 -C "jyotishiipsita@laptop" -f ~/.ssh/github_laptop
cat ~/.ssh/github_laptop.pub   # add this to GitHub too

cat >> ~/.ssh/config << 'EOF'
Host github.com
    HostName github.com
    User git
    IdentityFile ~/.ssh/github_laptop
EOF

ssh -T git@github.com
# → Hi jyotishiipsita! You've successfully authenticated...
```

---

## Lab 1 — Git Object Model

### Initialize repo and make first commit

```bash
cd /D/devops-mentorship/week-01-git-linux/labs
git init git-object-lab
cd git-object-lab

echo "Hello DevOps" > hello.txt
git add hello.txt
git commit -m "first commit"
# → [master (root-commit) b8fee2f] first commit
#    1 file changed, 1 insertion(+)
#    create mode 100644 hello.txt
```

### Find all objects Git created

```bash
find .git/objects -type f
# Output:
# .git/objects/0f/4ea4723a733287d8f23a86e98c1c9f54927141
# .git/objects/80/706c4e7f7317c578caec40e9ba121fd3018a86
# .git/objects/b8/fee2f25f6f9734968498e34faf2827b90ff1a8
# Exactly 3 objects for 1 file + 1 commit
```

### Inspect the blob

```bash
git cat-file -t 0f4ea4723a733287d8f23a86e98c1c9f54927141
# → blob

git cat-file -p 0f4ea4723a733287d8f23a86e98c1c9f54927141
# → Hello DevOps
# Just raw file content — no filename, no metadata
```

### Inspect the tree

```bash
git cat-file -t 80706c4e7f7317c578caec40e9ba121fd3018a86
# → tree

git cat-file -p 80706c4e7f7317c578caec40e9ba121fd3018a86
# → 100644 blob 0f4ea4723a733287d8f23a86e98c1c9f54927141    hello.txt
# permissions + type + blob hash + filename
```

### Inspect the commit

```bash
git cat-file -t b8fee2f25f6f9734968498e34faf2827b90ff1a8
# → commit

git cat-file -p b8fee2f25f6f9734968498e34faf2827b90ff1a8
# → tree 80706c4e7f7317c578caec40e9ba121fd3018a86
# → author Sailendu Tripathy <email> 1773513842 +0530
# → committer Sailendu Tripathy <email> 1773513842 +0530
# →
# → first commit
# No parent line — this is the root commit
```

---

## Lab 2 — Branch Pointer Internals

```bash
cat .git/HEAD
# → ref: refs/heads/master

cat .git/refs/heads/master
# → b8fee2f25f6f9734968498e34faf2827b90ff1a8

git log --oneline
# → b8fee2f (HEAD -> master) first commit

# KEY OBSERVATION:
# refs/heads/master contains EXACTLY the same hash as the commit
# A branch is just a 40-character text file — nothing more
```

---

## Lab 3 — Parent Chain (Second Commit)

```bash
echo "Second line" >> hello.txt
git add hello.txt
git commit -m "second commit"
# → [master ee43164] second commit

git cat-file -p ee43164
# → tree 65417f79b47f55c167e791c85892e4d59751b21d   ← NEW tree
# → parent b8fee2f25f6f9734968498e34faf2827b90ff1a8  ← NEW! points to first commit
# → author jyotishiipsita <email> 1773516197 +0530
# → committer jyotishiipsita <email> 1773516197 +0530
# →
# → second commit

# Trace the new tree
git cat-file -p 65417f79b47f55c167e791c85892e4d59751b21d
# → 100644 blob f4b2c947600436f8e1007dd42974c136793dba1c    hello.txt

# Trace the new blob
git cat-file -p f4b2c947600436f8e1007dd42974c136793dba1c
# → Hello DevOps
# → Second line

# Old blob still exists untouched
git cat-file -p 0f4ea4723a733287d8f23a86e98c1c9f54927141
# → Hello DevOps
# Git never deletes old objects — append-only storage
```

---

## Lab 4 — Standard Rebase

### Create diverged branch scenario

```bash
git branch -m master main    # rename to main

# Main moves forward
echo "Production config v1" > config.txt
git add config.txt && git commit -m "main: add production config"
# → [main 82e2510]

echo "Production config v2" > config.txt
git add config.txt && git commit -m "main: update production config"
# → [main a542aac]

# Create feature branch from 2 commits ago
git checkout -b feature/add-logging HEAD~2
# → Switched to a new branch 'feature/add-logging'

ls
# → hello.txt   (config.txt is NOT here — we branched before it was added)

echo "logging: enabled=true" > logging.txt
git add logging.txt && git commit -m "feature: add logging config"
# → [feature/add-logging fb5479f]
```

### Visualize the diverged history

```bash
git log --oneline --graph --all
# → * fb5479f (HEAD -> feature/add-logging) feature: add logging config
# → | * a542aac (main) main: update production config
# → | * 82e2510 main: add production config
# → |/
# → * ee43164 second commit
# → * b8fee2f first commit
# Branches clearly diverged at ee43164
```

### Perform the rebase

```bash
git rebase main
# → Successfully rebased and updated refs/heads/feature/add-logging.

git log --oneline --graph --all
# → * ccef211 (HEAD -> feature/add-logging) feature: add logging config
# → * a542aac (main) main: update production config
# → * 82e2510 main: add production config
# → * ee43164 second commit
# → * b8fee2f first commit
# History is now perfectly linear!

# KEY OBSERVATION:
# Before rebase: fb5479f  feature: add logging config
# After rebase:  ccef211  feature: add logging config
# SAME message, SAME changes — but DIFFERENT hash!
# Because parent pointer changed, the entire commit object changed.
```

---

## Lab 5 — Interactive Rebase (Squash WIP commits)

```bash
# Create 3 messy WIP commits
echo "fix attempt 1" >> logging.txt
git add logging.txt && git commit -am "wip"

echo "fix attempt 2" >> logging.txt
git add logging.txt && git commit -am "wip again"

echo "final fix" >> logging.txt
git add logging.txt && git commit -am "ok this time for real"

git log --oneline
# → 2df21c1 ok this time for real
# → 2d69f21 wip again
# → c455563 wip
# → ccef211 feature: add logging config
# → ... (rest of history)

# Clean them up into one professional commit
git rebase -i HEAD~3
# Editor opens showing:
# pick c455563 wip
# pick 2d69f21 wip again
# pick 2df21c1 ok this time for real
#
# Change to:
# pick c455563 wip
# squash 2d69f21 wip again
# squash 2df21c1 ok this time for real
#
# Save, then write final message:
# "feature: improve logging config with retry logic"

git log --oneline
# → 6d46bea (HEAD -> feature/add-logging) feature: improve logging config with retry logic
# → ccef211 feature: add logging config
# → a542aac (main) main: update production config
# 3 messy commits → 1 clean professional commit ✓
```

---

## Lab 6 — reflog Recovery

```bash
# Simulate accidental branch deletion
git checkout main
git branch -D feature/add-logging
# → Deleted branch feature/add-logging (was 6d46bea).

git branch
# → * main   (feature/add-logging is gone)

git log --oneline --graph --all
# → * a542aac (HEAD -> main) main: update production config
# → * 82e2510 main: add production config
# → * ee43164 second commit
# → * b8fee2f first commit
# (feature work appears completely lost)

# Use reflog to recover
git reflog | head -20
# → a542aac HEAD@{0}: checkout: moving from feature/add-logging to main
# → 6d46bea HEAD@{1}: rebase (finish): returning to refs/heads/feature/add-logging
# → 6d46bea HEAD@{2}: rebase (squash): feature: improve logging config...
# → 1f9077a HEAD@{3}: rebase (squash): # This is a combination of 2 commits.
# → 2df21c1 HEAD@{4}: rebase (start): checkout HEAD~3
# → c455563 HEAD@{5}: commit: ok this time for real
# → 2d69f21 HEAD@{6}: commit: wip again
# → 2df21c1 HEAD@{7}: commit: wip
# → ccef211 HEAD@{8}: rebase (finish): returning to refs/heads/feature/add-logging
# → fb5479f HEAD@{11}: commit: feature: add logging config  ← even pre-rebase hash!
# → ee43164 HEAD@{12}: checkout: moving from main to feature/add-logging
# → ee43164 HEAD@{15}: Branch: renamed refs/heads/master to refs/heads/main
# → b8fee2f HEAD@{18}: commit (initial): first commit

# Recover using the hash from reflog
git checkout -b feature/add-logging 6d46bea
# → Switched to a new branch 'feature/add-logging'

git log --oneline
# → 6d46bea (HEAD -> feature/add-logging) feature: improve logging config with retry logic
# → ccef211 feature: add logging config
# → a542aac (main) main: update production config
# Full history recovered — nothing lost ✓
```

---

## Lab 7 — Pre-commit Hook

```bash
# Create the hook
cat > .git/hooks/pre-commit << 'EOF'
#!/bin/bash
if git diff --cached | grep -E '^\+.*TODO|^\+.*FIXME' > /dev/null; then
    echo "BLOCKED: Remove TODO/FIXME before committing"
    exit 1
fi
echo "Pre-commit check passed ✓"
exit 0
EOF

chmod +x .git/hooks/pre-commit

# Test 1 — BLOCKED (added TODO)
echo "TODO: fix this later" >> logging.txt
git add logging.txt
git commit -m "test blocked commit"
# → warning: LF will be replaced by CRLF...
# → BLOCKED: Remove TODO/FIXME before committing ✓

# Test 2 — STILL BLOCKED (appended clean line but TODO still in file)
echo "this is properly fixed" >> logging.txt
git add logging.txt
git commit -m "test passing commit"
# → BLOCKED: Remove TODO/FIXME before committing
# LESSON: Hook scans ALL staged content, not just the new line

# Test 3 — PASSED (overwrote file, removing TODO entirely)
echo "this is a new line not appended" > logging.txt   # note: > not >>
git add logging.txt
git commit -m "test passing commit for test 4"
# → Pre-commit check passed ✓
# → [feature/add-logging b7025e8] test passing commit for test 4 ✓
```

---

## Lab 8 — Cherry-pick + Conflict Resolution

```bash
git checkout main

# Attempt cherry-pick of b7025e8 (modified logging.txt)
git cherry-pick b7025e8
# → CONFLICT (modify/delete): logging.txt deleted in HEAD
#   and modified in b7025e8...
# → error: could not apply b7025e8...
# REASON: logging.txt doesn't exist on main branch
# Git doesn't know what to do — conflict!

ls
# → config.txt  hello.txt  logging.txt
# (Git left the conflicted file in the working tree)

cat logging.txt
# → this is a new line not appended

# Resolve the conflict — accept the file
git add logging.txt
git cherry-pick --continue
# Write commit message: "cherry-pick: add logging config to main"
# Save and close editor

git log --oneline
# → <new hash>  cherry-pick: add logging config to main
# → a542aac (main) main: update production config
# → 82e2510 main: add production config
# Commit successfully applied to main ✓
```

---

## Cleanup — Push lab to GitHub

```bash
cd /D/devops-mentorship

# Add .gitkeep to empty folders so Git tracks them
touch week-01-git-linux/labs/.gitkeep
touch week-01-git-linux/scripts/.gitkeep
touch week-01-git-linux/notes/.gitkeep

git add .
git commit -m "chore: track empty lab folders with .gitkeep"
git push origin main
# All lab work now visible on github.com/jyotishiipsita/devops-mentorship
```
