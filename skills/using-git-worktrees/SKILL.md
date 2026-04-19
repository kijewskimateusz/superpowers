---
name: using-git-worktrees
description: Use when starting feature work that needs isolation from current workspace or before executing implementation plans - creates isolated git worktrees with smart directory selection and safety verification
---

# Using Git Worktrees

## Overview

Git worktrees create isolated workspaces sharing the same repository, allowing work on multiple branches simultaneously without switching.

**Core principle:** Systematic directory selection + safety verification = reliable isolation.

**Announce at start:** "I'm using the using-git-worktrees skill to set up an isolated workspace."

## Directory Selection Process

Follow this priority order:

### 1. Check Sibling Directory (VS Code default)

```bash
# Check for sibling worktree directory (created at same level as project root)
project_root=$(git rev-parse --show-toplevel)
ls -d "${project_root}.worktrees" 2>/dev/null
```

**If found:** Use it. This is the VS Code default pattern — a directory named `<project>.worktrees/` adjacent to the project root (e.g., `/foo/bar/myproject.worktrees/`). No `.gitignore` verification needed since it's outside the project.

### 2. Check In-Project Directories

```bash
# Check in priority order
ls -d .worktrees 2>/dev/null     # Preferred (hidden)
ls -d worktrees 2>/dev/null      # Alternative
```

**If found:** Use that directory. If both exist, `.worktrees` wins.

### 3. Check CLAUDE.md

```bash
grep -i "worktree.*director" CLAUDE.md 2>/dev/null
```

**If preference specified:** Use it without asking.

### 4. Ask User

If no directory exists and no CLAUDE.md preference:

```
No worktree directory found. Where should I create worktrees?

1. <project>.worktrees/ (sibling directory, VS Code default — no gitignore needed)
2. .worktrees/ (project-local, hidden)
3. ~/.config/superpowers/worktrees/<project-name>/ (global location)

Which would you prefer?
```

## Safety Verification

### For Sibling Directory (<project>.worktrees)

No `.gitignore` verification needed — directory is outside the project entirely.

### For Project-Local Directories (.worktrees or worktrees)

**MUST verify directory is ignored before creating worktree:**

```bash
# Check if directory is ignored (respects local, global, and system gitignore)
git check-ignore -q .worktrees 2>/dev/null || git check-ignore -q worktrees 2>/dev/null
```

**If NOT ignored:**

Per Jesse's rule "Fix broken things immediately":
1. Add appropriate line to .gitignore
2. Commit the change
3. Proceed with worktree creation

**Why critical:** Prevents accidentally committing worktree contents to repository.

### For Global Directory (~/.config/superpowers/worktrees)

No `.gitignore` verification needed — outside project entirely.

## Creation Steps

### 1. Detect Project Name

```bash
project=$(basename "$(git rev-parse --show-toplevel)")
```

### 2. Create Worktree

```bash
# Determine full path
project_root=$(git rev-parse --show-toplevel)

case $LOCATION in
  sibling)
    path="${project_root}.worktrees/$BRANCH_NAME"
    ;;
  .worktrees|worktrees)
    path="$LOCATION/$BRANCH_NAME"
    ;;
  ~/.config/superpowers/worktrees/*)
    path="~/.config/superpowers/worktrees/$project/$BRANCH_NAME"
    ;;
esac

# Create worktree with new branch
git worktree add "$path" -b "$BRANCH_NAME"
cd "$path"
```

### 3. Run Project Setup

Auto-detect and run appropriate setup:

```bash
# Node.js
if [ -f package.json ]; then npm install; fi

# Rust
if [ -f Cargo.toml ]; then cargo build; fi

# Python
if [ -f requirements.txt ]; then pip install -r requirements.txt; fi
if [ -f pyproject.toml ]; then poetry install; fi

# Go
if [ -f go.mod ]; then go mod download; fi
```

### 4. Verify Clean Baseline

Run tests to ensure worktree starts clean:

```bash
# Examples - use project-appropriate command
npm test
cargo test
pytest
go test ./...
```

**If tests fail:** Report failures, ask whether to proceed or investigate.

**If tests pass:** Report ready.

### 5. Report Location

```
Worktree ready at <full-path>
Tests passing (<N> tests, 0 failures)
Ready to implement <feature-name>
```

## Quick Reference

| Situation | Action |
|-----------|--------|
| `<project>.worktrees/` exists (sibling) | Use it (no gitignore needed) |
| `.worktrees/` exists (in-project) | Use it (verify ignored) |
| `worktrees/` exists (in-project) | Use it (verify ignored) |
| Sibling + in-project both exist | Sibling wins |
| Both in-project exist | Use `.worktrees/` |
| None exist | Check CLAUDE.md → Ask user |
| In-project directory not ignored | Add to .gitignore + commit |
| Tests fail during baseline | Report failures + ask |
| No package.json/Cargo.toml | Skip dependency install |

## Common Mistakes

### Skipping ignore verification

- **Problem:** Worktree contents get tracked, pollute git status
- **Fix:** Always use `git check-ignore` before creating project-local worktree

### Assuming directory location

- **Problem:** Creates inconsistency, violates project conventions
- **Fix:** Follow priority: sibling > in-project > CLAUDE.md > ask

### Proceeding with failing tests

- **Problem:** Can't distinguish new bugs from pre-existing issues
- **Fix:** Report failures, get explicit permission to proceed

### Hardcoding setup commands

- **Problem:** Breaks on projects using different tools
- **Fix:** Auto-detect from project files (package.json, etc.)

## Example Workflows

### Sibling directory (VS Code default)

```
You: I'm using the using-git-worktrees skill to set up an isolated workspace.

[Check /Users/jesse/myproject.worktrees/ - exists]
[Sibling directory — no gitignore verification needed]
[Create worktree: git worktree add /Users/jesse/myproject.worktrees/auth -b feature/auth]
[Run npm install]
[Run npm test - 47 passing]

Worktree ready at /Users/jesse/myproject.worktrees/auth
Tests passing (47 tests, 0 failures)
Ready to implement auth feature
```

### In-project directory

```
You: I'm using the using-git-worktrees skill to set up an isolated workspace.

[Check myproject.worktrees/ - not found]
[Check .worktrees/ - exists]
[Verify ignored - git check-ignore confirms .worktrees/ is ignored]
[Create worktree: git worktree add .worktrees/auth -b feature/auth]
[Run npm install]
[Run npm test - 47 passing]

Worktree ready at /Users/jesse/myproject/.worktrees/auth
Tests passing (47 tests, 0 failures)
Ready to implement auth feature
```

## Red Flags

**Never:**
- Create worktree without verifying it's ignored (project-local)
- Skip baseline test verification
- Proceed with failing tests without asking
- Assume directory location when ambiguous
- Skip CLAUDE.md check

**Always:**
- Follow directory priority: sibling > in-project > CLAUDE.md > ask
- Verify directory is ignored for in-project paths (sibling and global skip this)
- Auto-detect and run project setup
- Verify clean test baseline

## Integration

**Called by (optional):**
- **brainstorming** - When the user requests worktree isolation or the change is substantial
- **subagent-driven-development** - When isolation is desired for multi-task execution
- **executing-plans** - When isolation is desired for plan execution
- Any skill needing isolated workspace

**When to use:** The user explicitly requests worktree isolation, OR the change is substantial (multi-file, multi-task). Skip for small, focused changes.

**Pairs with:**
- **finishing-a-development-branch** - Cleans up worktree if one was used
