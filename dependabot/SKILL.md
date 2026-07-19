---
name: dependabot
description: Configure and maintain Dependabot for GitHub Actions workflows, especially in repos that pin actions to major-version tags (e.g. @v4, @v7). Use this skill when asked to set up Dependabot, add a cooldown period, reduce Dependabot PR noise, or when Dependabot keeps opening PRs to bump an action's minor/patch version even though the workflow already tracks the major tag.
---

## Instructions

1. **Locate or create the config file** at `.github/dependabot.yaml` (or `.yml`) in the repository root.

2. **Baseline config** for a `github-actions` ecosystem entry:

   ```yaml
   version: 2

   updates:
     - package-ecosystem: github-actions
       directory: /
       schedule:
         interval: weekly
   ```

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

6. **After pushing the config change, close any now-redundant open PRs.** Dependabot typically auto-closes PRs that fall under a newly-added `ignore` rule on its next run, but verify:
   ```bash
   gh pr list --repo <owner>/<repo> --state open
   ```
   If any remain open and clearly match the ignored update type, close them manually:
   ```bash
   gh pr close <number> --repo <owner>/<repo> --comment "<reason>"
   ```

## Notes

- If a workflow pins actions to a commit SHA instead of a tag (common when a stricter security posture is wanted, e.g. via `unpinned-uses` in a zizmor policy — see the `zizmor` skill), the "redundant bump" reasoning does NOT apply: every version bump is meaningful there, since there's no floating tag doing the update for you. Don't add the `ignore` rule in that case.
- Do not disable Dependabot entirely or set an excessively long cooldown "to reduce noise" — the `ignore` rule targets the actual redundancy without losing visibility into major version bumps or other ecosystems (npm, pip, etc.) that don't have this floating-tag property.
