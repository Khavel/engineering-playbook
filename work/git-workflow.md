# Feature Branch & PR Workflow

## Branch Naming Convention

Use descriptive, issue-linked branch names following this pattern:

```
feature/{issue-number}-description    # New features
fix/{issue-number}-description        # Bug fixes
docs/{description}                    # Documentation
refactor/{description}                # Code refactoring
```

Examples:
- `feature/5-admin-dashboard`
- `fix/27-feedback-authorization`
- `fix/25-n1-query-optimization`
- `docs/adding-api-endpoints`

## Commit Message Format

Use conventional commit format to enable automated changelog generation:

```
{type}: {description} (closes #{issue})
```

Types: `feat`, `fix`, `docs`, `style`, `refactor`, `test`, `chore`

Examples:
```
feat: add authorization to feedback endpoints (closes #27)
fix: optimize moderation queue with eager loading (closes #25)
docs: update API contracts for admin endpoints
```

## PR Workflow

### 1. Create Feature Branch
```bash
git checkout -b fix/issue-27-feedback-authorization
```

### 2. Make Changes & Commit
```bash
# Stage specific files related to the issue
git add src/ArcRaiders.Trust.Api/Controllers/FeedbackController.cs

# Commit with conventional message
git commit -m "fix: add authorization and validation to feedback endpoints (closes #27)"
```

### 3. Push & Create PR
```bash
# Push with upstream tracking
git push -u origin fix/issue-27-feedback-authorization

# Create PR via GitHub CLI
gh pr create --title "fix: add authorization..." --body "## Issue\n\nCloses #27\n\n## Changes\n..."
```

### 4. PR Description Template

```markdown
## Issue
Closes #{issue-number}

## Changes
- Bullet point describing what changed
- Another change
- etc

## Testing
- How you tested this locally
- Tests that pass/verify the fix

## Checklist
- [ ] Code builds without errors
- [ ] All tests pass
- [ ] Documentation updated (if needed)
- [ ] No breaking changes
```

### 5. Address Review Feedback
Make changes locally, commit with a message that references the issue, and push:

```bash
git commit -m "fix: address review feedback"
git push
```

### 6. Merge & Close
Once approved, use GitHub UI to merge PR which auto-closes the linked issue.

## Key Principles

- **One issue per branch**: Keep branches focused on a single issue
- **Descriptive messages**: Make history readable for future developers
- **Reference issues**: Use `closes #123` to auto-link and close issues
- **Small, reviewable PRs**: Easier to review and faster feedback cycle
- **Separate concerns**: Don't mix unrelated changes in one PR

## When Issues Are Blocked By Other Issues

Update the blocked issue's description to list blockers:

```markdown
## Blocked By
This issue is blocked by the following issues:
- #27 - Authorization fixes required first
- #25 - Performance optimization needed
```

Then tackle blockers in priority order, closing them one by one.
