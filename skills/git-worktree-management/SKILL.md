---
name: git-worktree-management
description: Git Worktree management for AI-driven development. Includes workflows, best practices, and usage patterns for isolated development sessions.
---

# Git Worktree Management (2026)

## Core Concepts

### What are Git Worktrees?

Git Worktrees allow you to have **multiple working trees** (checkouts) from the same repository, each in its own directory. This enables:

* **Isolation**: Work on multiple features simultaneously without conflicts
* **Efficiency**: Share the same `.git` directory to save disk space
* **Context Switching**: Quick switching between branches without commits or stashing
* **Safety**: Protect your main branch from destructive changes

### Basic Workflow

```bash
# Create a worktree for a new feature
git worktree add ../myapp-feature feature/awesome-feature

# List all worktrees
git worktree list

# Remove worktree when done
git worktree remove ../myapp-feature
```

## Project Structure

```
project/
├── .git/                    # Shared git repository
├── .opencode/               # OpenCode configuration
│   └── worktree.jsonc       # Sync configuration for worktrees
├── src/                     # Main source
└── worktrees/               # (Optional) Organized worktree location
    ├── feature-login/
    ├── hotfix-auth/
    └── experiment-ui/
```

## Use Cases

### 1. Feature Development
```bash
# Create worktree for feature branch
git worktree add worktrees/feature/user-auth feature/user-auth

# Work in isolation
cd worktrees/feature/user-auth
# ... develop feature ...
cd ../..
git worktree remove worktrees/feature/user-auth
```

### 2. Hotfix Development
```bash
# Create hotfix worktree from main
git worktree add worktrees/hotfix/security-patch fix/security-patch

# Apply and test fix
cd worktrees/hotfix/security-patch
# ... fix security issue ...
cd ../..
git worktree remove worktrees/hotfix/security-patch
```

### 3. Code Review Preview
```bash
# Create worktree for pull request preview
git worktree add worktrees/pr-preview feature/new-feature

# Review changes in isolated environment
cd worktrees/pr-preview
# ... test and review ...
```

## OpenCode Integration

### OpenCode Worktree Configuration

```json
// .opencode/worktree.jsonc
{
  "$schema": "https://registry.kdco.dev/schemas/worktree.json",
  "sync": {
    "copyFiles": [
      ".env",
      ".env.local",
      ".opencode/config.json"
    ],
    "symlinkDirs": [
      "node_modules",
      "pnpm-store",
      ".cache"
    ],
    "exclude": [
      "logs/",
      "*.log",
      "node_modules/.cache/"
    ]
  },
  "hooks": {
    "postCreate": [
      "pnpm install",
      "npm run build"
    ],
    "preDelete": [
      "git status --porcelain",
      "git log --oneline -10"
    ]
  }
}
```

### OpenCode Command Usage

```markdown
/worktree create feature-name
```

This creates a new worktree with OpenCode ready to go.

```markdown
/worktree list
```

This lists all active worktrees with their status.

```markdown
/worktree delete path/to/worktree
```

This safely removes a worktree and commits changes.

## OpenCode Plugin: `opencode-worktree`

### Features

* **Zero-Friction**: Automatic terminal spawning when worktree is created
* **Sync**: Automatically copy files and symlink directories
* **Hooks**: Pre/post hooks for setup and cleanup
* **Cleanup**: Automatic commit and cleanup when worktree is deleted

### Commands

| Command | Description |
|---------|-------------|
| `worktree_create(branch, baseBranch?)` | Create new worktree with OpenCode running in terminal |
| `worktree_delete(reason)` | Delete worktree, auto-commit changes |

### Installation

```bash
# Using OCX
ocx add kdco/worktree --from https://registry.kdco.dev

# Or manually (see plugin documentation)
```

## Worktree Management Patterns

### Pattern 1: Multi-Feature Development

```bash
# Start multiple worktrees simultaneously
git worktree add worktrees/feature/api-v2 feature/api-v2
git worktree add worktrees/feature/ui-redesign feature/ui-redesign
git worktree add worktrees/feature/documentation feature/documentation

# Each worktree has its own process, IDE, and build
# Main repository stays clean
```

### Pattern 2: Experimentation

```bash
# Try something risky in isolation
git worktree add worktrees/experiment/react-19 experiment/react-19
cd worktrees/experiment/react-19
# ... try new React version ...
# If it breaks, just delete worktree
git worktree remove worktrees/experiment/react-19

# Back to main development without cleanup
```

### Pattern 3: Cross-Branch Comparison

```bash
# Compare current feature with production
git worktree add worktrees/compare/live main

# Check differences
cd worktrees/compare/live
git diff main
```

## Best Practices

### ✅ Do

* Use descriptive worktree names: `feature/`, `hotfix/`, `experiment/`
* Organize worktrees in a dedicated directory: `worktrees/`
* Use hooks for automatic setup (install dependencies, build)
* Delete worktrees when done to keep workspace clean
* Commit worktree cleanup notes to maintain history

### ❌ Don't

* Don't create worktrees inside the main repository (use `../`)
* Don't forget to update submodules in new worktrees
* Don't share build artifacts between worktrees (use separate `node_modules`)
* Don't push worktree branches until they're ready
* Don't leave worktrees unattended (they accumulate!)

## Worktree Safety Features

### Protection Against Accidental Deletion

```bash
# Check if worktree is clean before deletion
git worktree list

# Only delete clean worktrees
git worktree prune
```

### Backup Strategy

```bash
# Add worktree branch to remote before deletion
cd worktrees/feature/my-feature
git push origin feature/my-feature

# Delete worktree
cd ../..
git worktree remove worktrees/feature/my-feature
```

## Advanced Usage

### Worktree with Existing Branch

```bash
# Create worktree from existing branch
git worktree add worktrees/feature/branch-name branch-name
```

### Worktree from Specific Commit

```bash
# Create worktree from a specific commit
git worktree add -b temp-feature worktrees/experiment/commit-123 abc1234
```

### Moving Worktrees

```bash
# Move worktree to different location
git worktree move old/path new/path
```

### Pruning Stale Worktrees

```bash
# Remove worktrees that have been deleted
git worktree prune
```

## Troubleshooting

### Common Issues

**Issue**: Worktree shows as "pruned" but still exists
```bash
# Fix: Manually removing the worktree directory and running prune
rm -rf worktrees/feature/my-feature
git worktree prune
```

**Issue**: Can't create worktree (branch already exists)
```bash
# Fix: Use different branch name or delete existing branch
git branch -d existing-branch
git worktree add worktrees/feature/my-feature new-feature
```

**Issue**: Hook commands fail in worktree
```bash
# Fix: Ensure hooks are configured to work in subdirectories
# Check .opencode/hooks/postCreate for path issues
```

## References

* [Git Worktree Documentation](https://git-scm.com/docs/git-worktree)
* [OpenCode Worktree Plugin](https://github.com/kdcokenny/opencode-worktree)
* [Pro Git Book - Worktrees](https://git-scm.com/book/en/v2/Git-Tools-Worktrees)

## 2026 Best Practices

* **OpenCode Integration**: Use `opencode-worktree` plugin for automation
* **Isolation**: Keep features in separate worktrees for safety
* **Cleanup**: Delete worktrees when not needed to prevent clutter
* **Hooks**: Use hooks for consistent setup across worktrees
* **Documentation**: Document worktree workflows for your team
