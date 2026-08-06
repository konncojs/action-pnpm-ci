# `@konncojs/action-pnpm-ci`

> An internal, opinionated GitHub composite action for running the standard CI workflow (checkout, install, lint, typecheck, test, build) across organization libraries and repositories.

## Overview

`@konncojs/action-pnpm-ci` is a **composite GitHub Action** designed for internal reuse. It codifies the common CI workflow used by our libraries into a single reusable action, so every repository and package gets the same checkout, dependency-install, lint, typecheck, test, and build pipeline.

The action is published to the GitHub Packages registry and released from this repository using **Nx Release** with file-based version plans. Each release also updates a rolling major-version tag (`v1`, `v2`, …) so consumers can pin to `@v1` for automatic patch/minor updates.

## Features

- ✅ **Composite action** — runs as native GitHub Action steps (no Docker overhead).
- 📦 **pnpm-first** — uses `pnpm/action-setup` with `pnpm install --frozen-lockfile`.
- 🔧 **Node version from file** — respects `.node-version` (or any file you point to).
- 🔑 **GitHub Packages .npmrc** — optionally configures scoped registry auth for internal `@konncojs/*` packages.
- 🧑‍💻 **Git user setup** — configures `github-actions[bot]` as the git identity.
- 🧪 **Standard lifecycle** — runs `lint`, `typecheck`, `test`, and optional `build` scripts.
- 🚀 **Nx Release managed** — versioning, changelogs, GitHub releases, and rolling major tags.

## Installation / Usage

> **Note:** This action publishes to the GitHub Packages registry. To consume it, your consumer workflows must authenticate with a `GITHUB_TOKEN` that has `packages: read` (and `contents: read` for the action itself if the repo is private).

Pin to a rolling major tag:

```yaml
jobs:
  ci:
    runs-on: ubuntu-latest
    steps:
      - uses: konncojs/action-pnpm-ci@v1
        with:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

Or pin to an exact version:

```yaml
- uses: konncojs/action-pnpm-ci@v1.0.0
  with:
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

### Full workflow example

```yaml
# .github/workflows/ci.yaml
name: CI

on:
  pull_request:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: read
    steps:
      - uses: konncojs/action-pnpm-ci@v1
        with:
          node-version-file: .node-version
          configure-npmrc: "true"
          run-build: "true"
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

## Inputs

| Input               | Description                                                           | Required | Default         |
| ------------------- | --------------------------------------------------------------------- | -------- | --------------- |
| `node-version-file` | Path to the file containing the Node.js version to use.               | No       | `.node-version` |
| `configure-npmrc`   | Whether to configure `.npmrc` for GitHub Packages authentication.     | No       | `true`          |
| `run-build`         | Whether to run the `build` step. Set to `false` to skip builds.       | No       | `true`          |
| `GITHUB_TOKEN`      | GitHub token for authentication (used for registry auth in `.npmrc`). | **Yes**  | —               |

### Input details

#### `node-version-file`

Passed directly to `actions/setup-node`. The default value is `.node-version`, which this repository also defines. The file must contain a Node.js version understood by `setup-node` (e.g. `24.19.0`, `lts/*`, etc.).

#### `configure-npmrc`

When `true`, the action runs:

```bash
pnpm config set "//npm.pkg.github.com/:_authToken" "${GITHUB_TOKEN}" --location project
pnpm config set "@${GITHUB_REPOSITORY_OWNER}:registry" "https://npm.pkg.github.com" --location project
```

This allows `pnpm install` to fetch internal scoped packages (`@konncojs/*`) from the GitHub Packages npm registry. If the consumer repository does not consume private packages, set this to `false`.

#### `run-build`

When `true`, the action runs `pnpm build`. Set to `false` for repositories that only need typecheck/test or have a custom build workflow.

#### `GITHUB_TOKEN`

Must be a GitHub token with at least:

- `packages: read` to install private GitHub Packages npm dependencies.
- `contents: read` to fetch the action (if the action repository is private).

## Who is this for?

This action is built for internal use. It is meant to be reused across:

- Organization monorepo libraries
- npm packages published under `@konncojs/*`
- Any repository inside the `konncojs` GitHub organization that follows the same pnpm/Nx conventions

If you use this action outside the organization, fork it and adapt the GitHub Packages registry configuration to match your own scope.

## What the action does

1. **Checkout code** — full-depth checkout (`fetch-depth: 0`) so tools like Nx release or conventional changelog can inspect git history.
2. **Set Git user** — `github-actions[bot]`.
3. **Setup Node.js** — version resolved from `node-version-file`.
4. **Setup pnpm** — installs and caches pnpm.
5. **Configure `.npmrc`** — (optional) sets GitHub Packages auth and scoped registry.
6. **Install dependencies** — `pnpm install --frozen-lockfile`.
7. **Run linter** — `pnpm lint`.
8. **Run typecheck** — `pnpm typecheck`.
9. **Run tests** — `pnpm test`.
10. **Build** — `pnpm build` (unless `run-build` is `false`).

### Required scripts in the consumer repo

The consumer project **must** define the following scripts in `package.json`:

```json
{
  "scripts": {
    "lint": "...",
    "typecheck": "...",
    "test": "...",
    "build": "..."
  }
}
```

## Repository layout

```text
.
├── action.yaml                  # Composite action manifest
├── package.json                 # Package metadata (Nx + pnpm scripts)
├── pnpm-workspace.yaml          # pnpm workspace settings
├── nx.json                      # Nx workspace + release configuration
├── project.json                 # Nx project metadata for this package
├── .node-version                # Node version pinned for CI
├── .npmrc                       # GitHub Packages registry skeleton
├── .github/
│   └── workflows/
│       └── release.yaml         # Nx Release workflow
└── .nx/version-plans/           # Version-plan files for Nx Release
```

## Release workflow

Releases are driven by the `🚀 Release Action` workflow (`.github/workflows/release.yaml`). It is triggered manually via `workflow_dispatch` and supports a `dryRun` option.

The workflow:

1. Checks out the repo with full history.
2. Sets the `github-actions[bot]` git user.
3. Sets up Node.js (from `.node-version`) and pnpm.
4. Installs dependencies (`pnpm install --frozen-lockfile`).
5. Runs `pnpm nx release --skip-publish` to:
   - Resolve version bumps from `.nx/version-plans/*.md` files.
   - Update `package.json` versions.
   - Generate project and workspace changelogs.
   - Commit, tag, and push the release.
   - Create a GitHub Release from the workspace changelog.
6. Updates the rolling major tag (`vX`) so consumers using `@v1` automatically pick up the latest compatible release.

> `pnpm nx release --skip-publish` is used because this repository releases a **GitHub Action**, not an npm package. The `skip-publish` flag prevents Nx from attempting an npm registry publish while still performing versioning, changelog generation, git operations, and GitHub release creation.

### Version plans

This project uses Nx Release **file-based version plans**. When you want to release, create a markdown file under `.nx/version-plans/` (the easiest way is `pnpm plan`):

```bash
pnpm plan
```

Each plan declares the intended semver bump per project, for example:

```markdown
---
__default__: major
---

Init `action.yaml`
```

`__default__` is the shorthand used for single-project workspaces (or when the change applies to every project). During `nx release`, Nx reads these files and computes the next version.

### Rolling major tags

After a release, the workflow force-updates a major-version tag:

```bash
VERSION=$(pnpm pkg get version)      # e.g. "1.2.3"
MAJOR="v$(echo ${VERSION} | cut -d. -f1)"  # => v1
git tag -fa "$MAJOR" -m "Update $MAJOR to v${VERSION}"
git push origin "$MAJOR" --force
```

This lets consumers depend on `konncojs/action-pnpm-ci@v1` and receive compatible updates automatically.

## Authentication & permissions

### In the action itself

The action requires a `GITHUB_TOKEN` input because it writes the token into the local `.npmrc` so pnpm can read private GitHub Packages. The token provided to the action only needs `packages: read` unless your consumer repo itself performs write operations.

### In `.github/workflows/release.yaml`

That workflow is granted:

```yaml
permissions:
  contents: write
```

`contents: write` is required to:

- Push release commits and tags.
- Create GitHub Releases.
- Force-push the rolling major tag.

## Development

### Prerequisites

- Node.js version specified in `.node-version`. Using a Node version manager? Run:
  - `nvm use` or
  - `fnm use`
- pnpm `11.20.0` (enforced via `packageManager` field; corepack will install it automatically).

### Install dependencies

```bash
pnpm install --frozen-lockfile
```

### Create a version plan

```bash
pnpm plan
# or the full Nx command
npx nx release plan
```

Fill in the generated file under `.nx/version-plans/`, commit it, and push to `main`. The release process is triggered manually through GitHub Actions.

### Simulate a release

From a clean working tree on the latest `main`:

```bash
npx nx release --dry-run
```

or via the `workflow_dispatch` `dryRun` input in GitHub Actions.

### Build settings

`pnpm-workspace.yaml` opts into two pnpm features:

```yaml
allowBuilds:
  nx: false

enableGlobalVirtualStore: true
```

- `allowBuilds.nx: false` — prevents running nx postinstall build scripts.
- `enableGlobalVirtualStore: true` — uses pnpm’s global virtual store for faster, shared dependency installs.

## Why a composite action instead of a reusable workflow?

A **composite action** is run in the caller’s job context, so it has direct access to the caller’s default working directory, environment files, `GITHUB_TOKEN`, and job matrix. This makes it ideal for a “standard CI pipeline” that should feel like a single step to consumers, while still running exactly the same commands in their own repository.

## Limitations

- The consumer repository must expose `lint`, `typecheck`, `test`, and (optionally) `build` scripts in `package.json`.
- The action always runs Linux/bash shell steps; use it on `ubuntu-latest` runners for best compatibility.
- GitHub Packages authentication is scoped to the repository owner (`@${GITHUB_REPOSITORY_OWNER}`).

## License

[ISC](./package.json)

---

Maintained for internal use across organization libraries and repositories.
