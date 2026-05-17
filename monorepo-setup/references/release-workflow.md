# Release Workflow (standard-version)

The release workflow lives inside the **package workspace** (`packages/my-package`), not at the root. Each package manages its own version and changelog independently.

## How it works

`standard-version` analyses git commits since the last tag, derives the SemVer bump from commit types, then:

1. Bumps the version in `package.json`
2. Appends a new section to `CHANGELOG.md`
3. Creates a release commit: `chore(release): vX.Y.Z`
4. Creates a git tag: `vX.Y.Z`

## Prerequisites

- `standard-version` installed as a `devDependency` in the package (see [npm-package.md](npm-package.md))
- `.versionrc.json` present in the package directory
- `CHANGELOG.md` initialised
- All changes committed with conventional commit messages

## Step-by-step release

### 1. Commit your changes

```bash
git add .
git commit -m "feat(my-package): add new utility function"
```

commitlint will validate the format before the commit lands.

### 2. Preview the release (dry run)

```bash
cd packages/my-package
npx standard-version --dry-run
```

Review the planned version bump and changelog entry without writing anything.

### 3. Create the release

```bash
# Let standard-version auto-detect the bump from commits
npm run release

# Or specify explicitly
npm run release:patch   # 0.1.0 → 0.1.1  (bug fixes)
npm run release:minor   # 0.1.0 → 0.2.0  (new features)
npm run release:major   # 0.1.0 → 1.0.0  (breaking changes)
```

### 4. Push and publish

```bash
# Push commits + tags together
git push origin main --follow-tags

# Publish to npm (or let CI do it — see cicd.md)
npm publish --access public --workspace=packages/my-package
```

## Version bump rules recap

`standard-version` derives the bump from commits since the **last git tag** (`v*`):

| Commits present | Resulting bump |
|---|---|
| At least one `feat!` or `BREAKING CHANGE` footer | MAJOR |
| At least one `feat` (no breaking) | MINOR |
| Only `fix`, `perf` | PATCH |
| Only `docs`, `refactor`, `test`, `chore` | No bump (no release) |

If no bump-worthy commits are found, `standard-version` exits with no changes.

## Undoing a release

Only undo a release that **has not been pushed or published**:

```bash
# Delete the local tag
git tag -d v0.2.0

# Undo the release commit (keeps changes staged)
git reset --soft HEAD~1

# Fix, amend, or retry
npm run release:minor
```

If the tag was already pushed:

```bash
git push origin :v0.2.0   # ⚠️ destructive — confirm before running
```

If the package was already published to npm, you cannot un-publish (for security reasons). Use `npm deprecate` instead:

```bash
npm deprecate @your-org/my-package@"0.2.0" "Released in error, use 0.2.1"
```

## First release (v0.1.0 → v1.0.0)

If this is the first official release and you want to skip `standard-version`'s bump logic:

```bash
npx standard-version --release-as 1.0.0
```

## Multiple packages releasing independently

Each package under `packages/` has its own git tag namespace. Tag format is `vX.Y.Z` by default. If multiple packages need independent tags, configure `standard-version` with a `tagPrefix`:

```json
{
  "tagPrefix": "my-package-v"
}
```

This produces tags like `my-package-v1.2.0` instead of `v1.2.0`, allowing `packages/foo` and `packages/bar` to release without interfering with each other.
