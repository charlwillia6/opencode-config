---
description: Generate PR description from branch commits
agent: commit-drafter
model: ollama/rnj-1:8b
---

Generates a short paragraph description of the current changes based on commit messages.

### Context

**Git repository check:**
!`git rev-parse --git-dir`

**Current branch:**
!`git branch --show-current`

**Commit messages:**
!`git log main..HEAD --pretty=format:"%s"`

### Steps

1. **Verify git repo** → Exit if not in git repository

2. **Determine base branch** → Use main

3. **Get commit messages** → Exit if no commits on current branch

4. **Detect type** → "fixes" if fix keywords found, else "implements"

5. **Generate summary** → Short paragraph starting with "This PR ${TYPE}...", incorporating $ARGUMENTS as additional context if provided

6. **Output result**

### Additional context (optional)
$ARGUMENTS — Custom context to include in the description

### Error handling

- Not in git repo
- No commits on branch
- Base branch not found
