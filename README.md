<div align="center">

<img src="assets/nodus-mark.svg" alt="Nodus" width="100" height="100">

# workflow

**The shared GitHub automation control plane for the [`Intuitumxyz`](https://github.com/Intuitumxyz) org.**
<br>
Reusable workflows, the Renovate preset, and the repo standard — in one place, so the glue isn't rebuilt per repo.

<br>

[![reusable workflows](https://img.shields.io/badge/reusable-workflows-000000?style=flat-square&labelColor=000000)](.github/workflows)
[![renovate preset](https://img.shields.io/badge/renovate-shared_preset-000000?style=flat-square&labelColor=000000)](default.json)
[![license](https://img.shields.io/badge/license-Apache--2.0-000000?style=flat-square&labelColor=000000)](LICENSE)

</div>

---

This repo holds the reusable workflows, automation policy, and shared config that
other `Intuitumxyz` repos call or extend — kept in one place so the glue isn't
rebuilt per repo. Org *identity* (profile, issue templates, community health
files) stays in [`Intuitumxyz/.github`](https://github.com/Intuitumxyz/.github), which
GitHub renders specially; *automation* lives here.

## What's here

| Path | What it is |
|---|---|
| `.github/workflows/actionlint.yml` | Lints workflow files in this repo. |
| `WORKFLOW.md` | Automation control-plane policy and security posture. |
| `REPO_STANDARD.md` | Default shape for `Intuitumxyz` repos. |
| `default.json` | Shared Renovate preset. Repos extend it via `github>Intuitumxyz/workflow`. |
| `pre-commit/lefthook.yml` | Shared lefthook hooks (prettier + eslint + typecheck). |

## Using it from another repo

**Renovate** — in `renovate.json`:

```json
{ "extends": ["github>Intuitumxyz/workflow"] }
```

**Pre-commit hooks** — see [`pre-commit/README.md`](pre-commit/README.md).

Read the header of each workflow file before wiring it up — they document the
security gating (`author_association` allowlists, no `secrets: inherit`,
untrusted-input handling).
