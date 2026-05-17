# Publishable npm Package Workspace

This covers scaffolding a TypeScript package that is published to npm from within the monorepo.

## Scaffold

```bash
mkdir -p packages/my-package/src
cd packages/my-package
npm init -y
```

## `packages/my-package/package.json`

```json
{
  "name": "@your-org/my-package",
  "version": "0.1.0",
  "description": "My reusable package",
  "type": "module",
  "main": "./dist/index.js",
  "types": "./dist/index.d.ts",
  "exports": {
    ".": {
      "import": "./dist/index.js",
      "types": "./dist/index.d.ts"
    }
  },
  "files": [
    "dist",
    "README.md",
    "CHANGELOG.md"
  ],
  "scripts": {
    "build":          "tsc",
    "dev":            "tsx watch src/index.ts",
    "clean":          "rm -rf dist",
    "test":           "vitest run",
    "test:watch":     "vitest",
    "prepublishOnly": "npm run build",
    "release":        "standard-version",
    "release:patch":  "standard-version --release-as patch",
    "release:minor":  "standard-version --release-as minor",
    "release:major":  "standard-version --release-as major"
  },
  "publishConfig": {
    "access": "public"
  },
  "license": "MIT",
  "devDependencies": {
    "standard-version": "^9.5.0",
    "typescript":       "^5.0.0",
    "tsx":              "^4.7.0",
    "@types/node":      "^20.0.0",
    "vitest":           "^1.5.0"
  },
  "engines": {
    "node": ">=20.0.0"
  }
}
```

**Key fields:**

- `"type": "module"` — emit ESM. Omit (or use `"commonjs"`) if you need CJS consumers.
- `exports` — preferred over `main` for dual ESM/CJS or subpath exports.
- `files` — whitelist what goes into the tarball. Always include `dist` and `README.md`.
- `prepublishOnly` — guarantees a fresh build before every `npm publish`.

## `packages/my-package/tsconfig.json`

```json
{
  "extends": "../../tsconfig.json",
  "compilerOptions": {
    "declaration":    true,
    "declarationMap": true,
    "sourceMap":      true,
    "outDir":         "./dist",
    "rootDir":        "./src"
  },
  "include":  ["src/**/*"],
  "exclude":  ["node_modules", "dist"]
}
```

If not using a root base config, expand the `compilerOptions` in full:

```json
{
  "compilerOptions": {
    "target":                     "ES2020",
    "module":                     "ES2020",
    "lib":                        ["ES2020"],
    "moduleResolution":           "node",
    "esModuleInterop":            true,
    "allowSyntheticDefaultImports": true,
    "strict":                     true,
    "skipLibCheck":               true,
    "declaration":                true,
    "declarationMap":             true,
    "sourceMap":                  true,
    "outDir":                     "./dist",
    "rootDir":                    "./src"
  },
  "include":  ["src/**/*"],
  "exclude":  ["node_modules", "dist"]
}
```

## Source entry point

```typescript
// src/index.ts
export const hello = (name: string): string => `Hello, ${name}!`;
```

## `.versionrc.json` — changelog configuration

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
  "commitUrlFormat":          "https://github.com/YOUR_ORG/YOUR_REPO/commit/{{hash}}",
  "compareUrlFormat":         "https://github.com/YOUR_ORG/YOUR_REPO/compare/{{previousTag}}...{{currentTag}}",
  "issueUrlFormat":           "https://github.com/YOUR_ORG/YOUR_REPO/issues/{{id}}",
  "userUrlFormat":            "https://github.com/{{user}}",
  "releaseCommitMessageFormat": "chore(release): my-package@v{{currentTag}}",
  "issuePrefixes":            ["#"],
  "tagPrefix": "my-package@v"
}
```

Replace `YOUR_ORG/YOUR_REPO` and `my-package` with your actual values.

**Critical:** `tagPrefix` must match the package directory name followed by `@v`. This prevents tag collisions between packages in the same repo and allows the CI publish workflow to route by tag. See [multi-package-release.md](multi-package-release.md) for details.

**Tip:** `"hidden": false` on `refactor` and `docs` surfaces them in the changelog — useful for internal consumers tracking API documentation updates.

## Initialise `CHANGELOG.md`

```bash
cat > packages/my-package/CHANGELOG.md << 'EOF'
# Changelog

All notable changes to this project will be documented in this file. See [standard-version](https://github.com/conventional-changelog/standard-version) for commit guidelines.
EOF
```

## Install dependencies

```bash
# From the monorepo root
npm install --workspace=packages/my-package
```

## Using the package in a sibling app workspace

```bash
# Symlink the local package — no publishing needed
npm install @your-org/my-package --workspace=apps/web
```

Import normally:

```typescript
import { hello } from "@your-org/my-package";
```

npm workspaces creates a symlink in `node_modules/@your-org/my-package` pointing to `packages/my-package`. Changes in the package are immediately reflected (after a build) in the consuming app.

## `vitest.config.ts` — test setup

Vitest is the recommended test runner for packages in this monorepo. Add it alongside TypeScript:

```typescript
// packages/my-package/vitest.config.ts
import { defineConfig } from "vitest/config";

export default defineConfig({
  test: {
    globals: false,         // set true if you want describe/it/expect without imports
    environment: "node",    // "jsdom" for browser-like tests
  },
});
```

Write tests in `tests/`:

```typescript
// packages/my-package/tests/index.test.ts
import { describe, it, expect } from "vitest";
import { hello } from "../src/index.js";

describe("hello", () => {
  it("returns a greeting", () => {
    expect(hello("world")).toBe("Hello, world!");
  });
});
```

Run from the workspace:

```bash
# Single run (used in CI)
npm run test --workspace=packages/my-package

# Watch mode during development
npm run test:watch --workspace=packages/my-package

# All packages from root
npm run test --workspaces --if-present
```

**Note:** When importing from your own package's source in tests, use the `.js` extension even for `.ts` files — this is required for Node16/ESM resolution:

```typescript
import { hello } from "../src/index.js";   // correct
import { hello } from "../src/index";       // will fail with ESM
```
