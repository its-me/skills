---
name: dependabot
description: Configure and maintain Dependabot for GitHub Actions workflows, especially in repos that pin actions to major-version tags (e.g. @v4, @v7). Use this skill when asked to set up Dependabot, add a cooldown period, reduce Dependabot PR noise, merge or close open Dependabot PRs, or when Dependabot keeps opening PRs to bump an action's minor/patch version even though the workflow already tracks the major tag.
argument-hint: "[config|merge]"
---

## Instructions

Pick a mode from the argument:

- no argument — run the full workflow: edit the config (see "Configure") and then reconcile open PRs (see "Merge or close PRs").
- `config` — only edit `.github/dependabot.yaml`; do not touch open PRs.
- `merge` — only review and merge/close open Dependabot PRs; do not edit the config file.

## Configure

1. **Locate or create the config file** at `.github/dependabot.yaml` (or `.yml`) in the repository root.

2. **Baseline config** for a `github-actions` ecosystem entry. Always keep the blank line between `version:` and `updates:` — it's a stylistic convention, not a YAML requirement, but follow it anyway:

   ```yaml
   version: 2

   updates:
     - package-ecosystem: github-actions
       directory: /
       schedule:
         interval: weekly
   ```

   **Verify the blank line is actually there before moving on** — it's easy to drop when editing an existing file instead of writing fresh:
   ```bash
   grep -A1 '^version:' .github/dependabot.yaml
   ```
   The second line of output must be empty. If it isn't, edit the file to insert it — do not proceed to later steps with this missing.

3. **Add a cooldown period** so new releases aren't proposed the moment they're published (gives time for a bad release to surface before you'd pull it in):

   ```yaml
       cooldown:
         default-days: 7
   ```

4. **Diagnose "unnecessary bump" PRs.** If workflows reference actions by major-version tag only (e.g. `uses: docker/build-push-action@v7`, `uses: docker/login-action@v4`), that tag already floats forward automatically to pick up every minor/patch release at run time. Dependabot proposing to bump `@7` → `@7.2.0` or `@4` → `@4.2.0` is redundant churn — it doesn't add anything the floating tag doesn't already give you, and it locks the workflow to that specific minor version instead of continuing to float.

   Check what tag style a workflow actually uses before deciding this applies:
   ```bash
   grep -rn "uses: " .github/workflows/*.yaml
   ```
   If every `uses:` line pins to a bare major version (`@v4`, `@v7`, not `@v4.2.0` or a commit SHA), the fix below applies.

5. **Suppress minor/patch bump PRs** by adding an `ignore` rule to the same ecosystem entry:

   ```yaml
       ignore:
         - dependency-name: "*"
           update-types:
             - version-update:semver-minor
             - version-update:semver-patch
   ```

   This leaves major-version bumps (e.g. `v4` → `v5`) as the only PRs Dependabot opens — the ones that can actually contain breaking changes and deserve a human look.

## Merge or close PRs

1. **List open Dependabot PRs:**
   ```bash
   gh pr list --repo <owner>/<repo> --state open --search "author:app/dependabot"
   ```

2. **Close now-redundant PRs first.** If `.github/dependabot.yaml` has an `ignore` rule (from "Configure" or already in place), any open PR matching that update type will never be mergeable as-is — Dependabot typically auto-closes these on its next run, but don't wait on that:
   ```bash
   gh pr close <number> --repo <owner>/<repo> --comment "<reason>"
   ```

3. **Check CI before merging anything — never merge blind:**
   ```bash
   gh pr checks <number> --repo <owner>/<repo>
   ```
   - If it's a major-version bump (any ecosystem — these can contain breaking changes): leave it open and report it regardless of CI status. That's exactly the kind of PR this skill's `ignore` rule is designed to preserve for a human look — don't auto-merge it.
   - If it's a minor/patch bump on something that isn't redundant (e.g. an npm/pip dependency, or a github-actions entry with no `ignore` rule covering it) and checks pass: merge it.
     ```bash
     gh pr merge <number> --repo <owner>/<repo> --squash
     ```
   - If checks fail: leave it open and report why, regardless of bump type.

## Notes

- If a workflow pins actions to a commit SHA instead of a tag (common when a stricter security posture is wanted, e.g. via `unpinned-uses` in a zizmor policy — see the `zizmor` skill), the "redundant bump" reasoning does NOT apply: every version bump is meaningful there, since there's no floating tag doing the update for you. Don't add the `ignore` rule in that case.
- Do not disable Dependabot entirely or set an excessively long cooldown "to reduce noise" — the `ignore` rule targets the actual redundancy without losing visibility into major version bumps or other ecosystems (npm, pip, etc.) that don't have this floating-tag property.
