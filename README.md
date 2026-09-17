# Amos CI

Central home for the shared CI/policy system used across `amos-*` repositories. Every `amos-*` repo runs the same set of checks on its pull requests by calling into this repo, instead of each repo maintaining its own copy.

## What Amos CI checks

Amos CI composes three concerns, each its own reusable GitHub Actions workflow:

| Workflow | File | What it does |
|---|---|---|
| **Workflow** | `.github/workflows/amos-ci-workflow.yml` | Enforces the Amos branching policy: validates branch naming (`feature/*`, `bugfix/*`, `hotfix/*`, `chore/*`) and PR direction (e.g. `feature/*` → `dev`, `dev` → `uat`, `uat` → `main`), checks branch freshness/promotion safety, labels PRs with the required merge method, posts a policy summary comment, and deletes merged topic/hotfix branches. |
| **Quality** | `.github/workflows/amos-ci-quality.yml` | Lint and test checks. Currently a no-op stub — language detection and real checks are not yet implemented. |
| **Security** | `.github/workflows/amos-ci-security.yml` | Security scanning. Currently a no-op stub. |

`.github/workflows/amos-ci.yml` composes all three into a single reusable workflow. It's the one file consuming repos actually call.

## How it fits together

```
amos-<repo>/.github/workflows/amos-ci.yml   (per-repo caller, copied from the sample below)
  → amos-ci/.github/workflows/amos-ci.yml   (this repo, composes the three below)
      → amos-ci-workflow.yml   (branch policy)
      → amos-ci-quality.yml    (lint/tests)
      → amos-ci-security.yml   (security scanning)
```

`amos-ci.yml` in this repo triggers on `pull_request` directly (so this repo's own PRs are checked too) and on `workflow_call` (so other repos can call it as a reusable workflow). Permissions are scoped per job — the Workflow job gets write access for labels/comments/branch cleanup, Quality and Security are read-only — regardless of how much a consuming repo grants at its own call site, since permissions can only narrow as they pass down a call chain, never widen.

## Adding Amos CI to a repo

1. Copy [`.github/workflows/amos-ci-repo.yml.sample`](.github/workflows/amos-ci-repo.yml.sample) into the target repo as `.github/workflows/amos-ci.yml`.
2. Confirm the `uses:` line still points at the correct `owner/amos-ci` — update it if the org changes.
3. Point the `uses:` line at `@main`. Every `amos-*` repo tracks `main` directly, so a change here takes effect everywhere on the next PR run — there's no per-repo version pinning.
4. Add any repo-specific jobs underneath the shared `amos-ci` job — it's a normal caller workflow, so local jobs run alongside the shared one without needing changes here.

### Branch ruleset

`.github/ruleset.json` is a template for the branch protection ruleset each repo should apply to `dev`/`uat`/`main` (deletion protection, non-fast-forward, required PR review settings, and a required status check for the Workflow policy job).