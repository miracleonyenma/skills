# Root Workspace Setup

## Initialise the repo

```bash
mkdir my-monorepo && cd my-monorepo
git init
npm init -y
```

## Directory structure

```bash
mkdir -p apps/web apps/api packages/my-package
```

## Root `package.json`

```json
{
  "name": "my-monorepo",
  "version": "1.0.0",
  "private": true,
  "type": "module",
  "description": "My monorepo",
  "workspaces": [
    "apps/*",
    "packages/*"
  ],
  "scripts": {
    "dev":           "npm run dev --workspace=apps/web",
    "build":         "npm run build --workspaces --if-present",
    "start":         "npm run start --workspace=apps/web",
    "lint":          "npm run lint --workspaces --if-present",
    "test":          "npm run test --workspaces --if-present",
    "prepare":       "[ -d .git ] && husky || true",
    "release":       "node scripts/release.mjs",
    "release:patch": "node scripts/release.mjs patch",
    "release:minor": "node scripts/release.mjs minor",
    "release:major": "node scripts/release.mjs major",
    "publish:all":   "for pkg in my-package my-package-addon; do npm publish --access public --workspace=packages/$pkg; done"
  },
  "engines": {
    "node": ">=20.0.0"
  },
  "devDependencies": {
    "@commitlint/cli":                 "^19.0.0",
    "@commitlint/config-conventional": "^19.0.0",
    "conventional-changelog-cli":      "^5.0.0",
    "husky":                           "^9.0.0",
    "typescript":                      "^5.4.0"
  }
}
```

**Key rules:**

- `"private": true` is mandatory at the root — prevents accidental `npm publish` of the root.
- `"type": "module"` enables ESM for root-level scripts (e.g. `scripts/release.mjs`). Omit if you prefer CJS.
- `workspaces` globs are evaluated relative to the root; add new workspace directories here.
- Root scripts use `--workspace=` or `--workspaces` flags to delegate to child packages.
- Never add app-specific runtime dependencies to the root `package.json`.
- The `prepare` script uses `[ -d .git ] && husky || true` so it safely no-ops in CI where `.git` may be absent.
- Release scripts delegate to `scripts/release.mjs` — see [multi-package-release.md](multi-package-release.md) for the full orchestration pattern.

## Root `tsconfig.json` (optional base config)

If you share TypeScript settings across workspaces, put a base config at the root and extend it in each workspace:

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "ES2020",
    "moduleResolution": "bundler",
    "esModuleInterop": true,
    "allowSyntheticDefaultImports": true,
    "strict": true,
    "skipLibCheck": true
  }
}
```

Each workspace then extends it:

```json
{
  "extends": "../../tsconfig.json",
  "compilerOptions": {
    "outDir": "./dist",
    "rootDir": "./src"
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

## `.gitignore`

```
node_modules/
.next/
dist/
*.log
.env
.env*.local
.DS_Store
```

## Installing dependencies

```bash
# Root-level tooling (commitlint, husky, etc.)
npm install --save-dev <pkg>

# Into a specific workspace
npm install <pkg> --workspace=apps/api

# All workspaces at once (after cloning)
npm install
```

npm workspaces hoist shared dependencies to the root `node_modules` automatically.
