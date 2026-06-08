# Git — The Complete Field Guide

> "Git is not hard. Git's data model is simple. What's hard is using Git the wrong way."

---

## Table of Contents

1. [Git Internals](#git-internals)
2. [Configuration](#configuration)
3. [Core Workflow](#core-workflow)
4. [Branching](#branching)
5. [Merging vs Rebasing](#merging-vs-rebasing)
6. [Stashing](#stashing)
7. [Remote Operations](#remote-operations)
8. [Undoing Things](#undoing-things)
9. [History & Inspection](#history--inspection)
10. [Bisect](#bisect)
11. [Reflog — The Safety Net](#reflog--the-safety-net)
12. [Tags](#tags)
13. [Worktrees](#worktrees)
14. [Submodules](#submodules)
15. [Hooks](#hooks)
16. [Advanced Techniques](#advanced-techniques)
17. [Aliases Worth Having](#aliases-worth-having)
18. [gitconfig Reference](#gitconfig-reference)

---

## Git Internals

Understanding the object model saves you from confusion later.

### The Four Object Types
```
blob    — file contents (no filename, just bytes)
tree    — directory listing (filenames + mode + blob/tree refs)
commit  — snapshot (tree + author + message + parent refs)
tag     — named reference to a commit
```

### The Three Areas
```
Working Directory  →  Staging Area (Index)  →  Repository (.git)
     edit                  git add                git commit
```

### References
```
HEAD        → current branch (or detached commit)
branch      → mutable pointer to a commit
tag         → immutable pointer to a commit
origin/main → remote-tracking branch
```

### Viewing internals
```bash
git cat-file -t HEAD            # type of HEAD object
git cat-file -p HEAD            # contents of HEAD
git ls-tree HEAD                # tree at HEAD
git ls-files --stage            # what's in the index
git show HEAD:path/to/file      # file at specific commit
```

---

## Configuration

```bash
# Identity
git config --global user.name "Your Name"
git config --global user.email "you@example.com"

# Editor
git config --global core.editor "vim"
git config --global core.editor "code --wait"     # VS Code

# Default branch
git config --global init.defaultBranch main

# Pull strategy
git config --global pull.rebase true              # prefer rebase over merge

# Push strategy
git config --global push.default current          # push to same-name remote branch
git config --global push.autoSetupRemote true     # auto set upstream on push

# Diff tools
git config --global diff.tool vimdiff
git config --global merge.tool vimdiff

# Aliases
git config --global alias.st status
git config --global alias.lg "log --oneline --graph --decorate --all"

# Credential helper
git config --global credential.helper store        # store credentials (less secure)
git config --global credential.helper osxkeychain  # macOS keychain

# Core settings
git config --global core.autocrlf input            # LF on commit (Linux/macOS)
git config --global core.autocrlf true             # CRLF on checkout, LF on commit (Windows)
```

---

## Core Workflow

### Staging
```bash
git status
git add file.txt                  # stage specific file
git add .                         # stage all
git add -p                        # stage interactively (patch mode) ← USE THIS MORE
git add -N file.txt               # intent-to-add (shows in diff untracked)

git diff                          # unstaged changes
git diff --staged                 # staged changes (about to commit)
git diff HEAD                     # all changes since last commit
```

### Committing
```bash
git commit -m "message"
git commit                        # opens editor (use for multi-line)
git commit -a -m "message"        # stage tracked files + commit
git commit --amend                # amend last commit (message or files)
git commit --amend --no-edit      # amend last commit (keep message)
git commit --fixup HEAD~1         # mark as fixup for rebase --autosquash
```

### Good Commit Messages
```
<type>(<scope>): <short summary>

Types: feat, fix, docs, style, refactor, test, chore, perf
Max 50 chars in subject. Blank line before body. Body: explain WHY not WHAT.

Examples:
  fix(auth): handle expired JWT tokens gracefully
  feat(api): add pagination to /users endpoint
  refactor: extract email validation to shared util
```

---

## Branching

```bash
git branch                        # list local branches
git branch -a                     # list all (local + remote)
git branch -v                     # show last commit on each
git branch --merged               # branches merged into HEAD
git branch --no-merged            # branches NOT merged

git branch feature/login          # create branch
git switch feature/login          # switch to branch
git switch -c feature/login       # create + switch (modern)
git checkout -b feature/login     # create + switch (classic)

git branch -d feature/login       # delete (safe — must be merged)
git branch -D feature/login       # delete (force)
git push origin --delete feature  # delete remote branch

git branch -m old-name new-name   # rename branch
git branch -M main                # force rename (if main exists)
```

---

## Merging vs Rebasing

### Merge
```bash
git merge feature                 # merge feature into current branch
git merge --no-ff feature         # always create merge commit
git merge --squash feature        # squash to single commit (no merge commit)
git merge --abort                 # abort in-progress merge
```

### Rebase
```bash
git rebase main                   # rebase current branch onto main
git rebase --onto main old-base   # rebase onto main, cut from old-base
git rebase -i HEAD~5              # interactive rebase last 5 commits
git rebase --abort                # abort
git rebase --continue             # continue after resolving conflicts
git rebase --skip                 # skip current commit
```

### Interactive Rebase Commands
```
pick   p    use commit as-is
reword r    use commit, but edit the message
edit   e    pause here to amend the commit
squash s    meld into previous commit
fixup  f    meld into previous commit (discard message)
drop   d    remove commit entirely
exec   x    run shell command
break  b    pause here (continue with --continue)
```

### When to Use Which
```
Merge when:
  - Integrating a completed feature branch
  - Preserving full history is important
  - Shared/public branches

Rebase when:
  - Cleaning up local commits before PR
  - Staying up to date with main
  - Never on commits others have
```

---

## Stashing

```bash
git stash                         # stash changes
git stash push -m "description"   # stash with name
git stash -u                      # include untracked files
git stash -a                      # include ignored files too

git stash list                    # view all stashes
git stash show stash@{0}          # show diff of stash
git stash show -p stash@{0}       # full patch

git stash pop                     # apply + drop top stash
git stash apply stash@{1}         # apply without dropping
git stash drop stash@{0}          # delete a stash
git stash clear                   # delete ALL stashes
git stash branch feature stash@{0} # create branch from stash
```

---

## Remote Operations

```bash
git remote -v                     # list remotes
git remote add origin URL         # add remote
git remote rename origin upstream # rename
git remote remove origin          # remove

git fetch                         # fetch all remotes (no merge)
git fetch origin main             # fetch specific branch
git pull                          # fetch + merge
git pull --rebase                 # fetch + rebase (cleaner)

git push origin feature           # push branch
git push -u origin feature        # push + set upstream
git push --force-with-lease       # safe force push (checks remote state)
git push --force                  # DANGEROUS: overwrites remote
git push origin :branch           # delete remote branch (old syntax)
git push origin --delete branch   # delete remote branch

git remote prune origin           # remove stale remote-tracking branches
git fetch --prune                 # prune while fetching
```

---

## Undoing Things

### Before Committing
```bash
git restore file.txt              # discard unstaged changes
git restore --staged file.txt     # unstage file (keep changes)
git restore --staged --worktree . # unstage + discard all changes
git clean -fd                     # remove untracked files + dirs
git clean -fdx                    # also remove ignored files (DESTRUCTIVE)
git clean -n                      # dry run (show what would be removed)
```

### After Committing
```bash
git revert HEAD                   # create new commit that undoes HEAD
git revert HEAD~3..HEAD           # revert a range of commits
git commit --amend                # fix last commit (don't push)

# Reset — moves HEAD and optionally the index/working tree
git reset --soft HEAD~1           # undo commit, keep staged
git reset HEAD~1                  # undo commit, keep unstaged (default: --mixed)
git reset --hard HEAD~1           # undo commit, DISCARD changes
git reset --hard origin/main      # match remote (DESTRUCTIVE)
```

### Recovering "Lost" Commits
```bash
git reflog                        # see everything HEAD has pointed to
git checkout abc1234              # detach HEAD at any point
git branch recovery abc1234       # create branch at lost commit
git cherry-pick abc1234           # apply a single commit
```

---

## History & Inspection

```bash
git log
git log --oneline
git log --oneline --graph --all --decorate
git log -p                        # show patches
git log -p file.txt               # history of one file
git log --stat                    # files changed per commit
git log --follow file.txt         # follow renames
git log --author="Name"
git log --grep="pattern"
git log --since="2 weeks ago"
git log --until="2024-01-01"
git log main..feature             # commits in feature not in main
git log feature..main             # commits in main not in feature
git log --diff-filter=D -- file   # find deleted file in history

git show HEAD                     # show last commit + diff
git show abc1234:file.txt         # show file at specific commit

git diff main feature             # diff two branches
git diff main...feature           # diff since they diverged
git diff HEAD~3 -- file.txt       # file diff against N commits ago

git blame file.txt                # who changed each line
git blame -L 10,20 file.txt       # blame for line range
git blame -w file.txt             # ignore whitespace
```

---

## Bisect

Find which commit introduced a bug using binary search:

```bash
git bisect start
git bisect bad                    # current commit is bad
git bisect good v1.2.3            # this tag was good

# Git checks out middle commit — test it
# Then mark it:
git bisect good                   # or:
git bisect bad

# Repeat until Git identifies the culprit
git bisect reset                  # exit bisect mode

# Automate with a test script:
git bisect start HEAD v1.2.3
git bisect run npm test           # must exit 0 for good, non-zero for bad
git bisect run ./test.sh
```

---

## Reflog — The Safety Net

The reflog records every position HEAD has been at for the past 90 days. **You can recover almost anything with it.**

```bash
git reflog                        # show full reflog
git reflog show branch-name       # reflog for specific branch

# Recover deleted branch
git reflog                        # find the SHA before deletion
git checkout -b recovered-branch abc1234

# Undo a hard reset
git reflog                        # find the SHA before reset
git reset --hard abc1234

# Undo a bad rebase
git reflog                        # find SHA before "rebase: ..."
git reset --hard ORIG_HEAD        # or use the SHA
```

---

## Tags

```bash
git tag                           # list tags
git tag v1.0.0                    # lightweight tag at HEAD
git tag -a v1.0.0 -m "Release"   # annotated tag (recommended)
git tag -a v1.0.0 abc1234         # tag specific commit

git show v1.0.0                   # show tag info
git push origin v1.0.0            # push tag
git push origin --tags            # push all tags

git tag -d v1.0.0                 # delete local tag
git push origin :refs/tags/v1.0.0 # delete remote tag
git push origin --delete v1.0.0  # delete remote tag (newer syntax)

git checkout v1.0.0               # detached HEAD at tag
```

---

## Worktrees

Check out multiple branches simultaneously in different directories:

```bash
git worktree add ../hotfix hotfix-branch
git worktree add -b new-branch ../feature origin/main

git worktree list                 # show all worktrees
git worktree remove ../hotfix     # remove worktree
git worktree prune                # clean stale entries
```

**Use case:** You're deep in a feature but need to check out main for a hotfix. Instead of stashing, use a worktree.

---

## Submodules

```bash
git submodule add URL path/to/sub
git submodule init
git submodule update --init --recursive

# Clone a repo with submodules
git clone --recurse-submodules URL

# Update submodules
git submodule update --remote --merge

# Run command in each submodule
git submodule foreach 'git pull origin main'
```

---

## Hooks

Located in `.git/hooks/`. Copy sample and make executable.

### Useful hooks

**`pre-commit`** — run tests/linting before commit:
```bash
#!/bin/bash
npm run lint || exit 1
npm test || exit 1
```

**`commit-msg`** — validate commit message format:
```bash
#!/bin/bash
commit_msg=$(cat "$1")
pattern="^(feat|fix|docs|style|refactor|test|chore)(\(.+\))?: .{1,50}"
if ! echo "$commit_msg" | grep -qE "$pattern"; then
    echo "❌ Commit message format invalid. Use: type(scope): description"
    exit 1
fi
```

**`pre-push`** — prevent pushing to protected branches:
```bash
#!/bin/bash
protected_branches="main master production"
current_branch=$(git symbolic-ref HEAD | sed 's!refs/heads/!!')
for branch in $protected_branches; do
    if [ "$current_branch" = "$branch" ]; then
        echo "❌ Direct push to $branch is not allowed."
        exit 1
    fi
done
```

### Share hooks across a team
```bash
git config core.hooksPath .githooks   # use project-level hooks dir
```

---

## Advanced Techniques

### Cherry-pick
```bash
git cherry-pick abc1234               # apply single commit
git cherry-pick abc..def              # apply range
git cherry-pick abc1234 --no-commit   # apply changes but don't commit
git cherry-pick --abort
```

### Partial staging (`add -p`)
```
y  stage this hunk
n  skip this hunk
s  split into smaller hunks
e  manually edit the hunk
q  quit
a  stage this hunk and all remaining
d  skip this hunk and all remaining
```

### Find a file in history
```bash
git log --all --full-history -- "**/filename.txt"
git show SHA:path/to/file > recovered.txt
```

### Archive a repo at a point in time
```bash
git archive HEAD --format=zip > archive.zip
git archive v1.0.0 --format=tar.gz > release.tar.gz
```

### Searching code in history
```bash
git grep "pattern" HEAD
git grep "pattern" $(git rev-list --all)     # search all commits
git log -S "function_name" -p                # pickaxe: find when string added/removed
git log -G "regex" -p                        # pickaxe: regex version
```

### Large file handling
```bash
# See what's bloating the repo
git count-objects -vH
git rev-list --all --objects | sort -k 2 | uniq | \
  git cat-file --batch-check | sort -k 3 -n | tail -20

# BFG Repo Cleaner (faster than git filter-branch)
bfg --delete-files "*.zip"
bfg --replace-text passwords.txt
git reflog expire --expire=now --all && git gc --prune=now --aggressive
```

---

## Aliases Worth Having

```bash
[alias]
    # Basics
    st = status -sb
    co = checkout
    sw = switch
    br = branch

    # Logging
    lg = log --oneline --graph --all --decorate
    ll = log --oneline -20
    last = log -1 HEAD --stat
    who = log --format='%aN' | sort | uniq -c | sort -rn

    # Diffs
    df = diff
    dc = diff --cached
    dw = diff --word-diff

    # Shortcuts
    unstage = restore --staged
    discard = restore
    undo = reset HEAD~1 --mixed
    save = stash push -u -m

    # Utility
    aliases = config --get-regexp alias
    root = rev-parse --show-toplevel
    branches = branch --sort=-committerdate --format='%(committerdate:relative)%09%(refname:short)'
```

---

## gitconfig Reference

```ini
[core]
    editor = vim
    autocrlf = input
    excludesFile = ~/.gitignore_global
    pager = less -F -X              # don't page if output fits on screen

[color]
    ui = auto

[push]
    default = current
    autoSetupRemote = true

[pull]
    rebase = true

[rebase]
    autosquash = true               # auto-apply fixup!/squash! commits
    autostash = true                # stash before rebase, pop after

[merge]
    conflictstyle = diff3           # show base code in conflicts (very helpful)

[diff]
    algorithm = histogram           # better diff algorithm

[fetch]
    prune = true                    # auto-remove stale remote branches

[help]
    autocorrect = 10                # auto-correct typos after 1s

[branch]
    sort = -committerdate           # show recently updated branches first

[init]
    defaultBranch = main
```
