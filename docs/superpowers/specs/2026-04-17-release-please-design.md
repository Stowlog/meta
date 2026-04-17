# Release Please — Reusable Workflows Design

**Date:** 2026-04-17  
**Repo:** stowlog/meta  
**Status:** Approved

## Goal

Standardize GitHub releases across all Stowlog repositories using reusable GitHub Actions workflows backed by `release-please`. Repos import only what they need — no copy-paste maintenance.

## Context

- All repos are npm/Node.js
- Two monorepos: `stowlog/foundations` (pnpm, publishes to npm) and `stowlog/kiosks` (npm, internal packages, GitHub Releases only)
- Remaining repos are independent and only need GitHub Releases + tags + changelogs
- Standardizing on `main` as default branch; `target-branch` input supports migration from `production`

## Architecture

Two reusable workflows live in `stowlog/meta`:

```
meta/
└── .github/
    └── workflows/
        ├── release-please-standard.yml   ← independent repos
        └── release-please-monorepo.yml   ← monorepos (manifest-based)
```

Consumed via `uses: stowlog/meta/.github/workflows/<file>.yml@main` with `secrets: inherit`.

## Workflow: `release-please-standard.yml`

**Use case:** Independent repos that need GitHub Releases, tags, and changelogs. No npm publish.

**Trigger:** `workflow_call`

**Inputs:**

| Name | Type | Default | Description |
|---|---|---|---|
| `target-branch` | string | `main` | Branch to track for releases |
| `release-type` | string | `node` | release-please release type |
| `changelog-types` | string | `""` | JSON array of extra changelog commit types |

**Behavior:**
1. Runs `google-github-actions/release-please-action@v4` on every push to `target-branch`
2. Creates/updates a Release PR with auto-generated changelog from conventional commits
3. On PR merge: creates a GitHub Release and tag

**Caller example:**
```yaml
on:
  push:
    branches: [main]

jobs:
  release:
    uses: stowlog/meta/.github/workflows/release-please-standard.yml@main
    secrets: inherit
```

## Workflow: `release-please-monorepo.yml`

**Use case:** Monorepos using release-please manifest mode. Supports optional npm publish for packages like `@stowlog/cli`, `@stowlog/ui`, `@stowlog/biome-config`.

**Trigger:** `workflow_call`

**Inputs:**

| Name | Type | Default | Description |
|---|---|---|---|
| `target-branch` | string | `main` | Branch to track for releases |
| `publish-npm` | boolean | `false` | Whether to publish packages to npm after release |
| `node-version` | string | `20` | Node.js version for publish job |
| `package-manager` | string | `npm` | `npm` or `pnpm` |

**Secrets (via `secrets: inherit`):**
- `NPM_TOKEN` — required only when `publish-npm: true`
- `GITHUB_TOKEN` — automatic

**Behavior:**
1. Runs `release-please` in manifest mode using `release-please-config.json` + `.release-please-manifest.json` from the calling repo root
2. On release created: if `publish-npm: true`, installs deps with the configured package manager and runs `npm publish` (or workspace publish) for each released package

**Required files in the consuming monorepo:**
- `release-please-config.json` — defines packages and their release types
- `.release-please-manifest.json` — tracks current versions per package

**Caller example (stowlog/foundations):**
```yaml
on:
  push:
    branches: [main]

jobs:
  release:
    uses: stowlog/meta/.github/workflows/release-please-monorepo.yml@main
    secrets: inherit
    with:
      publish-npm: true
      package-manager: pnpm
```

**Caller example (stowlog/kiosks):**
```yaml
on:
  push:
    branches: [main]

jobs:
  release:
    uses: stowlog/meta/.github/workflows/release-please-monorepo.yml@main
    secrets: inherit
```

## Conventional Commits Convention

Both workflows rely on conventional commits to generate changelogs:

| Prefix | Changelog Section |
|---|---|
| `feat:` | Features |
| `fix:` | Bug Fixes |
| `perf:` | Performance |
| `docs:` | Documentation |
| `chore!:` / `feat!:` | Breaking Changes |

## Migration Path

Repos on `production` branch pass `target-branch: production` until migrated to `main`.

## Out of Scope

- Publishing to private npm registries (can be added later via input)
- Python, Go, or other language stacks
- Automated PR labeling or additional status checks
