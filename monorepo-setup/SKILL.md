---
name: monorepo-setup
description: 'Scaffold a production-ready monorepo from scratch using npm workspaces, TypeScript, conventional commits, and automated versioning. Use when initializing a new monorepo, adding a workspace app or package, setting up commitlint + Husky hooks, publishing multiple packages to npm, or wiring up a release workflow with standard-version and GitHub Actions CI/CD. Covers: workspace structure, publishable package scaffold, multi-package orchestration release script, per-package tag prefixes, Vitest test setup, Next.js web app and Express API workspace config, cross-workspace dependencies, and the full patch/minor/major release cycle.'
---

# Monorepo Setup (npm Workspaces + TypeScript + Conventional Commits)

This skill covers scaffolding and maintaining a production monorepo used in this project. It is grounded in real configuration files but uses sanitised values where needed.

## When to Use

- Initialise a new monorepo from scratch
- Add a new app workspace (`apps/web`, `apps/api`, etc.)
- Add a new publishable package workspace (`packages/*`)
- Set up or debug commitlint + Husky commit hooks
- Configure `standard-version` release workflow for a package
- **Release multiple packages simultaneously** from one root script
- Set up per-package tag namespacing to avoid tag collisions
- Set up GitHub Actions CI or a publish pipeline that routes by package tag
- Add Vitest test runner to a package
- Wire a local package into an app workspace without publishing

---

## Repository Structure

```
my-monorepo/
├── apps/
│   ├── web/              # Next.js SaaS dashboard  →  @my-monorepo/web
│   └── api/              # Express/Node API server  →  @my-monorepo/api
├── packages/
│   └── my-package/       # Publishable npm package  →  @your-org/my-package
├── .husky/               # Git hooks (commit-msg)
├── .gitignore
├── commitlint.config.js
├── package.json          # Root workspace config
└── package-lock.json
```

**Rules of thumb:**

- `apps/*` — private, not published to npm; each is an independently runnable service
- `packages/*` — publishable libraries; version bumps and changelogs live here
- Root `package.json` is always `"private": true`

---

## Architecture at a Glance

```
Root package.json  (workspaces: ["apps/*", "packages/*"])
│
├── apps/web        next dev / next build
├── apps/api        tsx watch / tsc + node
└── packages/       tsc → dist/  →  npm publish
    └── my-package  ← symlinked into apps/ via npm workspaces
```

Cross-workspace flow for a release:

```
conventional commit
     │
     ▼
commitlint hook validates
     │
     ▼
npm run release:minor  (in packages/my-package)
     │
     ├─ bumps package.json version
     ├─ appends CHANGELOG.md
     ├─ creates git tag  vX.Y.Z
     │
     ▼
git push --follow-tags
     │
     ▼
GitHub Actions → npm publish
```

---

## Key Mental Models

### 1. Workspace isolation — each app owns its devDependencies

Even though all packages share a single `node_modules` at the root (via hoisting), each workspace's `package.json` should list its own dependencies. Never add app-specific packages to the root.

### 2. Release lives in the package, not the root

`standard-version` reads commits since the last git tag. Tag and bump happen inside `packages/my-package`, not at the repo root. This lets multiple packages release independently.

### 3. Commit message = version bump signal

`standard-version` derives the SemVer bump purely from commit types since the last tag:

| Type | SemVer impact |
|---|---|
| `feat` | MINOR |
| `fix`, `perf` | PATCH |
| `feat!` / `BREAKING CHANGE` footer | MAJOR |
| `docs`, `refactor`, `test`, `chore` | No bump |

### 4. commitlint enforces the format at the git level

If the commit message does not match the conventional format, the `.husky/commit-msg` hook rejects the commit. This prevents changelog pollution from unstructured messages.

---

## Quick-Reference Commands

### Workspace commands (run from root)

```bash
# Run a script in one workspace
npm run <script> --workspace=apps/web

# Run a script in all workspaces (skip missing)
npm run <script> --workspaces --if-present

# Install a dep into a specific workspace
npm install <pkg> --workspace=packages/my-package

# Symlink a local package into an app workspace
npm install @your-org/my-package --workspace=apps/web
```

### Release commands (run from `packages/my-package`)

```bash
npm run release           # auto-detect bump from commits
npm run release:patch     # bug fixes only  → 0.1.0 → 0.1.1
npm run release:minor     # new features    → 0.1.0 → 0.2.0
npm run release:major     # breaking change → 0.1.0 → 1.0.0

npx standard-version --dry-run   # preview without writing

git push origin main --follow-tags   # push commits + tags
npm publish --access public --workspace=packages/my-package
```

### Undo a release

```bash
git tag -d v0.2.0          # delete tag locally
git push origin :v0.2.0    # delete tag on remote (confirm before running)
git reset --soft HEAD~1    # undo the release commit, keep changes staged
npm run release:minor      # redo the release
```

---

## Reference Files

| Topic | File |
|---|---|
| Root workspace config | [references/root-setup.md](references/root-setup.md) |
| Conventional commits + Husky | [references/conventional-commits.md](references/conventional-commits.md) |
| Publishable npm package + Vitest | [references/npm-package.md](references/npm-package.md) |
| Multi-package orchestration + tag prefixes | [references/multi-package-release.md](references/multi-package-release.md) |
| Next.js web app workspace | [references/web-app.md](references/web-app.md) |
| Express API workspace | [references/api-app.md](references/api-app.md) |
| Single-package release workflow | [references/release-workflow.md](references/release-workflow.md) |
| GitHub Actions CI/CD | [references/cicd.md](references/cicd.md) |
| Common pitfalls | [references/pitfalls.md](references/pitfalls.md) |
