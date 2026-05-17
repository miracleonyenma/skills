# Common Pitfalls and Fixes

## 1) Package not found after adding workspace

**Problem:** `Cannot find module '@your-org/my-package'` in an app workspace despite the package existing locally.

**Fix:**

```bash
# Re-link workspaces (run from root)
npm install

# Or explicitly install into the app
npm install @your-org/my-package --workspace=apps/web
```

npm workspaces only symlinks a package into `node_modules` if it is declared as a dependency in the consuming workspace's `package.json`. Running `npm install` at the root rebuilds all symlinks.

---

## 2) ESM package not resolved in consuming app

**Problem:** Next.js or Express throws `ERR_REQUIRE_ESM` or cannot find the package exports.

**Fix — Next.js:**
Add the package to `transpilePackages` in `next.config.mjs`:

```js
const nextConfig = {
  transpilePackages: ["@your-org/my-package"],
};
```

**Fix — TypeScript resolution:**
Use `"moduleResolution": "node16"` or `"bundler"` in the consuming app's `tsconfig.json` to correctly resolve `package.json` `exports` fields.

---

## 3) `standard-version` bumps the wrong version

**Problem:** Running `npm run release` in `packages/my-package` but the version in the root `package.json` bumps instead, or no commits are detected.

**Cause:** `standard-version` reads the nearest `package.json` and scans commits since the last git tag. If you run it from the wrong directory, or there is no prior tag, it reads all commits.

**Fix:**

- Always `cd packages/my-package` before running `npm run release`.
- Ensure a git tag exists for the previous version. For a fresh package, create an initial tag first:

  ```bash
  git tag v0.1.0
  ```

---

## 4) commitlint hook not running

**Problem:** Bad commit messages are not rejected.

**Checklist:**

1. Is `.husky/commit-msg` executable? Run `chmod +x .husky/commit-msg`.
2. Did you run `npx husky init`? Check that `.husky/` exists at the root.
3. Is `"prepare": "husky"` in the root `package.json` scripts?
4. Are you in the git repo root when committing? Hooks don't run outside the repo.

---

## 5) `npm run build --workspaces` fails on one workspace, stops everything

**Problem:** A failing build in one workspace aborts all others.

**Fix:** Use `--if-present` to skip workspaces without the script, and wrap in `||` or use `continue-on-error` in CI if you want partial failures to not block the pipeline:

```bash
npm run build --workspaces --if-present
```

In GitHub Actions, use `continue-on-error: true` on steps you expect to be flaky during development.

---

## 6) Dependency hoisting conflict

**Problem:** Two workspaces require different versions of the same package and one gets a version it didn't ask for.

**Fix:** Pin the conflicting package explicitly in the workspace that needs a specific version. npm workspaces will install the version listed in that workspace's `package.json` locally under `apps/foo/node_modules/` and hoist the other version to the root.

---

## 7) Accidentally publishing the wrong workspace

**Problem:** `npm publish` from the root or wrong directory publishes the root package (or nothing).

**Fix:** Always publish with an explicit `--workspace` flag:

```bash
npm publish --access public --workspace=packages/my-package
```

And keep the root `package.json` as `"private": true` to make it unpublishable.

---

## 8) `git push --follow-tags` not triggering the publish workflow

**Problem:** The GitHub Actions publish workflow doesn't run after `git push --follow-tags`.

**Checklist:**

1. The workflow trigger must be `on: push: tags: ["v*"]` — confirm it matches your tag prefix.
2. The tag must be pushed to the correct remote (`origin`).
3. Confirm the tag was created: `git tag -l`.
4. Check the Actions tab on GitHub for any trigger errors.
