# Semantic Release Setup for Monorepos

> **Context**: Setting up automated versioning and release notes for a monorepo with .NET backend + Next.js frontend using semantic-release.

## When to Use

- Monorepo with multiple technologies (backend + frontend)
- Want automated version bumps from Conventional Commits
- Need auto-generated CHANGELOG.md and GitHub Releases
- Single version for entire repo (not per-package)

## Core Files

### 1. Root package.json (for tooling only)

```json
{
  "name": "your-project",
  "version": "0.0.0",
  "private": true,
  "scripts": {
    "release": "semantic-release",
    "release:dry-run": "semantic-release --dry-run",
    "commitlint": "commitlint --from=HEAD~1"
  },
  "devDependencies": {
    "@commitlint/cli": "^19.6.1",
    "@commitlint/config-conventional": "^19.6.0",
    "@semantic-release/changelog": "^6.0.3",
    "@semantic-release/commit-analyzer": "^13.0.0",
    "@semantic-release/exec": "^6.0.3",
    "@semantic-release/git": "^10.0.1",
    "@semantic-release/github": "^11.0.1",
    "@semantic-release/npm": "^12.0.1",
    "@semantic-release/release-notes-generator": "^14.0.1",
    "conventional-changelog-conventionalcommits": "^8.0.0",
    "semantic-release": "^24.2.0"
  },
  "engines": {
    "node": ">=20"
  }
}
```

### 2. .releaserc.json

```json
{
  "branches": ["main"],
  "tagFormat": "v${version}",
  "plugins": [
    ["@semantic-release/commit-analyzer", {
      "preset": "conventionalcommits",
      "releaseRules": [
        {"type": "feat", "release": "minor"},
        {"type": "fix", "release": "patch"},
        {"type": "perf", "release": "patch"},
        {"type": "refactor", "release": "patch"},
        {"type": "docs", "release": false},
        {"type": "style", "release": false},
        {"type": "chore", "release": false},
        {"type": "test", "release": false},
        {"type": "ci", "release": false},
        {"breaking": true, "release": "major"}
      ]
    }],
    ["@semantic-release/release-notes-generator", {
      "preset": "conventionalcommits",
      "presetConfig": {
        "types": [
          {"type": "feat", "section": "🚀 Features"},
          {"type": "fix", "section": "🐛 Bug Fixes"},
          {"type": "perf", "section": "⚡ Performance"},
          {"type": "refactor", "section": "♻️ Refactoring"}
        ]
      }
    }],
    ["@semantic-release/changelog", {
      "changelogFile": "CHANGELOG.md"
    }],
    ["@semantic-release/npm", {
      "npmPublish": false
    }],
    ["@semantic-release/exec", {
      "prepareCmd": "dotnet-setversion ${nextRelease.version} path/to/Project.csproj"
    }],
    ["@semantic-release/git", {
      "assets": ["CHANGELOG.md", "package.json", "path/to/Project.csproj"],
      "message": "chore(release): ${nextRelease.version} [skip ci]"
    }],
    "@semantic-release/github"
  ]
}
```

### 3. commitlint.config.js

```javascript
module.exports = {
  extends: ['@commitlint/config-conventional'],
  rules: {
    'type-enum': [2, 'always', [
      'feat', 'fix', 'docs', 'style', 'refactor',
      'perf', 'test', 'chore', 'ci', 'revert', 'build'
    ]],
    'header-max-length': [2, 'always', 100],
    'body-max-line-length': [0, 'always', Infinity]
  }
};
```

### 4. GitHub Actions: release.yml

```yaml
name: Release

on:
  push:
    branches: [main]
  workflow_dispatch:
    inputs:
      dry_run:
        description: 'Dry run mode'
        type: boolean
        default: false

permissions:
  contents: write
  issues: write
  pull-requests: write

jobs:
  release:
    runs-on: ubuntu-latest
    if: "!contains(github.event.head_commit.message, '[skip ci]')"
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
          persist-credentials: false

      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '8.0.x'

      - run: dotnet tool install -g dotnet-setversion
      - run: npm ci
      - run: npx commitlint --from=${{ github.event.before || 'HEAD~1' }} --to=HEAD
      
      - name: Release
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: npx semantic-release ${{ github.event.inputs.dry_run == 'true' && '--dry-run' || '' }}
```

### 5. GitHub Actions: commitlint.yml

```yaml
name: Commitlint

on:
  pull_request:
    branches: [main]

jobs:
  commitlint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci
      - run: npx commitlint --from ${{ github.event.pull_request.base.sha }} --to ${{ github.event.pull_request.head.sha }}
```

## First Release (Seed Tag)

Semantic-release needs a baseline tag:

```bash
git tag -a v1.0.0 -m "Initial release"
git push origin v1.0.0
gh release create v1.0.0 --title "v1.0.0" --notes "Initial release"
```

## Key Points

1. **No husky** - Keep commit linting in CI only to avoid dev friction
2. **`npmPublish: false`** - Don't publish to npm for private monorepos
3. **`@semantic-release/exec`** - Update .csproj version with dotnet-setversion
4. **`[skip ci]`** - Prevent infinite loops from release commits
5. **`persist-credentials: false`** - Required for semantic-release to push

## Version Bump Rules

| Commit Type | Version Bump |
|-------------|--------------|
| `feat:` | Minor (0.X.0) |
| `fix:`, `perf:`, `refactor:` | Patch (0.0.X) |
| `feat!:` or `BREAKING CHANGE:` | Major (X.0.0) |
| `docs:`, `style:`, `test:`, `chore:`, `ci:` | No release |

## Troubleshooting

- **No release created**: Check if commits since last tag have release-triggering types
- **Permission denied**: Ensure `GITHUB_TOKEN` has write permissions
- **dotnet-setversion not found**: Add `dotnet tool install -g dotnet-setversion` step
