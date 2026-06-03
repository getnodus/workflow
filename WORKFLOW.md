# getnodus GitHub workflow control plane

This repository is the shared GitHub automation home for the `getnodus` org.
Keep it small. Repo-local workflows should call a reusable workflow from here
or defer to the official GitHub Apps — don't rebuild glue per repo. Org
*identity* (profile, issue templates, community-health files) lives in
`getnodus/.github`, which GitHub renders specially; *automation* lives here.

## Current shared workflows

All live in `.github/workflows/`. Three are reusable (`workflow_call`); only
`actionlint.yml` runs directly here.

- **`auto-triage.yml`** (`workflow_call`) — Runs Claude Code on issues labeled
  `auto-triage` to investigate and (when confident) open a draft PR. **Treats
  the issue body and title as untrusted input** — passed via `env`, never
  template-expanded. Callers must pass `CLAUDE_CODE_OAUTH_TOKEN` explicitly;
  `secrets: inherit` is forbidden because it would expose every org secret to a
  prompt-injectable surface.
- **`pr-autofix.yml`** (`workflow_call`) — When a PR is failing (CI red and/or
  merge-conflicted) runs Claude Code to get it back to a healthy, mergeable
  state, pushing to the PR's own head branch — never `main`. Two caller entry
  points: `workflow_run` on a red CI run, or a `schedule` sweep of conflicting
  PRs. Nothing is merged except a Renovate/Dependabot PR it heals that has
  cleared the stability window, where it enables GitHub auto-merge. Hard-gated
  to non-fork PRs from allowlisted authors (dependency bots + org humans).
- **`claude.yml`** (`workflow_call` + direct) — The heavy `@claude` handler.
  Trusted collaborators invoke Claude Code by writing `@claude` in an issue,
  PR, or review comment. Gated on `author_association` (OWNER / MEMBER /
  COLLABORATOR) so internet drive-bys can't burn the org's Claude credits or
  trigger the action with our OAuth token. The heavy logic lives here once;
  other repos add a tiny caller (`uses: getnodus/workflow/.github/workflows/claude.yml@main`)
  with the event triggers. It also self-serves `@claude` on this repo via its
  own direct triggers. The caller skeleton is in the file header.
- **`actionlint.yml`** (direct trigger) — Lints workflow files on PRs that
  touch `.github/workflows/**` so changes to the actions that power other repos
  have a real green signal. Mark it required in branch protection to gate
  auto-merge on it.

## CI policy

PR CI is advisory by default and inlined per repo — there is no shared CI
workflow (`ci-node.yml` was removed after nobody adopted it). Fast checks
(`typecheck`, `lint`, small project-specific scripts) only. CI must not
auto-fix, deploy, or block merges. See `REPO_STANDARD.md` for the canonical
advisory `ci.yml` shape.

Full production builds (e.g., Cloudflare) should be `workflow_dispatch` checks
unless a repo explicitly needs them on every PR.

## Auto-merge

There is no general "auto-merge on green" workflow. Two narrow paths exist:

- **Renovate self-merges** non-major dependency PRs once checks are green and
  the 3-day stability window clears. This is configured in the shared Renovate
  preset (`default.json`), not a workflow. Major bumps always stay manual.
- **`pr-autofix.yml`** enables GitHub auto-merge on a Renovate/Dependabot PR it
  has healed past the stability window — a no-op unless the base branch has a
  required status check (otherwise auto-merge would merge with no CI gate).

Neither path merges human-authored PRs. The org owner decides when those land.

## Agent integrations

Two paths exist and they are not the same:

1. **Org-level GitHub Apps** for Codex (`@codex review`, `@codex fix the CI
   failures`) and Claude (`@claude review`). These run with the App's own
   identity and are managed at the org level.
2. **The `claude.yml`, `auto-triage.yml`, and `pr-autofix.yml` workflows in
   this repo.** These run with `CLAUDE_CODE_OAUTH_TOKEN` (org secret) and the
   calling repo's permissions. They are tighter to operate but easier to
   misconfigure — read the file headers before wiring them up in a new repo.

Manual triggers:

- `@codex review` — Codex GitHub App
- `@codex fix the CI failures` — Codex GitHub App
- `@claude review` — Claude GitHub App
- `@claude <anything>` (issue/PR/review comment) — a repo's own `claude.yml`

Do not enable automatic agent reviews by default.

## Org ruleset policy

The org ruleset keeps `main` protected from deletion and force pushes. Default
branch protection does not require:

- approving reviews
- CODEOWNERS review
- status checks
- strict up-to-date branches

Checks and AI reviews are advisory. The org owner decides when to merge.

## Required org secrets

- `CLAUDE_CODE_OAUTH_TOKEN` — required by repos that run `claude.yml` or call
  `auto-triage.yml` / `pr-autofix.yml`. Available as an org secret; make sure
  it is accessible to those repos, and pass it explicitly from caller
  workflows, never via `secrets: inherit`.

Keep product/deploy secrets in the repos that need them. Release automation
uses the default GitHub Actions token; do not add a standing org service-
account PAT unless a repo has a concrete release requirement that cannot use
GitHub-native automation.

## Security posture

- **No `secrets: inherit`** in any caller workflow that touches user-supplied
  content. Pass secrets explicitly so the blast radius is bounded.
- **No template expansion of issue/PR bodies into prompts or shell.** Pass via
  `env:` and reference as `$VAR`. Issue/PR titles and bodies are untrusted.
- **Author allowlists / `author_association` checks** on every workflow that
  spends org credit or has write permissions. The action's internal checks
  are defense; the workflow `if:` is defense-in-depth.
- **Tight `--allowedTools` lists** on Claude Code invocations, with writes
  scoped to safe branches — `auto-triage.yml` pushes only to `auto-triage/*`
  branches; `pr-autofix.yml` pushes only to the triggering PR's own head ref,
  never `main`. Broad shell access is not granted.

## Repo onboarding checklist

1. Add a small advisory `ci.yml` (inline it per `REPO_STANDARD.md`).
2. Install/enable the official Codex and Claude GitHub Apps at the org level.
3. Add `release.yml` only if the repo publishes versions.
4. Add `deploy.yml` only if the repo deploys production infrastructure.
5. Wire `claude.yml`, `auto-triage.yml`, or `pr-autofix.yml` only if you
   actually want them — add a tiny caller (`uses: getnodus/workflow/...`) and
   pass `CLAUDE_CODE_OAUTH_TOKEN` explicitly.
6. Extend the shared Renovate preset: `{ "extends": ["github>getnodus/workflow"] }`.
7. Remove repo-local custom AI review, cleanup, stale, dependency digest, and
   mixed-purpose bot workflows.
8. Avoid CODEOWNERS unless there is a real ownership boundary.
9. Add `conductor.json` when a repo should be easy to run from Conductor.
10. Add `.worktreeinclude` only for safe local development files; never broad
    production secret globs.
