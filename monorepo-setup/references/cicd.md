# GitHub Actions CI/CD

Two workflows cover the full lifecycle: continuous integration on every push, and npm publish on version tags.

## Workflow 1 — CI (lint, build, test on every push)

Create `.github/workflows/ci.yml`:

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:

env:
  HUSKY: 0    # disable Husky for all jobs

jobs:
  ci:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: "20"
          cache: "npm"

      - name: Install dependencies
        run: npm ci

      - name: Build
        run: npm run build --workspaces --if-present

      - name: Test
        run: npm run test --workspaces --if-present

      # Optional: only add if all workspaces define a lint script
      # - name: Lint
      #   run: npm run lint --workspaces --if-present
```

**Notes:**

- `npm ci` is preferred over `npm install` in CI — it installs from `package-lock.json` exactly, then deletes `node_modules` first.
- `--workspaces --if-present` runs a script across all workspaces and silently skips workspaces that don't define that script.
- `cache: "npm"` caches the npm cache directory for faster installs.

## Workflow 2 — Publish on tag push (per-package tags)

Create `.github/workflows/publish.yml`:

```yaml
name: Publish

on:
  push:
    tags:
      - "my-package@v*"
      - "my-package-addon@v*"
      # Add one entry per publishable package

jobs:
  publish:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write   # required for npm provenance
    env:
      HUSKY: 0          # disable Husky hooks in CI at the job level
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: "20"
          registry-url: "https://registry.npmjs.org"
          cache: "npm"

      - name: Install dependencies
        run: npm ci

      - name: Determine workspace from tag
        id: workspace
        run: |
          TAG="${GITHUB_REF_NAME}"
          PKG=$(echo "$TAG" | sed 's/@v.*//')
          echo "package=packages/${PKG}" >> $GITHUB_OUTPUT

      - name: Build package
        run: npm run build --workspace=${{ steps.workspace.outputs.package }}

      - name: Publish to npm
        run: npm publish --access public --workspace=${{ steps.workspace.outputs.package }} --provenance
        env:
          NODE_AUTH_TOKEN: ${{ secrets.NPM_TOKEN }}
```

**How tag routing works:**

- Tag `my-package-addon@v0.3.1` → `sed` strips `@v0.3.1` → `PKG=my-package-addon` → workspace = `packages/my-package-addon`.
- No hardcoded mapping — adding a package only needs a new `on.push.tags` entry.
- Tags are produced by the root orchestration script; see [multi-package-release.md](multi-package-release.md).

**Notes:**

- `registry-url` must be set for `NODE_AUTH_TOKEN` to be written to `.npmrc`.
- `--provenance` links the package to its GitHub Actions build (supply-chain transparency).
- Store your npm token as repo secret `NPM_TOKEN` (Settings → Secrets → Actions).
- `HUSKY: 0` at the job level is cleaner than per-step env and prevents Husky from running during `npm ci`.

## Skipping Husky in CI

Set `HUSKY: 0` at the **job** level (not per-step) so Husky's `prepare` script no-ops during `npm ci`:

```yaml
jobs:
  ci:
    env:
      HUSKY: 0
```

Alternatively, use the CI-safe prepare script in root `package.json` (both together is fine):

```json
{
  "scripts": {
    "prepare": "[ -d .git ] && husky || true"
  }
}
```

## Required repository secrets

| Secret | Where to get it |
|---|---|
| `NPM_TOKEN` | npm.com → Access Tokens → Automation token |

Set via GitHub repo → Settings → Secrets and variables → Actions → New repository secret.
