# GitHub Issue-to-PR Workflow

**Location:** Arc Raiders Trust Platform
**Status:** COMPLETE WORKFLOW REFERENCE
**Last Updated:** 2026-01-14

## Complete Workflow for Issue Resolution

This is the standardized workflow for taking a GitHub issue from selection to PR creation and review.

### 1. Pick Issue & Create Branch

```bash
gh issue view XXX                              # Review issue details
git checkout main && git pull origin main      # Ensure main is latest
git checkout -b feature/issue-XXX-description # Create feature branch
```

**Branch naming convention:**
- `feature/issue-XXX-short-description` - For new features
- `fix/issue-XXX-short-description` - For bug fixes

### 2. Implement Changes

Follow code standards for the project:
- Backend (.NET): async/await, DTOs, DI, RESTful conventions
- Frontend (React): strict TS, interfaces, null-safe operators, error handling
- Tests: 85%+ coverage required, all tests must pass

### 3. Commit with Conventional Format

```bash
git add -A
git commit -m "feat: short description (closes #XXX)

- Change 1
- Change 2
- Change 3"
```

**Commit types:**
- `feat:` - New feature
- `fix:` - Bug fix
- `docs:` - Documentation
- `style:` - Code style
- `refactor:` - Code refactoring
- `test:` - Test additions
- `chore:` - Build/tooling

### 4. Push to Remote

```bash
git push origin feature/issue-XXX-description
```

GitHub will show a prompt with a PR creation link.

### 5. Create Pull Request

**Using GitHub CLI (recommended):**

```bash
gh pr create \
  --title "feat: description" \
  --body "Closes #XXX

## Changes
- Change 1
- Change 2

## Testing
- Test coverage: XX%
- All tests passing

## Type
- [x] Feature
- [ ] Bug Fix" \
  --base main
```

**Response includes PR number:** `https://github.com/Khavel/RaidersRep/pull/YYY`

### 6. Update Issue with PR Link

```bash
gh issue comment XXX --body "✅ **Implementation Complete**

**Branch:** feature/issue-XXX-description
**PR:** #YYY

Changes verified and ready for review"
```

---

## Complete Command Sequence

For quick reference, here's the full automation sequence:

```bash
# 1. Review and create branch
gh issue view 48
git checkout main && git pull origin main
git checkout -b feature/issue-48-displayname-field

# 2. Make changes and test
dotnet build
dotnet test

# 3. Commit and push
git add -A
git commit -m "feat: add DisplayName field to User model (closes #48)

- Add DisplayName property with validation
- Create EF Core migration
- Update DTOs and controllers
- Fix test fixtures
- All 59 tests passing"

git push origin feature/issue-48-displayname-field

# 4. Create PR
gh pr create \
  --title "feat: add DisplayName field to User model" \
  --body "Closes #48

## Changes
- DisplayName field added with validation
- Database migration created
- All DTOs updated
- Controllers updated
- Test fixtures fixed

## Testing
- All 59 tests passing
- Build successful
- Coverage: 85%+" \
  --base main

# 5. Comment on issue
gh issue comment 48 --body "✅ **Implementation Complete**
**Branch:** feature/issue-48-displayname-field
**PR:** #50
Ready for review"
```

---

## Checklist Before Pushing

- [ ] `dotnet build` passes with no errors
- [ ] `dotnet test` passes - ALL tests must pass
- [ ] Test coverage meets 85% threshold
- [ ] Code follows project standards
- [ ] DTOs used for API responses (no entities)
- [ ] Async/await for I/O operations
- [ ] Proper HTTP status codes
- [ ] Components support light/dark theme (frontend)
- [ ] Translations use i18n keys (frontend)

---

## Post-PR

1. **Wait for CI/CD to pass** - GitHub Actions will run tests automatically
2. **Address review feedback** - Push additional commits to same branch
3. **Merge after approval** - PR will auto-close the issue with `Closes #XXX`
4. **Delete branch** - Clean up after merge

---

## Why This Matters

This workflow ensures:
- Clean git history with meaningful commits
- Automatic issue linking via `Closes #XXX`
- Traceability from issue → branch → commit → PR
- Clear communication in PR description
- Test quality enforced before merge
- Proper code review process

## Related Issues

- See: `.github/WORKFLOW-PR-CREATION.md` in the main repo
- See: `copilot-instructions.md` for detailed development rules
