# Multi-Package Release Orchestration

When a monorepo contains **multiple independently publishable packages** under `packages/`, the single-package `standard-version` workflow no longer scales. Use a root-level orchestration script instead.

## The problem with per-package manual releases

- Running `standard-version` in each package individually requires switching directories and repeating steps.
- Tags must not clash between packages: `v1.0.0` from `packages/foo` conflicts with `v1.0.0` from `packages/bar`.
- The publish CI workflow must know _which_ package a tag belongs to.

## Solution: per-package tag prefix + root orchestration script

### Tag convention

Each package owns a tag namespace prefixed by its directory name:

```
pay@v1.2.0
pay-100pay@v0.3.1
pay-mongoose@v0.2.0
```

This means `standard-version` for `packages/pay` always uses `--tag-prefix pay@v`.

### `.versionrc.json` per package

Every publishable package needs its own `.versionrc.json` that sets `tagPrefix` and a matching `releaseCommitMessageFormat`:

```json
{
  "types": [
    { "type": "feat",     "section": "Features",        "hidden": false },
    { "type": "fix",      "section": "Bug Fixes",        "hidden": false },
    { "type": "perf",     "section": "Performance",      "hidden": false },
    { "type": "docs",     "section": "Documentation",    "hidden": false },
    { "type": "refactor", "section": "Code Refactoring", "hidden": false },
    { "type": "style",    "section": "Styling",          "hidden": true  },
    { "type": "test",     "section": "Tests",            "hidden": true  },
    { "type": "chore",    "section": "Chores",           "hidden": true  }
  ],
  "commitUrlFormat":            "https://github.com/YOUR_ORG/YOUR_REPO/commit/{{hash}}",
  "compareUrlFormat":           "https://github.com/YOUR_ORG/YOUR_REPO/compare/{{previousTag}}...{{currentTag}}",
  "issueUrlFormat":             "https://github.com/YOUR_ORG/YOUR_REPO/issues/{{id}}",
  "userUrlFormat":              "https://github.com/{{user}}",
  "releaseCommitMessageFormat": "chore(release): my-package@v{{currentTag}}",
  "issuePrefixes":              ["#"],
  "tagPrefix":                  "my-package@v"
}
```

Replace `my-package` with the directory name of the package (e.g. `pay`, `pay-mongoose`).

---

## Root orchestration script (`scripts/release.mjs`)

Place this at the monorepo root. It auto-discovers all non-private packages, bumps each one, pushes tags, then publishes.

```js
#!/usr/bin/env node
/**
 * Monorepo release script
 * Usage: node scripts/release.mjs [patch|minor|major]
 *
 * 1. Bumps version + generates changelog for every publishable package
 * 2. Pushes commits and tags to origin/main
 * 3. Publishes each package to npm
 */

import { execSync } from "child_process";
import { readdirSync, existsSync } from "fs";
import { resolve, join } from "path";
import { createRequire } from "module";

const require = createRequire(import.meta.url);
const root = resolve(new URL(".", import.meta.url).pathname, "..");

const releaseAs = process.argv[2] ?? "patch";
if (!["patch", "minor", "major"].includes(releaseAs)) {
  console.error(`Invalid release type: "${releaseAs}". Use patch, minor, or major.`);
  process.exit(1);
}

function run(cmd, cwd = root) {
  console.log(`\n$ ${cmd}`);
  execSync(cmd, { cwd, stdio: "inherit" });
}

// Auto-discover publishable packages (no "private": true)
const packagesDir = join(root, "packages");
const packages = readdirSync(packagesDir)
  .filter((dir) => {
    const pkgPath = join(packagesDir, dir, "package.json");
    if (!existsSync(pkgPath)) return false;
    const pkg = require(pkgPath);
    return !pkg.private;
  })
  .map((dir) => ({ dir }));

console.log(`\n=== Releasing ${packages.length} packages as ${releaseAs} ===\n`);
console.log(packages.map((p) => `  packages/${p.dir}`).join("\n"));

// Step 1: bump versions + changelogs
for (const { dir } of packages) {
  const pkgDir = join(packagesDir, dir);
  const tagPrefix = `${dir}@v`;
  console.log(`\n--- standard-version: ${dir} (prefix: ${tagPrefix}) ---`);
  run(
    `npx standard-version --release-as ${releaseAs} --tag-prefix ${tagPrefix}`,
    pkgDir
  );
}

// Step 2: push commits + tags
console.log("\n--- Pushing commits and tags ---");
run("git push origin main --follow-tags");

// Step 3: publish
console.log("\n--- Publishing packages ---");
for (const { dir } of packages) {
  console.log(`\n--- Publishing ${dir} ---`);
  run(`npm publish --access public --workspace=packages/${dir}`);
}

console.log("\n✅ Release complete!");
```

### Wire it into the root `package.json`

```json
{
  "scripts": {
    "release":        "node scripts/release.mjs",
    "release:patch":  "node scripts/release.mjs patch",
    "release:minor":  "node scripts/release.mjs minor",
    "release:major":  "node scripts/release.mjs major",
    "publish:all":    "for pkg in my-package my-package-addon; do npm publish --access public --workspace=packages/$pkg; done"
  }
}
```

`publish:all` is a convenience script for re-publishing without a version bump (e.g. after a failed CI publish).

---

## GitHub Actions publish workflow (per-package tags)

The publish workflow must match all per-package tag patterns and dynamically route to the correct workspace:

```yaml
name: Publish

on:
  push:
    tags:
      - "my-package@v*"
      - "my-package-addon@v*"
      # add one line per package

jobs:
  publish:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write   # required for npm provenance
    env:
      HUSKY: 0
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

**How the tag routing works:**

- Tag `pay-mongoose@v0.3.1` → `sed` strips `@v0.3.1` → `PKG=pay-mongoose` → workspace = `packages/pay-mongoose`.
- No hardcoded mapping needed; adding a new package only requires a new tag pattern in the `on.push.tags` list.

---

## Dry run before a real release

Always preview before writing:

```bash
cd packages/my-package
npx standard-version --dry-run --tag-prefix my-package@v
```

Or preview the full orchestration script without pushing/publishing (comment out Steps 2 and 3 in `release.mjs`, or add a `--dry` flag).

---

## Adding a new package to the release pipeline

1. Add `.versionrc.json` to `packages/new-package/` with the correct `tagPrefix` and `releaseCommitMessageFormat`.
2. Add `"new-package@v*"` to the `on.push.tags` list in `.github/workflows/publish.yml`.
3. Ensure the package does **not** have `"private": true` in its `package.json` (the script skips private packages).
4. Create an initial tag manually for the first release:

   ```bash
   git tag new-package@v0.1.0
   git push origin new-package@v0.1.0
   ```
