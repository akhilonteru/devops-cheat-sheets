# Git Cheat Sheet

> Essential Git commands for DevOps engineers — configuration, branching, merging, rebasing, stashing, remotes, tags, and undoing mistakes.

---

## Table of Contents

- [Setup & Configuration](#1-setup--configuration)
- [Creating & Cloning](#2-creating--cloning)
- [Staging & Committing](#3-staging--committing)
- [Branching](#4-branching)
- [Merging & Rebasing](#5-merging--rebasing)
- [Stashing](#6-stashing)
- [Remotes — Push, Pull, Fetch](#7-remotes--push-pull-fetch)
- [Tags & Releases](#8-tags--releases)
- [History & Inspection](#9-history--inspection)
- [Undoing Changes](#10-undoing-changes)
- [Useful Aliases & Config](#11-useful-aliases--config)

---

## 1. Setup & Configuration

```
git config --global user.name "John Doe"
git config --global user.email "john@example.com"
git config --global init.defaultBranch main
git config --global core.editor vim
git config --global credential.helper store        # Cache credentials
git config --global pull.rebase true               # Pull with rebase by default
git config --list                                  # Show all config
```

## 2. Creating & Cloning

```
git init                              # Initialize repository
git clone https://github.com/user/repo.git
git clone git@github.com:user/repo.git repo-dir     # Clone into specific dir
git clone --branch develop --depth 1 URL            # Shallow clone of a branch
```

## 3. Staging & Committing

```
git status                            # Working tree status
git status -sb                        # Short + branch info
git add file.txt                      # Stage file
git add .                             # Stage all changes
git add -p                            # Interactive staging (hunks)
git commit -m "Add login feature"     # Commit
git commit -am "Fix typo"             # Stage tracked files + commit
git commit --amend                    # Amend last commit
git commit --amend --no-edit          # Amend keeping message
git rm --cached file.txt              # Untrack file, keep on disk
git mv old.txt new.txt                # Rename + stage
```

## 4. Branching

```
git branch                            # List local branches
git branch -a                         # List all (local + remote)
git branch -d feature-x               # Delete merged branch
git branch -D feature-x               # Force delete
git switch main                       # Switch branch
git switch -c feature-x               # Create + switch
git checkout main                     # (older syntax) switch
git checkout -b feature-x             # (older syntax) create + switch
git branch --show-current             # Print current branch name
git branch -m old-name new-name       # Rename branch
```

## 5. Merging & Rebasing

```
git switch main
git merge feature-x                   # Merge feature into main
git merge --no-ff feature-x           # Always create merge commit
git merge --abort                     # Abort conflicted merge

git rebase main                       # Rebase current branch onto main
git rebase -i HEAD~3                  # Interactive rebase last 3 commits
# In interactive mode: pick / reword / squash / fixup / drop / edit

git cherry-pick <commit>              # Apply a specific commit to current branch
```

## 6. Stashing

```
git stash push -m "work in progress"  # Save changes
git stash                             # Quick stash
git stash list                        # List stashes
git stash pop                         # Apply + drop latest
git stash apply stash@{1}             # Apply specific stash
git stash drop stash@{0}              # Drop stash
git stash show -p stash@{0}           # Show stash diff
git stash branch feature-x            # New branch from stash
```

## 7. Remotes — Push, Pull, Fetch

```
git remote -v                         # List remotes
git remote add origin git@github.com:user/repo.git
git remote remove origin
git remote set-url origin NEW_URL     # Change remote URL

git fetch origin                      # Download remote changes (no merge)
git pull origin main                  # Fetch + merge
git pull --rebase origin main         # Fetch + rebase

git push origin main                  # Push branch
git push -u origin feature-x          # Push + set upstream
git push --tags                       # Push all tags
git push origin --delete feature-x    # Delete remote branch
git push -f origin main               # Force push (DANGER — shared branches)
```

## 8. Tags & Releases

```
git tag                               # List tags
git tag v1.0.0                        # Lightweight tag
git tag -a v1.0.0 -m "Release 1.0.0"  # Annotated tag
git tag -a v1.0.0 <commit>            # Tag specific commit
git push origin v1.0.0                # Push single tag
git push --tags                       # Push all tags
git tag -d v1.0.0                     # Delete local tag
git push origin :refs/tags/v1.0.0     # Delete remote tag
git describe --tags                   # Nearest tag description
```

## 9. History & Inspection

```
git log --oneline --graph --decorate --all   # Pretty full history
git log --oneline -10                        # Last 10 commits
git log --author="John" --since="1 week ago"
git log --stat                               # Show changed files
git show <commit>                            # Show commit diff
git diff                                     # Unstaged changes
git diff --staged                            # Staged changes
git diff main..feature-x                     # Branch comparison
git blame file.txt                           # Line-by-line authorship
git shortlog -sn                             # Commits per author
git reflog                                   # All HEAD movements (rescue tool)
```

## 10. Undoing Changes

```
git restore file.txt                  # Discard unstaged changes
git restore --staged file.txt         # Unstage file
git checkout -- file.txt              # (older syntax) discard changes

git reset --soft HEAD~1               # Undo commit, keep staged
git reset --mixed HEAD~1              # Undo commit, keep working tree (default)
git reset --hard HEAD~1               # Undo commit + discard changes (DANGER)
git reset --hard origin/main          # Match remote (DANGER)

git revert <commit>                   # Create inverse commit (safe for shared history)
git clean -fd                         # Remove untracked files/dirs (DANGER)
git clean -fdn                        # Dry run first!
```

## 11. Useful Aliases & Config

```
git config --global alias.st status
git config --global alias.co checkout
git config --global alias.br branch
git config --global alias.lg "log --oneline --graph --decorate --all"
git config --global alias.unstage "restore --staged --"
```

**.gitignore essentials:**
```
node_modules/
*.log
.env
dist/
.terraform/
*.pem
.DS_Store
```

### Typical Daily Flow

```bash
git switch -c feature-x
# ... edit files ...
git add .
git commit -m "Add feature x"
git fetch origin
git rebase origin/main        # keep history linear
git push -u origin feature-x
# ... open pull request ...
```

---
