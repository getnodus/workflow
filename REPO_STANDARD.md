# getnodus repo standard

This is the default shape for repositories in the `getnodus` org. Keep repo
automation quiet, advisory, and easy to understand.

## README

Every repo opens with the same centered, public-facing header: the Nodus mark,
the repo name, a one-line bold tagline, and a row of `flat-square` badges with a
black label color. Ship light + dark mark variants (the white-circle mark
vanishes on GitHub's light theme) and swap them with `<picture>`.

Copy the mark assets from `getnodus/identity` (`marks/`) into the repo's
`assets/` so it renders without a cross-repo raw URL (required for private
repos), then use:

```html
<div align="center">

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="assets/nodus-mark-dark.svg">
  <img src="assets/nodus-mark-light.svg" alt="Nodus" width="120" height="120">
</picture>

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

PR checks should report useful signal without blocking the owner from merging.
Use one lightweight advisory check for normal PRs. CI is currently inlined
per-repo (there is no shared CI workflow — `ci-node.yml` was removed after
nobody adopted it). Match this shape:

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
    runs-on: ubuntu-latest
    permissions:
      contents: read
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '22'
          # cache: pnpm | npm | bun — match the repo's package manager
      - run: <install command>      # pnpm install --frozen-lockfile | npm ci | bun install --frozen-lockfile
      - run: <typecheck command>    # pnpm typecheck | npm run typecheck
```

Product repos that need a full Cloudflare production build should expose it as
a manual check, not an automatic PR check:

```yaml
on:
  workflow_dispatch:
    inputs:
      check:
        type: choice
        options: [typecheck, full-cloudflare]
        default: typecheck
```

Run the full check only when Fischer or an agent asks for it.

## Agent integrations

Two paths exist:

1. **Official GitHub Apps** for Codex and Claude, installed at the org level —
   handle the `review`-style triggers:
   - `@codex review` — Codex review
   - `@codex fix the CI failures` — Codex task work on a PR
   - `@claude review` — Claude review
2. **Workflows in `getnodus/workflow`** for richer Claude Code interactions:
   - `claude.yml` — repo-local copy lets trusted collaborators write
     `@claude <anything>` on issues/PRs. Hardened with an
     `author_association` allowlist; copy from `getnodus/workflow` rather
     than rolling your own.
   - `auto-triage.yml` — opt-in via label, opens draft PRs from issues.
     Treats issue bodies as untrusted; pass `CLAUDE_CODE_OAUTH_TOKEN`
     explicitly (never `secrets: inherit`).

Keep these manual. Do not enable automatic reviews by default.

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
`getnodus/context/.github/workflows/release.yml` for a working example).
There is no shared `release-please.yml` — it was removed after nothing
ever adopted it.

Most product repos that deploy from Cloudflare Builds do not need release
automation.

## Dependency lane

Dependency updates use Renovate via the shared preset. Add a `renovate.json`
that extends it:

```json
{ "extends": ["github>getnodus/workflow"] }
```

The preset (`default.json` in this repo) batches non-major npm updates into one
PR, groups GitHub Actions bumps, pins action digests, enforces a 3-day
stability window, and self-merges non-major updates once green. Major bumps
always stay manual. Do not add Dependabot — the two would fight over the same
lockfile.

## Conductor lane

Repos that are used from Conductor should include a small `conductor.json`
with setup/run/archive scripts that work from an isolated workspace. Use
`CONDUCTOR_PORT` for dev servers instead of hard-coded ports. Add
`.worktreeinclude` only for safe local development files such as `.env.local`;
do not copy production secrets with broad `.env.*` patterns.

## Auto-merge

There is no general "auto-merge on green" workflow. Dependency auto-merge is
narrow and bot-only:

- **Renovate** self-merges non-major dependency PRs once green and past the
  stability window (configured in the shared preset, not a workflow).
- **`pr-autofix.yml`** (opt-in, called from this repo) enables GitHub
  auto-merge on a Renovate/Dependabot PR it heals past the stability window.

Neither merges human-authored PRs, deploys, or mutates workflow, auth,
billing, migration, secret, or security-sensitive paths. Bots and agents
otherwise comment, review, open PRs, and stop.
