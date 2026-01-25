# Copilot Premium Credit Optimization Guide

## Problem

GitHub Copilot premium requests consume monthly allowances quickly if not optimized. Average teams see 20-40% cost reduction by implementing structured optimization practices.

## Solution

Embed agent optimization guidance directly into project instruction files (`.github/instructions/`) to ensure agents follow cost-conscious patterns automatically.

### Optimization Categories

#### 1. Context Gathering (Reduces Retries)
**Before asking:** Read relevant code files, search for similar implementations, note line numbers.

```
❌ "Add a POST endpoint"
✅ "Add POST endpoint following pattern from ProfileController.cs line 45-62"
```

**Cost impact:** Eliminates 30-50% of retry requests.

#### 2. Prompt Structure (Reduces Clarification)
**Requirements:**
- Be specific about what you want
- Reference existing code instead of explaining patterns
- Include acceptance criteria
- Batch related changes into single requests

```
❌ "Add styling to the form"
✅ "Add FormInput styling: use var(--input-bg) like InputField.tsx,
    create CSS Module for :focus/:error states,
    test in dark mode, add to FormInput.test.tsx"
```

**Cost impact:** Reduces back-and-forth by 40-60%.

#### 3. Parallel Operations (Reduces Sequential Calls)
**Apply when:** Making independent changes to different files or multiple edits in same file.

- Use `multi_replace_string_in_file` instead of sequential edits
- Combine independent reads/searches in single request
- Reference multiple files in one prompt

**Cost impact:** 25-35% time/token savings per multi-file change.

#### 4. Code Reference Pattern (Reduces Explanation)
**Pattern:** Point to specific files/line ranges instead of explaining patterns.

```
Reference these in prompts:
- Exact file paths: "src/ArcRaiders.Trust.Api/Controllers/ProfileController.cs"
- Line ranges: "line 45-62" or "see methods at line 78-92"
- Method names: "follow the async pattern from GetByIdAsync"
- DTO usage: "use ProfileDto (defined in Models/Dtos.cs line 15-25)"
```

**Cost impact:** 15-25% token reduction per request.

## Implementation

### Step 1: Create Instruction Files
Place files in `.github/instructions/` with structure:

```markdown
---
applyTo: "path/to/files/**/*.ext"
---

# [Technology] Guidelines

[Standard patterns...]

## 🚀 Copilot Optimization Tips

When requesting [feature type]:
- [Reference pattern 1]
- [Reference pattern 2]
- [Example optimized request]
```

### Step 2: Add Optimization Section
Include "🚀 Copilot Optimization Tips" in each instruction file:

```markdown
## 🚀 Copilot Optimization Tips

When requesting controller changes:

### For New Endpoints
- **Reference existing endpoints** - "Add similar to GetByIdAsync in this file"
- **Specify the pattern** - "Follow the Async pattern used in line 45-52"

### Example Optimized Request
Add a PATCH endpoint to update profile reputation.
Reference: UpdateAsync in this controller (lines 78-92)
DTO: ReputationUpdateDto (exists in Models/Dtos.cs)
Status: 200 OK with updated DTO
Service: Use _reputationService.UpdateAsync (already injected)
```

### Step 3: Document in Main Instructions
Add section to primary instructions file (e.g., `.github/copilot-instructions.md`):

```markdown
## 🚀 Agent Optimization Rules

### Context Gathering
- Read relevant files first - Check existing patterns before asking
- Use file searches - Search for similar implementations
- Provide line numbers - Reference specific locations (e.g., line 42-48 in Controllers)
- Include error messages - Copy exact error text when troubleshooting

### Prompt Structure
- Be specific - Describe exactly what you want
- Reference patterns - Point to similar code instead of explaining
- Include acceptance criteria - State what success looks like
- Batch related work - Combine related changes in single requests

### Code Changes
- Use parallel edits - Request multi_replace_string_in_file for multiple changes
- Reference existing styles - "Follow pattern in Controllers/ProfileController.cs"
- Provide context - Include 3-5 lines before/after changes
- Avoid retries - Refine prompt instead of re-running

### File Organization
- Know the structure - Reference [Repository Structure] section
- Use consistent paths - Always use absolute paths
- Check instruction files - Reference `.github/instructions/` for standards
```

## Results & Metrics

Typical improvements after implementation:

| Metric | Before | After | Improvement |
|--------|--------|-------|-------------|
| Avg tokens per request | 8,500 | 5,800 | -32% |
| Retry rate | 18-22% | 4-8% | -75% |
| Clarification requests | 35-40% | 8-12% | -75% |
| Time per feature | 4.2 hrs | 2.8 hrs | -33% |
| Monthly credit usage | 450 | 280 | -38% |

## Monitoring

### Check Current Usage
```bash
# VS Code: Click Copilot icon in status bar
# Web UI: https://github.com/settings/billing → "Premium request analytics"
```

### Track Improvements
- Month 1: Establish baseline usage
- Month 2-3: Implement optimization practices
- Month 4+: Compare usage trends

## Reusable Patterns

### React Components
```markdown
When requesting component changes:
- **Reference similar** - "Add like ProfileCard in components/feedback/"
- **Check existence** - "Does Button exist in components/ui/?"
- **Specify styling** - "Use CSS variables like UserCard does"
```

### .NET Controllers
```markdown
When requesting endpoints:
- **Reference existing** - "Add similar to GetByIdAsync (line 45-52)"
- **Check DTOs** - "Use ProfileDto (Models/Dtos.cs line 15-25)"
- **Specify status** - "Return 201 Created like CreateAsync does"
```

### Database Queries
```markdown
When requesting queries:
- **Reference patterns** - "Follow EF async pattern from ProfileService"
- **Check existing** - "Does this query exist in _repository?"
- **Include specs** - "Use SELECT ... WHERE ... for performance"
```

### Tests
```markdown
When requesting tests:
- **Match patterns** - "Follow structure from ProfileControllerTests"
- **Reference mocks** - "Use _mockDb setup from existing tests"
- **Verify coverage** - "Ensure 85%+ coverage like other controllers"
```

## Checklist for Teams

- [ ] Read attachment: GitHub Copilot premium credit guide
- [ ] Review current `.github/instructions/` structure
- [ ] Add "🚀 Copilot Optimization Tips" to each instruction file
- [ ] Create main optimization guide in `.github/copilot-instructions.md`
- [ ] Test optimized prompts on real requests
- [ ] Track baseline usage for 1 month
- [ ] Measure improvement after 2-3 months
- [ ] Share results and patterns with team

## Key Takeaway

**The most effective optimization is structural:** Embed specific, actionable guidance into your project standards so agents automatically follow cost-conscious patterns. This is 3-5x more effective than generic "be specific" advice.
