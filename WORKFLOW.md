# getnodus GitHub workflow control plane

This repository is the shared GitHub *automation* home for the `getnodus` org.
Org *identity* (profile, issue templates, community-health files) lives in
`getnodus/.github`, which GitHub renders specially. Two ideas keep this place
small:

- **Centralize the glue, not the work.** Cross-cutting agent automation that
  every repo wants *identically* (the `@claude` handler) lives here once as a
  reusable workflow; repos add a thin caller. Anything specific to a repo —
  above all its CI — is owned by that repo, not templated from here.
- **Advisory by default.** Checks and AI reviews inform; they don't block the
  owner from merging or mutate the repo on their own.

## What lives here

In `.github/workflows/`:

- **`claude.yml`** (`workflow_call` + direct) — the heavy `@claude` handler.
  Trusted collaborators invoke Claude Code by writing `@claude` in an issue, PR,
  or review comment. Gated on `author_association` (OWNER / MEMBER /
  COLLABORATOR) so internet drive-bys can't spend the org's Claude credit or
  trigger the action with our OAuth token. The heavy logic lives here once;
  other repos add a tiny caller
  (`uses: getnodus/workflow/.github/workflows/claude.yml@main`). The caller
  skeleton is in the file header.
- **`actionlint.yml`** (direct) — lints workflow files on PRs touching
  `.github/workflows/**`, so changes to the actions that power other repos have
  a real green signal.
- **`secret-scan.yml`** (direct) — gitleaks over this repo's history.

Also here: **`default.json`** (the shared Renovate preset, extended as
`github>getnodus/workflow`) and **`REPO_STANDARD.md`** (the default shape for a
getnodus repo).

## CI is bespoke per repo, not shared

There is no shared CI workflow, by design — one was tried (`ci-node.yml`) and
removed when it fit nobody. Each repo's CI is written for its own stack and
lives in that repo. What's constant is a set of principles, not a template:

- **Match the stack, derive versions from the repo.** npm / pnpm (corepack) /
  bun for Node; `swift build`/`xcodebuild` on macOS for Swift; whatever the repo
  actually uses. Read the toolchain version from `.nvmrc`, `engines`,
  `Package.swift` — never hardcode one the repo doesn't pin.
- **Fast and advisory.** Typecheck / lint / build or a small test set, using the
  repo's *real* scripts. Name it `CI`, add `concurrency` cancel-in-progress,
  `permissions: contents: read`, and `workflow_dispatch`. Don't gate merges on
  it by default.
- **Secret-scan every repo** with the pinned gitleaks *binary* (not
  `gitleaks-action`, which needs a paid license for org-owned repos), scoped to
  PRs + pushes to `main` + tags.
- **Full production builds** (e.g. Cloudflare) belong behind `workflow_dispatch`,
  not on every PR.

`REPO_STANDARD.md` has worked examples across stacks.

## Where CI runs: GitHub-hosted vs self-hosted

- **GitHub-hosted** (`ubuntu-latest`, `macos-latest`) is the default. It keeps
  working when the homelab is down, so small and critical repos stay here.
- **Self-hosted** (`runs-on: self-hosted`) is for heavy, trusted builds that are
  slow or costly on hosted runners — today, `getnodus/solo` (the VS Code fork).
  The fleet runs on `server` (i9-12900KF, 24 threads, NVMe); operate it with
  `/srv/infra/bin/ci-fleet` and read `/srv/infra/docs/ci-runners.md` on that box.
  **Trade-off:** anything self-hosted stops when that machine does, so move
  workloads there consciously, per repo — don't default the whole org onto it.

## Auto-merge

No general "auto-merge on green" workflow exists. Dependency auto-merge is
narrow and bot-only: **Renovate** self-merges non-major dependency PRs once
green and past the stability window — configured in the shared preset
(`default.json`), not a workflow. Major bumps stay manual. Human-authored PRs
are never auto-merged; the org owner decides when those land.

## Agent integrations

- **Org-level GitHub Apps** — Codex (`@codex review`, `@codex fix the CI
  failures`) and Claude (`@claude review`). Run with the App's own identity.
- **Cursor Bugbot** — a GitHub App (installed from the Cursor dashboard) that
  reviews every PR once; configured per repo via `.cursor/BUGBOT.md`.
- **`claude.yml` here** — the `@claude <anything>` handler for trusted
  collaborators, wired via a thin per-repo caller. Heavy logic and the
  `author_association` allowlist live here; don't roll your own.

## Required org secrets

- `CLAUDE_CODE_OAUTH_TOKEN` — used by repos that run or call `claude.yml`. Pass
  it explicitly from the caller, never via `secrets: inherit`.

Keep product/deploy secrets in the repos that need them. Don't add a standing
org service-account PAT unless a repo has a concrete need GitHub-native
automation can't cover.

## Security posture

- **No `secrets: inherit`** on any workflow touching user-supplied content —
  pass explicitly so the blast radius is bounded.
- **Untrusted input never expands into prompts or shell.** Issue/PR titles and
  bodies go through `env:` and are referenced as `$VAR`.
- **Author allowlists / `author_association` checks** on anything that spends
  org credit or holds write permissions — defense-in-depth over the action's
  own checks.
- **Tight tool/permission scopes** on agent invocations; writes scoped to safe
  branches, never broad shell on `main`.

## Onboarding a repo

1. A bespoke advisory `ci.yml` for the repo's stack (see `REPO_STANDARD.md`).
2. `secret-scan.yml` (gitleaks binary).
3. `renovate.json`: `{ "extends": ["github>getnodus/workflow"] }`.
4. `claude.yml` thin caller — pass `CLAUDE_CODE_OAUTH_TOKEN` explicitly.
5. `.cursor/BUGBOT.md` with project-specific review rules.
6. Enable the org Codex/Claude Apps.
7. `release.yml` / `deploy.yml` only if the repo publishes or deploys.
8. No CODEOWNERS unless there's a real ownership boundary.
