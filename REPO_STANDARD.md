# getnodus repo standard

This is the default shape for repositories in the `getnodus` org. Keep repo
automation quiet, advisory, and easy to understand.

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

Do not add dependency automation by default. Add Dependabot only when the repo
needs scheduled update PRs, and keep it quiet:

- monthly schedule
- grouped patch/minor updates
- one open PR at a time
- no automatic major version PRs

## Conductor lane

Repos that are used from Conductor should include a small `conductor.json`
with setup/run/archive scripts that work from an isolated workspace. Use
`CONDUCTOR_PORT` for dev servers instead of hard-coded ports. Add
`.worktreeinclude` only for safe local development files such as `.env.local`;
do not copy production secrets with broad `.env.*` patterns.

## Auto-merge

Two opt-in shared workflows exist for narrow auto-merge use:

- `auto-merge-on-green.yml` — PRs labeled `auto-merge` and authored by the
  org owner squash-merge when checks pass.
- `dependabot-auto-merge.yml` — patch/minor Dependabot PRs only.

Neither pushes fixes to human branches, deploys, or mutates workflow, auth,
billing, migration, secret, or security-sensitive paths. Bots and agents
otherwise comment, review, open PRs, and stop.
