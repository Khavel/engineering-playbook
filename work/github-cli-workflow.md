## GitHub CLI Workflow: Issue-Driven Development

**Purpose**: Structured workflow for AI agents and developers to manage GitHub issues, assign work, and track progress using the `gh` CLI.

### Issue States & Labels

```
[Available]
    ↓ (gh issue edit --add-assignee @me --add-label in-progress)
[In Progress]
    ↓ (if blocked: gh issue edit --remove-label in-progress --add-label blocked)
[Blocked] ↔ (unblock, then remove blocked label and add in-progress)
[In Progress]
    ↓ (git commit closes #123)
[Review]
    ↓ (gh issue edit --remove-label in-progress --add-label review)
[Merged]
    ↓
[Closed] (auto-close with commit message)
```

### Workflow Steps

#### 1. Find Work
```bash
# List open issues (priority order)
gh issue list --label "P1"
gh issue list --label "P1,P2"

# View issue details
gh issue view 123
```

#### 2. Start Task
```bash
# Assign yourself
gh issue edit 123 --add-assignee "@me"

# Mark as in-progress
gh issue edit 123 --add-label "in-progress"

# Comment that work started
gh issue comment 123 --body "Starting work on this task"

# Create feature branch
git checkout -b feature/issue-123-short-description
```

#### 3. During Development
```bash
# Update progress
gh issue comment 123 --body "Progress: Completed X, working on Y"

# If blocked
gh issue edit 123 --remove-label "in-progress" --add-label "blocked"
gh issue comment 123 --body "⚠️ Blocked: Need API endpoint /api/foo implemented first"

# When unblocked
gh issue edit 123 --remove-label "blocked" --add-label "in-progress"
```

#### 4. Create PR
```bash
# Commit with issue reference (auto-closes issue when PR merges)
git commit -m "feat: description (closes #123)"
git push -u origin feature/issue-123-short-description

# Create PR with issue reference in body
gh pr create --title "feat: description" --body "Closes #123

## Changes
- What changed
- Why it changed

## Testing
- How to test
"

# Update issue label to review
gh issue edit 123 --remove-label "in-progress" --add-label "review"
```

### Key Commands Reference

```bash
# View/Edit Issues
gh issue list [--label "P1"] [--milestone "MVP"]
gh issue view 123
gh issue edit 123 --add-label "label" --remove-label "label"
gh issue edit 123 --add-assignee "@me"
gh issue comment 123 --body "message"

# Create Issues
gh issue create --title "Title" --label "bug" --body "Details"

# Pull Requests
gh pr create --title "Title" --body-file .pr-body.md
gh pr view 32
gh pr list
```

### Tips
- Use `@me` for self-assignment: `--add-assignee "@me"`
- Combine multiple labels: `--label "P1,backend"`
- Use issue numbers in commit messages for auto-linking
- Prefix branches with issue number: `feature/issue-123-name`
- Update PR body to reference blocking issues and what it unblocks

**Related Issue**: Arc Raiders #21, #5 (Admin Dashboard)
**Proven in**: ArcRaiders RaidersRep project
