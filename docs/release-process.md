# Release Process

This document describes how releases are produced for the Konveyor editor
extensions, including VS Code Marketplace and Open VSX, and what each step
relies on.

## Release Channels

- Stable releases: even minor versions (e.g., 0.4.x, 0.6.x)
- Pre-releases: odd minor versions (e.g., 0.5.x) published with pre-release flag

The odd/even cadence exists because the VS Code Marketplace does not allow
semantic pre-release identifiers and expects distinct versions for pre-releases.

## Primary Workflows

All automated releases flow through `./.github/workflows/release.yml`, which:

- Builds and tests
- Packages VSIX artifacts
- Runs E2E tests
- Publishes to marketplaces
- Creates tag + GitHub Release

Other workflows call it with different inputs.

### Stable Release (cut a new minor series)

Workflow: `./.github/workflows/cut-release.yml`

Use this to create a new stable release series from `main`.

Steps:

1. Run the workflow (Actions -> Cut Release -> Run workflow).
2. It validates the current `package.json` version:
   - Must be even minor (stable)
   - Tag and release branch must not already exist
3. It calls `release.yml` with:
   - `ref: main`
   - `prerelease: false`
4. Post-release housekeeping:
   - Assembles changelog on `main`
   - Bumps `main` to the next even minor version
   - Creates `release-X.Y` branch from the tag
   - Assembles changelog on the release branch

### Patch Release (stable patch on a release branch)

Workflow: `./.github/workflows/patch-release.yml`

Use this for 0.X.Z patch releases on an existing `release-X.Y` branch.

Steps:

1. Run the workflow with `branch=release-X.Y`.
2. It computes the next patch version based on tags.
3. If there are no new commits since the previous tag, it skips.
4. Otherwise, it calls `release.yml` with:
   - `ref: release-X.Y`
   - `prerelease: false`
5. It assembles the changelog on the release branch.

### Scheduled Patch Releases (daily)

Workflow: `./.github/workflows/scheduled-patch-releases.yml`

This dispatches `patch-release.yml` for all `release-*` branches daily, skipping
very early branches excluded in the script.

### Scheduled Pre-releases (daily)

Workflow: `./.github/workflows/scheduled-prerelease.yml`

This runs daily and:

1. Computes the next odd-minor prerelease version from `main`.
2. Skips if there are no new commits since the last prerelease tag.
3. Calls `release.yml` with:
   - `ref: main`
   - `prerelease: true`

## Marketplace Publishing

Publishing is performed by `release.yml`:

- VS Code Marketplace: `npx @vscode/vsce publish`
- Open VSX: `npx ovsx publish`

Each VSIX in `dist/*.vsix` is published to both registries unless skipped.

You can skip either publish step with inputs:

- `skip_vscode_publish: true`
- `skip_openvsx_publish: true`

## Manual Open VSX Release (script)

Script: `./scripts/release-openvsx.js`

This script is a manual alternative for Open VSX releases. It:

- Bumps versions across workspace packages
- Builds, packages, and optionally publishes
- Optionally commits and tags

Example:

```
node scripts/release-openvsx.js patch
```

This script does NOT publish to the VS Code Marketplace.

## What Everything Relies On

### Versioning & Tags

- Tags are `vX.Y.Z` and are required for release artifacts and GitHub releases.
- Stable releases require even minor versions.
- Pre-releases use odd minor versions and publish with `--pre-release`.

### Build & Packaging

The release pipeline relies on:

- `npm run build` to compile all packages
- `npm run dist` to assemble distributable output
- `npm run package` to generate `dist/*.vsix`
- `npm run collect-assets` (or `--use-workflow-artifacts`) to fetch runtime assets

### GitHub Actions Secrets / Vars

The workflows rely on these secrets or variables:

- `VSCODE_MARKETPLACE_TOKEN` for VS Code Marketplace publishing
- `OVSX_MARKETPLACE_TOKEN` for Open VSX publishing
- `OPENAI_API_KEY` for release E2E tests
- `KONVEYOR_BOT_ID` and `KONVEYOR_BOT_KEY` for release tagging and pushes

### Release Artifacts

`release.yml` packages and uploads VSIX artifacts and uses them for:

- E2E testing
- Publishing to both marketplaces
- GitHub Release attachments

### Changelog Fragments

User-facing changes must include a changelog fragment in `changes/unreleased`.
Releases assemble these fragments into per-extension `CHANGELOG.md` files.

### Branches

- `main` is the source of prereleases.
- `release-X.Y` branches are the source of stable patch releases.

## Dry Runs

The following workflows support dry runs that do not publish or modify:

- `cut-release.yml` (input `dry_run`)
- `patch-release.yml` (input `dry_run`)
- `scheduled-prerelease.yml` (input `dry_run`)
- `scheduled-patch-releases.yml` (input `dry_run`)

## Troubleshooting

- If `release.yml` fails in publish steps, re-run with the relevant `skip_*`
  input and publish manually once fixed.
- If version validation fails for a stable release, verify `package.json`
  uses an even minor number.
- If prerelease or patch release skips, confirm there are commits since the
  previous tag on the target branch.
