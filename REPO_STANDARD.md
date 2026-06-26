# Intuitum repo standard

The default shape for repositories in the `Intuitum` org. Keep repo automation
quiet, advisory, and easy to understand. Treat this as principles to apply, not
a recipe to paste — every repo differs, so match the repo, not the template.

## README

Every repo opens with the same centered, public-facing header: the Nodus mark,
the repo name, a one-line bold tagline, and a row of `flat-square` badges with a
black label color. Use the single white-circle mark at `100x100` — there is no
black-background variant.

Copy the mark (`nodus-mark.svg`) from `Intuitum/identity` (`marks/`) into the
repo's `assets/` so it renders without a cross-repo raw URL (required for
private repos), then use:

```html
<div align="center">

<img src="assets/nodus-mark.svg" alt="Nodus" width="100" height="100">

# repo-name

**One-line bold tagline.**
<br>
A sentence of context under it.

<br>

[![badge](https://img.shields.io/badge/label-value-000000?style=flat-square&labelColor=000000)](#)

</div>

---
```

Badge style: `style=flat-square&labelColor=000000`. Keep values black/white;
use a single Thermal Scope accent (e.g. `DB2F61`) for at most one badge — one
thermal moment per surface, never decoration. Named "character" repos (e.g.
agents like Leo) keep their own avatar instead of the shared mark.

## Pull request CI

CI is **bespoke per repo** — written for that repo's stack, living in that repo.
There is no shared CI workflow (`ci-node.yml` was tried and removed when it fit
nobody). What's constant is the shape, not the commands:

- **Advisory, not blocking.** Report useful signal; don't gate the owner's merge.
- **Named `CI`**, with `concurrency` cancel-in-progress, `permissions:
  contents: read`, a `workflow_dispatch` entry, and PR triggers on `main`. The
  `CI` name matters — other automation keys off it.
- **Match the stack and derive versions from the repo** — never hardcode a
  toolchain the repo doesn't pin (`.nvmrc`, `engines`, `Package.swift`).
- **Run only what exists.** Use the repo's real scripts (`typecheck`, `lint`,
  `build`, `test`); don't invent script names. One advisory check is fine for a
  small repo; a matrix or several jobs is fine for a larger one.

Worked shapes — adapt, don't copy blindly:

```yaml
# Node + npm
- uses: actions/setup-node@v6
  with: { node-version-file: .nvmrc, cache: npm }   # or node-version: '24'
- run: npm ci
- run: npm run typecheck            # + lint / build / test if they exist
```
```yaml
# Node + pnpm
- uses: pnpm/action-setup@v4        # or `corepack enable`
- uses: actions/setup-node@v6
  with: { node-version-file: .nvmrc, cache: pnpm }
- run: pnpm install --frozen-lockfile
- run: pnpm typecheck
```
```yaml
# Swift (SwiftPM)
runs-on: macos-latest               # xcodebuild instead if there's an .xcodeproj
- run: swift build
- run: swift test
```

Full skeleton for a typical Node repo:

```yaml
name: CI

on:
  pull_request:
    branches: [main]
    types: [opened, synchronize, reopened, ready_for_review]
  workflow_dispatch:

concurrency:
  group: ci-${{ github.repository }}-${{ github.event.pull_request.number || github.run_id }}
  cancel-in-progress: true

jobs:
  typecheck:
    name: Typecheck (advisory)
    runs-on: ubuntu-latest          # self-hosted for heavy/trusted builds — see WORKFLOW.md
    permissions:
      contents: read
    steps:
      - uses: actions/checkout@v6
      - uses: actions/setup-node@v6
        with:
          node-version-file: .nvmrc
          cache: npm                # pnpm | npm | bun — match the repo
      - run: npm ci                 # the repo's real install command
      - run: npm run typecheck      # the repo's real check(s)
```

Heavy builds (large monorepos, the VS Code fork) can use `runs-on: self-hosted`;
see WORKFLOW.md for the GitHub-hosted-vs-self-hosted trade-off and the
`server` fleet. Full production builds (e.g. Cloudflare) go behind
`workflow_dispatch`, not every PR.

## Secret scanning

Every repo carries a `secret-scan.yml` running the pinned **gitleaks binary**
(not `gitleaks-action` — it needs a paid license for org-owned repos), scoped to
PRs + pushes to `main` + tags, with `fetch-depth: 0` and `--redact --exit-code 1`.

## Agent integrations

Two paths:

1. **Official GitHub Apps** for Codex, installed at the org level —
   `@codex review`, `@codex fix the CI failures`.
2. **Cursor Bugbot** — installed via the Cursor dashboard as a GitHub App.
   Reviews PRs automatically (Once Per PR mode). Customized per-repo via
   `.cursor/BUGBOT.md` files. No workflow trigger needed.

## Bugbot configuration

Every repo should have a `.cursor/BUGBOT.md` at the root with project-specific
review rules. Bugbot always includes the root file and traverses upward from
changed files to find nested `BUGBOT.md` files for directory-specific context.

```
project/
  .cursor/BUGBOT.md          # Always included (project-wide rules)
  backend/
    .cursor/BUGBOT.md        # Included when reviewing backend files
```

## Branch protection

Org rulesets should protect `main` from force pushes and deletion. They should
not require approving reviews, CODEOWNERS review, strict up-to-date branches, or
required status checks by default.

## CODEOWNERS

Do not add CODEOWNERS unless a repo has a real ownership boundary. Default
CODEOWNERS files create review-request noise and should stay out of normal
product repos.

## Release lane

Repos that publish versions inline their own `release.yml` (see
`Intuitum/context/.github/workflows/release.yml` for a working example).
There is no shared `release-please.yml` — it was removed after nothing
ever adopted it.

Most product repos that deploy from Cloudflare Builds do not need release
automation.

## Dependency lane

Dependency updates use Renovate via the shared preset. Add a `renovate.json`
that extends it:

```json
{ "extends": ["github>Intuitum/workflow"] }
```

The preset (`default.json` in this repo) batches non-major npm updates into one
PR, groups GitHub Actions bumps, pins action digests, enforces a stability
window, and self-merges non-major updates once green. Major bumps always stay
manual. Do not add Dependabot — the two would fight over the same lockfile.

## Conductor lane

Repos that are used from Conductor should include a small `conductor.json`
with setup/run/archive scripts that work from an isolated workspace. Use
`CONDUCTOR_PORT` for dev servers instead of hard-coded ports. Add
`.worktreeinclude` only for safe local development files such as `.env.local`;
do not copy production secrets with broad `.env.*` patterns.

## Auto-merge

There is no general "auto-merge on green" workflow. Dependency auto-merge is
narrow and bot-only: **Renovate** self-merges non-major dependency PRs once
green and past the stability window (configured in the shared preset, not a
workflow). Major bumps stay manual.

Human-authored PRs are never auto-merged. Bots and agents otherwise comment,
review, open PRs, and stop — the org owner decides when changes land.
