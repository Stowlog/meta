# Release Please Reusable Workflows — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Create two reusable GitHub Actions workflows in `stowlog/meta` that standardize release-please across all repos — one for independent repos (GitHub Releases only) and one for monorepos (manifest mode + optional npm publish).

**Architecture:** Both workflows use `google-github-actions/release-please-action@v4` as a `workflow_call` trigger. The standard workflow is a single job. The monorepo workflow has two jobs: `release` (always runs) and `publish` (conditional on `inputs.publish-npm` and `steps.release.outputs.releases_created`). Caller repos pass `secrets: inherit` and use minimal `with:` overrides.

**Tech Stack:** GitHub Actions, `google-github-actions/release-please-action@v4`, Node.js, npm/pnpm

---

## File Map

| File | Action | Responsibility |
|---|---|---|
| `.github/workflows/release-please-standard.yml` | Create | Reusable workflow for independent repos |
| `.github/workflows/release-please-monorepo.yml` | Create | Reusable workflow for monorepos (manifest mode + optional npm publish) |
| `docs/examples/release-standard-caller.yml` | Create | Reference caller for independent repos |
| `docs/examples/release-monorepo-npm-caller.yml` | Create | Reference caller for monorepos with npm publish |
| `docs/examples/release-monorepo-caller.yml` | Create | Reference caller for monorepos without npm publish |
| `README.md` | Modify | Document available workflows and usage |

---

## Task 1: Create `release-please-standard.yml`

**Files:**
- Create: `.github/workflows/release-please-standard.yml`

- [ ] **Step 1: Create the directory**

```bash
mkdir -p .github/workflows
```

- [ ] **Step 2: Write the workflow**

Create `.github/workflows/release-please-standard.yml`:

```yaml
name: Release Please (Standard)

on:
  workflow_call:
    inputs:
      target-branch:
        description: Branch to track for releases
        type: string
        default: main
      release-type:
        description: release-please release type (node, simple, etc.)
        type: string
        default: node
      changelog-types:
        description: JSON array string of extra conventional commit types for changelog
        type: string
        default: ''
    secrets:
      RELEASE_PLEASE_TOKEN:
        description: GitHub token with write permissions (falls back to GITHUB_TOKEN)
        required: false

permissions:
  contents: write
  pull-requests: write

jobs:
  release-please:
    runs-on: ubuntu-latest
    steps:
      - uses: google-github-actions/release-please-action@v4
        with:
          token: ${{ secrets.RELEASE_PLEASE_TOKEN || secrets.GITHUB_TOKEN }}
          release-type: ${{ inputs.release-type }}
          target-branch: ${{ inputs.target-branch }}
          changelog-types: ${{ inputs.changelog-types }}
```

- [ ] **Step 3: Commit**

```bash
git add .github/workflows/release-please-standard.yml
git commit -m "feat: add release-please-standard reusable workflow"
```

---

## Task 2: Create `release-please-monorepo.yml`

**Files:**
- Create: `.github/workflows/release-please-monorepo.yml`

- [ ] **Step 1: Write the workflow**

Create `.github/workflows/release-please-monorepo.yml`:

```yaml
name: Release Please (Monorepo)

on:
  workflow_call:
    inputs:
      target-branch:
        description: Branch to track for releases
        type: string
        default: main
      publish-npm:
        description: Whether to publish packages to npm after a release
        type: boolean
        default: false
      node-version:
        description: Node.js version used during npm publish
        type: string
        default: '20'
      package-manager:
        description: Package manager to use for install and publish (npm or pnpm)
        type: string
        default: npm
    secrets:
      RELEASE_PLEASE_TOKEN:
        description: GitHub token with write permissions (falls back to GITHUB_TOKEN)
        required: false
      NPM_TOKEN:
        description: npm auth token — required when publish-npm is true
        required: false

permissions:
  contents: write
  pull-requests: write

jobs:
  release-please:
    runs-on: ubuntu-latest
    outputs:
      releases_created: ${{ steps.release.outputs.releases_created }}
      paths_released: ${{ steps.release.outputs.paths_released }}
    steps:
      - uses: google-github-actions/release-please-action@v4
        id: release
        with:
          token: ${{ secrets.RELEASE_PLEASE_TOKEN || secrets.GITHUB_TOKEN }}
          target-branch: ${{ inputs.target-branch }}
          # Manifest mode is inferred automatically when release-please-config.json exists

  publish:
    needs: release-please
    if: ${{ inputs.publish-npm && needs.release-please.outputs.releases_created == 'true' }}
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ inputs.node-version }}
          registry-url: https://registry.npmjs.org

      - name: Setup pnpm
        if: ${{ inputs.package-manager == 'pnpm' }}
        uses: pnpm/action-setup@v4
        with:
          run_install: false

      - name: Get pnpm store path
        if: ${{ inputs.package-manager == 'pnpm' }}
        id: pnpm-cache
        run: echo "path=$(pnpm store path)" >> $GITHUB_OUTPUT

      - name: Cache pnpm store
        if: ${{ inputs.package-manager == 'pnpm' }}
        uses: actions/cache@v4
        with:
          path: ${{ steps.pnpm-cache.outputs.path }}
          key: ${{ runner.os }}-pnpm-${{ hashFiles('**/pnpm-lock.yaml') }}
          restore-keys: ${{ runner.os }}-pnpm-

      - name: Install dependencies (pnpm)
        if: ${{ inputs.package-manager == 'pnpm' }}
        run: pnpm install --frozen-lockfile

      - name: Install dependencies (npm)
        if: ${{ inputs.package-manager == 'npm' }}
        run: npm ci

      - name: Publish packages (pnpm)
        if: ${{ inputs.package-manager == 'pnpm' }}
        run: pnpm publish -r --access public --no-git-checks
        env:
          NODE_AUTH_TOKEN: ${{ secrets.NPM_TOKEN }}

      - name: Publish packages (npm)
        if: ${{ inputs.package-manager == 'npm' }}
        run: |
          PATHS='${{ needs.release-please.outputs.paths_released }}'
          echo "$PATHS" | jq -r '.[]' | while read pkg_path; do
            echo "Publishing $pkg_path"
            cd "$pkg_path"
            npm publish --access public
            cd -
          done
        env:
          NODE_AUTH_TOKEN: ${{ secrets.NPM_TOKEN }}
```

- [ ] **Step 2: Commit**

```bash
git add .github/workflows/release-please-monorepo.yml
git commit -m "feat: add release-please-monorepo reusable workflow"
```

---

## Task 3: Create example caller workflows

**Files:**
- Create: `docs/examples/release-standard-caller.yml`
- Create: `docs/examples/release-monorepo-npm-caller.yml`
- Create: `docs/examples/release-monorepo-caller.yml`

- [ ] **Step 1: Create examples directory**

```bash
mkdir -p docs/examples
```

- [ ] **Step 2: Write standard caller example**

Create `docs/examples/release-standard-caller.yml`:

```yaml
# Copy this file to .github/workflows/release.yml in your repo
# Adjust 'with:' inputs as needed — all have sensible defaults

name: Release

on:
  push:
    branches:
      - main  # change to 'production' if needed

jobs:
  release:
    uses: stowlog/meta/.github/workflows/release-please-standard.yml@main
    secrets: inherit
    # Optional overrides:
    # with:
    #   target-branch: production
    #   release-type: node
    #   changelog-types: '[{"type":"chore","section":"Miscellaneous","hidden":false}]'
```

- [ ] **Step 3: Write monorepo + npm publish caller example**

Create `docs/examples/release-monorepo-npm-caller.yml`:

```yaml
# Copy this to .github/workflows/release.yml in a monorepo that publishes to npm
# Requires in repo root:
#   - release-please-config.json
#   - .release-please-manifest.json
# Requires repository secret: NPM_TOKEN

name: Release

on:
  push:
    branches:
      - main

jobs:
  release:
    uses: stowlog/meta/.github/workflows/release-please-monorepo.yml@main
    secrets: inherit
    with:
      publish-npm: true
      package-manager: pnpm   # or npm
      node-version: '20'
```

- [ ] **Step 4: Write monorepo (no publish) caller example**

Create `docs/examples/release-monorepo-caller.yml`:

```yaml
# Copy this to .github/workflows/release.yml in a monorepo that only needs GitHub Releases
# Requires in repo root:
#   - release-please-config.json
#   - .release-please-manifest.json

name: Release

on:
  push:
    branches:
      - main

jobs:
  release:
    uses: stowlog/meta/.github/workflows/release-please-monorepo.yml@main
    secrets: inherit
```

- [ ] **Step 5: Commit**

```bash
git add docs/examples/
git commit -m "docs: add example caller workflows for release-please"
```

---

## Task 4: Update README

**Files:**
- Modify: `README.md`

- [ ] **Step 1: Replace README content**

Replace `README.md` with:

```markdown
# meta

Stowlog shared GitHub Actions workflows and configuration.

## Reusable Workflows

### `release-please-standard.yml` — Independent repos

Tracks conventional commits, opens a Release PR, and creates a GitHub Release on merge.

**Usage** — copy to `.github/workflows/release.yml` in your repo:

\`\`\`yaml
name: Release

on:
  push:
    branches: [main]

jobs:
  release:
    uses: stowlog/meta/.github/workflows/release-please-standard.yml@main
    secrets: inherit
    # with:
    #   target-branch: main          # default
    #   release-type: node           # default
    #   changelog-types: ''          # default (empty)
\`\`\`

---

### `release-please-monorepo.yml` — Monorepos

Manifest-based releases across multiple packages. Optionally publishes to npm.

**Requires in repo root:**
- `release-please-config.json`
- `.release-please-manifest.json`

**Usage — GitHub Releases only** (e.g. `stowlog/kiosks`):

\`\`\`yaml
name: Release

on:
  push:
    branches: [main]

jobs:
  release:
    uses: stowlog/meta/.github/workflows/release-please-monorepo.yml@main
    secrets: inherit
\`\`\`

**Usage — with npm publish** (e.g. `stowlog/foundations`):

\`\`\`yaml
name: Release

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
\`\`\`

> Requires `NPM_TOKEN` secret in the repo.

---

## Inputs Reference

### `release-please-standard.yml`

| Input | Type | Default | Description |
|---|---|---|---|
| `target-branch` | string | `main` | Branch to track |
| `release-type` | string | `node` | release-please type |
| `changelog-types` | string | `''` | JSON array of extra commit types |

### `release-please-monorepo.yml`

| Input | Type | Default | Description |
|---|---|---|---|
| `target-branch` | string | `main` | Branch to track |
| `publish-npm` | boolean | `false` | Publish packages to npm |
| `node-version` | string | `20` | Node.js version for publish job |
| `package-manager` | string | `npm` | `npm` or `pnpm` |

---

## Examples

See [`docs/examples/`](./docs/examples/) for ready-to-copy caller workflows.
```

- [ ] **Step 2: Commit**

```bash
git add README.md
git commit -m "docs: update README with reusable workflows documentation"
```

---

## Notes for agents

- Tasks 1 and 2 are **independent** — run in parallel.
- Tasks 3 and 4 depend on nothing — can also run in parallel with 1 and 2.
- No tests needed: these are GitHub Actions YAML files — correctness is validated by YAML linting and manual trigger in a consuming repo.
- The `RELEASE_PLEASE_TOKEN || secrets.GITHUB_TOKEN` pattern uses GitHub's expression fallback — note that `secrets.GITHUB_TOKEN` is always available without being declared.
- For the monorepo manifest mode, `release-please-action@v4` automatically detects manifest mode when `release-please-config.json` exists in the repo root.
