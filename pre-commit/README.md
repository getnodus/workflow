# Pre-commit hooks for Intuitumxyz/*

Shared [lefthook](https://github.com/evilmartians/lefthook) config that runs prettier + eslint on staged files before commit, and typecheck before push.

## Why

Catches lint/format/typecheck errors locally so they don't burn a `@claude` or `@codex` call when CI fails in a PR.

## Adopt in a repo

```sh
cd /path/to/repo
pnpm add -D lefthook
curl -o lefthook.yml https://raw.githubusercontent.com/Intuitumxyz/workflow/main/pre-commit/lefthook.yml
npx lefthook install
```

This installs `.git/hooks/pre-commit` and `.git/hooks/pre-push` that defer to `lefthook.yml`.

## Skip a hook (when you must)

```sh
LEFTHOOK=0 git commit -m "..."   # skip all hooks
git commit --no-verify ...      # same
```

Don't habit-skip; if a hook is wrong, fix the hook.
