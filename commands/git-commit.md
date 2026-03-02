---
description: Create a git commit with user-approved commit message
agent: talk
model: ollama/glm-4.7:cloud
---

Creates a git commit using:
1. Saved commit message from getCommitMessage tool, OR
2. A newly generated message via commit-drafter agent


## Staged changes

!`git diff --staged`

## Steps

1. **Verify staged changes**
   - If nothing staged → Exit with error: "Nothing staged. Use /git-stage or git add first."

2. **Get commit message**
   - Call getCommitMessage()
   - If found → Use that message
    - If empty/null → Use conventional-commit skill to generate message

3. **User approval**
   - Present message with question tool: Approve / Edit / Cancel
   - If approved → 
     - Call saveCommitMessage(approvedMessage)
     - Commit
   - If edited → 
     - Call saveCommitMessage(editedMessage)
     - Commit
   - If cancelled → Exit

4. **Execute commit**
   - Run `git commit` with the approved message
   - Display result (hash, message)

## Error handling

- Not in git repo: Inform and exit
- Pre-commit hook failure: Show error

