# meta

Stowlog shared GitHub Actions workflows and configuration.

## Reusable Workflows

### `release-please-standard.yml` — Independent repos

Tracks conventional commits, opens a Release PR, and creates a GitHub Release on merge.

**Usage** — copy to `.github/workflows/release.yml` in your repo:

```yaml
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
```

---

### `release-please-monorepo.yml` — Monorepos

Manifest-based releases across multiple packages. Optionally publishes to npm.

**Requires in repo root:**
- `release-please-config.json`
- `.release-please-manifest.json`

**Usage — GitHub Releases only** (e.g. `stowlog/kiosks`):

```yaml
name: Release

on:
  push:
    branches: [main]

jobs:
  release:
    uses: stowlog/meta/.github/workflows/release-please-monorepo.yml@main
    secrets: inherit
```

**Usage — with npm publish** (e.g. `stowlog/foundations`):

```yaml
name: Release

on:
  push:
    branches: [main]
  workflow_dispatch:

jobs:
  release:
    uses: stowlog/meta/.github/workflows/release-please-monorepo.yml@main
    secrets: inherit
    with:
      publish-npm: true
      package-manager: pnpm
      build-command: 'pnpm build'
```

> Requires the `npm` GitHub environment configured with npm OIDC (Trusted Publishers). No secrets needed.

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
| `build-command` | string | `''` | Command to run before publishing (e.g. `pnpm build`) |

---

## Examples

See [`docs/examples/`](./docs/examples/) for ready-to-copy caller workflows.
