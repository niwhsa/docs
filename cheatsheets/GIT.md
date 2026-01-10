# Git Cheatsheet

Git commands and multi-account setup reference.

## Multi-Account Setup

### Directory Structure
```
~/cursor/      → Work (git.ouryahoo.com)
~/git/         → Work (git.ouryahoo.com)
~/personal/    → Personal (github.com/niwhsa)
```

### SSH Config (`~/.ssh/config`)
```ssh-config
# Work - Yahoo Git
Host git.ouryahoo.com
  User git
  IdentityFile ~/.ssh/id_rsa

# Personal - GitHub
Host github.com
  User git
  IdentityFile ~/.ssh/id_ed25519_niwhsa_account
  IdentitiesOnly yes
```

### Git Identity (`~/.gitconfig`)
```gitconfig
[user]
    name = Ashwin Suresh
    email = ashwin.suresh@yahooinc.com  # default

[includeIf "gitdir:~/personal/"]
    path = ~/.gitconfig-personal
```

### Personal Config (`~/.gitconfig-personal`)
```gitconfig
[user]
    name = Ashwin Suresh
    email = ashwin.ms@gmail.com
```

### Generate SSH Keys
```bash
# Generate new key
ssh-keygen -t ed25519 -C "your@email.com" -f ~/.ssh/id_ed25519_keyname

# Show public key (copy to GitHub/GitLab settings)
cat ~/.ssh/id_ed25519_keyname.pub

# Add key to SSH agent
ssh-add ~/.ssh/id_ed25519_keyname

# Clear all keys from agent
ssh-add -D

# List keys in agent
ssh-add -l
```

### Verify Setup
```bash
# Check which identity is used
cd ~/personal/leetcode && git config user.email  # → ashwin.ms@gmail.com
cd ~/cursor/udping && git config user.email      # → ashwin.suresh@yahooinc.com

# Test SSH connection
ssh -T git@github.com          # → Hi niwhsa!
ssh -T git@git.ouryahoo.com    # → Welcome to Yahoo Git
```

### Find All Repos by Remote
```bash
# Find all repos pointing to a specific host
for dir in ~/personal/*/; do
  if [ -d "$dir/.git" ]; then
    echo -n "$(basename $dir) → "
    git -C "$dir" remote -v | grep fetch | awk '{print $2}'
  fi
done

# Search for repos with specific remote pattern
find ~ -maxdepth 4 -name ".git" -type d 2>/dev/null | while read gitdir; do
  remote=$(git -C "${gitdir%/.git}" remote -v 2>/dev/null | grep github.com | head -1)
  if [ -n "$remote" ]; then
    echo "${gitdir%/.git}: $remote"
  fi
done
```

## Daily Commands

### Status & Info
```bash
git status              # Current state
git log --oneline -10   # Recent commits
git log --graph --oneline --all  # Visual branch history
git diff                # Unstaged changes
git diff --staged       # Staged changes
git branch -a           # All branches
git remote -v           # Remote URLs
```

### Staging & Committing
```bash
git add <file>          # Stage specific file
git add -A              # Stage all changes
git add -p              # Interactive staging (patch mode)

git commit -m "message" # Commit with message
git commit --amend      # Modify last commit
git commit --amend --no-edit  # Amend without changing message
```

### Branches
```bash
git branch <name>       # Create branch
git checkout <name>     # Switch branch
git checkout -b <name>  # Create and switch
git branch -d <name>    # Delete branch (safe)
git branch -D <name>    # Delete branch (force)
git branch -m <new>     # Rename current branch
```

### Remote Operations
```bash
git fetch               # Download remote changes
git pull                # Fetch + merge
git push                # Push to remote
git push -u origin main # Set upstream and push
git push --force-with-lease  # Safe force push
```

### Merging & Rebasing
```bash
git merge <branch>      # Merge branch into current
git rebase <branch>     # Rebase current onto branch
git rebase -i HEAD~3    # Interactive rebase last 3 commits
git merge --abort       # Abort merge
git rebase --abort      # Abort rebase
```

### Undoing Changes
```bash
git checkout -- <file>  # Discard unstaged changes
git restore <file>      # Same as above (newer syntax)
git reset HEAD <file>   # Unstage file
git restore --staged <file>  # Same as above (newer syntax)

git reset --soft HEAD~1   # Undo commit, keep changes staged
git reset --mixed HEAD~1  # Undo commit, keep changes unstaged
git reset --hard HEAD~1   # Undo commit, discard changes

git revert <commit>     # Create new commit that undoes changes
```

### Stashing
```bash
git stash               # Stash changes
git stash pop           # Apply and remove stash
git stash apply         # Apply but keep stash
git stash list          # List stashes
git stash drop          # Remove top stash
git stash clear         # Remove all stashes
```

### Cherry-Pick
```bash
git cherry-pick <commit>        # Apply commit to current branch
git cherry-pick -x <commit>     # With reference to original
git cherry-pick --abort         # Abort cherry-pick
```

## Useful Aliases

Add to `~/.gitconfig`:
```gitconfig
[alias]
    st = status
    co = checkout
    br = branch
    ci = commit
    lg = log --oneline --graph --all
    last = log -1 HEAD
    unstage = reset HEAD --
    undo = reset --soft HEAD~1
```

## Clone Commands

```bash
# Work (Yahoo)
git clone git@git.ouryahoo.com:netcloud/udping.git

# Personal (GitHub)
git clone git@github.com:niwhsa/leetcode.git
```

## Fix Common Issues

### Wrong email on commits
```bash
# Check current email
git config user.email

# Fix for this repo only
git config user.email "correct@email.com"

# Amend last commit with correct email
git commit --amend --reset-author --no-edit
```

### Change remote URL
```bash
# HTTPS to SSH
git remote set-url origin git@github.com:user/repo.git

# View current
git remote -v
```

### Undo pushed commit
```bash
# Create reverting commit (safe)
git revert HEAD
git push

# Or force push (destructive, use with caution)
git reset --hard HEAD~1
git push --force-with-lease
```

### Resolve merge conflicts
```bash
# After conflict:
# 1. Edit files to resolve
# 2. Stage resolved files
git add <file>
# 3. Complete merge
git commit
```

### Clean untracked files
```bash
git clean -n            # Dry run (show what would be deleted)
git clean -f            # Delete untracked files
git clean -fd           # Delete untracked files and directories
```

## .gitignore Patterns

```gitignore
# OS files
.DS_Store
Thumbs.db

# IDEs
.idea/
.vscode/
*.swp

# Dependencies
node_modules/
venv/
.venv/
target/

# Build outputs
build/
dist/
*.o
*.exe

# Secrets
.env
*.pem
*.key
```

