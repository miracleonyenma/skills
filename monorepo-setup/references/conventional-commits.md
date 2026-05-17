# Conventional Commits + commitlint + Husky

Enforcing the [Conventional Commits](https://www.conventionalcommits.org/) spec at the git-hook level ensures `standard-version` can always generate accurate changelogs and derive the correct SemVer bump automatically.

## Install (at the root)

```bash
npm install --save-dev @commitlint/cli @commitlint/config-conventional husky
```

## `commitlint.config.js`

```js
// commitlint.config.js  (root)
export default {
  extends: ["@commitlint/config-conventional"],
  rules: {
    // Disable line-length limits on body and footer (useful for long BREAKING CHANGE notes)
    "body-max-line-length":   [0],
    "footer-max-line-length": [0],
    // Relax the default 100-char header limit to 120
    "header-max-length": [2, "always", 120],
  },
};
```

Or as a minimal `.commitlintrc.json` (no custom rules):

```json
{ "extends": ["@commitlint/config-conventional"] }
```

## Initialise Husky and add the commit-msg hook

```bash
npx husky init
echo 'npx --no -- commitlint --edit "$1"' > .husky/commit-msg
chmod +x .husky/commit-msg
```

After this, any `git commit` with a non-conforming message is rejected before it lands in history.

## Commit message format

```
<type>(<optional-scope>): <subject>

[optional body]

[optional footer(s)]
```

### Type reference

| Type | SemVer impact | When to use |
|---|---|---|
| `feat` | **MINOR** | New user-facing feature |
| `fix` | **PATCH** | Bug fix |
| `perf` | **PATCH** | Performance improvement (no API change) |
| `docs` | none | Documentation only |
| `refactor` | none | Internal restructure, no behaviour change |
| `test` | none | Adding or fixing tests |
| `chore` | none | Build process, tooling, deps |
| `style` | none | Formatting, whitespace — no logic change |
| `ci` | none | CI config changes |
| `feat!` / `BREAKING CHANGE` footer | **MAJOR** | Breaking API change |

### Examples

```bash
# New feature → MINOR bump
git commit -m "feat(api): add user authentication endpoint"

# Bug fix → PATCH bump
git commit -m "fix(web): correct token expiration check"

# Breaking change → MAJOR bump
git commit -m "feat!: rename CLI command syntax

BREAKING CHANGE: 'init-cmd' is now 'setup'"

# Internal — no version bump
git commit -m "chore(deps): upgrade express to v5"
git commit -m "docs: update README with new env vars"
```

## Scope conventions (recommended)

Define scope values that map to workspace boundaries:

| Scope | Workspace |
|---|---|
| `api` | `apps/api` |
| `web` | `apps/web` |
| `pkg` | `packages/my-package` |
| `ci` | `.github/workflows` |
| `deps` | dependency updates (any workspace) |

## Verifying the hook works

```bash
# Should be rejected
git commit -m "added stuff"

# Should pass
git commit -m "fix(api): handle null user in auth middleware"
```

## Troubleshooting

- **Hook not running:** Ensure Husky is initialised (`npx husky init`) and the `.husky/commit-msg` file is executable (`chmod +x .husky/commit-msg`).
- **`prepare` script missing:** npm runs the `prepare` lifecycle on `npm install`. Husky adds it automatically; check that `"prepare": "husky"` is in the root `package.json` scripts.
- **CI bypass:** CI environments should skip hooks. Add `HUSKY=0` to the CI environment or use `git commit --no-verify` only in automated release commits.
