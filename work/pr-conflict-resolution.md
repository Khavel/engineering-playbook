# PR Merge Conflict Resolution Workflow

When GitHub reports "merge commit cannot be cleanly created" on a PR.

## Quick Reference

```bash
# 1. Stash current work (include untracked files)
git stash push -u -m "temp-before-conflict-resolution"

# 2. Checkout the PR branch
git checkout feature/issue-xxx-description

# 3. Fetch latest and merge main
git fetch origin
git merge origin/main

# 4. Find conflict markers
git grep -n "<<<<<<<" path/to/file.ext

# 5. Resolve conflicts in each file, then stage
git add path/to/resolved-file.ext

# 6. Complete merge
git commit -m "Merge main into feature/issue-xxx-description"

# 7. Push updated branch
git push origin feature/issue-xxx-description

# 8. Merge PR (with admin override if CI non-blocking)
gh pr merge <PR#> --squash --admin --delete-branch
```

## Key Insights

### Stashing Untracked Files
Use `-u` flag to include untracked files in stash:
```bash
git stash push -u -m "descriptive-message"
```

### Finding All Conflict Markers
Search for markers in specific file or entire repo:
```bash
git grep -n "<<<<<<<" filename.ext     # Single file
git grep -n "<<<<<<<"                   # All files
```

### Admin Override for Non-Blocking Failures
When CI checks like SonarCloud fail but you've decided to proceed:
```bash
gh pr merge <PR#> --squash --admin --delete-branch
```

### Multiple PRs with Same Conflict Source
If multiple PRs conflict on the same file:
1. Merge PRs one at a time
2. After each merge, other PR branches need to re-merge main
3. Conflicts may compound — resolve incrementally

## Common Conflict Patterns

### Documentation Files (.md, .instructions.md)
Often whitespace or section ordering issues. Usually safe to keep both versions merged together.

### Package Files (package.json, package-lock.json)
Regenerate lock file after resolving package.json:
```bash
npm install --package-lock-only
```

### Config Files
Check for duplicate keys after resolving — JSON/YAML parsers will fail silently on duplicates.
