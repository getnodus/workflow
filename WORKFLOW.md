# getnodus GitHub workflow control plane

This repository is the shared GitHub automation home for the `getnodus` org.
Keep it small. Repo-local workflows should call a reusable workflow from here
or defer to the official GitHub Apps — don't rebuild glue per repo.

## Current shared workflows

Each is `workflow_call`-only and lives in `.github/workflows/`.

- **`ci-node.yml`** — Reusable Node/TypeScript CI (typecheck, lint, test).
  Used by product repos. Advisory by default — does not block merges.
- **`auto-merge-on-green.yml`** — Enables GitHub's native auto-merge on PRs
  labeled `auto-merge`. Hard-gated to PRs authored by the org owner so a
  compromised collaborator cannot use it to fast-track changes.
- **`dependabot-auto-merge.yml`** — Approves and auto-merges Dependabot PRs
  for patch/minor bumps only. Major bumps stay manual (and `dependabot.yml`
  in each repo is already configured to ignore them — belt + suspenders).
- **`claude.yml`** — Lets trusted collaborators invoke Claude Code by writing
  `@claude` in an issue, PR, or review comment. Gated on `author_association`
  to OWNER / MEMBER / COLLABORATOR so internet drive-bys can't burn the org's
  Claude credits or trigger the action with our OAuth token.
- **`auto-triage.yml`** — Runs Claude Code on issues labeled `auto-triage` to
  investigate and (when confident) open a draft PR. **Treats issue body as
  untrusted input** — body and title are passed via env, not template
  expansion. Hard-gated to the org owner as the label applier. Callers must
  pass `CLAUDE_CODE_OAUTH_TOKEN` explicitly; `secrets: inherit` is forbidden
  because it would expose every org secret to a prompt-injectable surface.

## CI policy

PR CI is advisory by default. Fast checks (`typecheck`, `lint`, small
project-specific scripts) only. CI must not auto-fix, deploy, or block merges.
The auto-merge workflows above are explicit opt-ins via label, not blanket
policy.

Full production builds (e.g., Cloudflare) should be `workflow_dispatch` checks
unless a repo explicitly needs them on every PR.

## Agent integrations

Two paths exist and they are not the same:

1. **Org-level GitHub Apps** for Codex (`@codex review`, `@codex fix the CI
   failures`) and Claude (`@claude review`). These run with the App's own
   identity and are managed at the org level.
2. **The `claude.yml` / `auto-triage.yml` workflows in this repo.** These run
   with `CLAUDE_CODE_OAUTH_TOKEN` (org secret) and the calling repo's
   permissions. They are tighter to operate but easier to misconfigure — read
   the file headers before wiring them up in a new repo.

Manual triggers:

- `@codex review` — Codex GitHub App
- `@codex fix the CI failures` — Codex GitHub App
- `@claude review` — Claude GitHub App
- `@claude <anything>` (issue/PR/review comment) — this repo's `claude.yml`

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

- `CLAUDE_CODE_OAUTH_TOKEN` — required only by repos that call `claude.yml` or
  `auto-triage.yml`. Available as an org secret; pass explicitly from caller
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
- **Tight `--allowedTools` lists** on Claude Code invocations. Push is scoped
  to the `auto-triage/*` branch prefix; broad shell access is not granted.

## Repo onboarding checklist

1. Add a small advisory `ci.yml` (call `ci-node.yml` from this repo).
2. Install/enable the official Codex and Claude GitHub Apps at the org level.
3. Add `release.yml` only if the repo publishes versions.
4. Add `deploy.yml` only if the repo deploys production infrastructure.
5. Wire `claude.yml` / `auto-triage.yml` only if you actually want them; pass
   `CLAUDE_CODE_OAUTH_TOKEN` explicitly.
6. Remove repo-local custom AI review, cleanup, stale, dependency digest, and
   mixed-purpose bot workflows.
7. Avoid CODEOWNERS unless there is a real ownership boundary.
8. Add `conductor.json` when a repo should be easy to run from Conductor.
9. Add `.worktreeinclude` only for safe local development files; never broad
   production secret globs.
