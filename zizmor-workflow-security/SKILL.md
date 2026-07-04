---
name: zizmor-workflow-security
description: Investigate and fix GitHub Actions security findings from zizmor (via GitHub code scanning alerts or the zizmor CLI directly), and configure zizmor's own linting workflow and its inline rule config. Use this skill when asked to fix a code-scanning alert produced by zizmor, when a repository's Actions workflows need a security review, or when tuning which zizmor rules run/are suppressed.
---

## Instructions

### 1. Fetch the exact finding before guessing at a fix

Given a code-scanning alert URL like `https://github.com/<owner>/<repo>/security/code-scanning/<N>`:

```bash
gh api repos/<owner>/<repo>/code-scanning/alerts/<N>
```

This returns the rule id (`rule.id`, e.g. `zizmor/template-injection`), severity, and the exact `most_recent_instance.location` (`path`, `start_line`, `end_line`) of the offending code. Always read the flagged file at that location before editing — don't pattern-match from memory.

### 2. Run zizmor locally to see the full picture and verify fixes

```bash
zizmor .github/workflows/*.yaml
```

Use JSON output to precisely enumerate all instances of one rule across every file (the plain-text output's `-->` lines are easy to misattribute across findings when there are many):

```bash
zizmor --format json .github/workflows/*.yaml 2>/dev/null | \
  jq -r '.[] | select(.ident=="<rule-id>") | (.locations[0].symbolic.key.Local.given_path) + ":" + (.locations[0].concrete.location.start_point.row+1|tostring)'
```

After editing, confirm the count is zero for that rule/file:

```bash
zizmor --format json .github/workflows/<file>.yaml 2>/dev/null | jq -r '[.[] | select(.ident=="<rule-id>")] | length'
```

A single alert on GitHub is often just one high-confidence instance of a broader pattern that also shows up at lower severity elsewhere (e.g. `note`/`info`) in the same or other workflow files. When asked to fix "the alert... across all workflows," search for every instance of the same underlying pattern, not just the one exact location the alert points to.

### 3. Known finding types and their fixes

**`template-injection`** — a `${{ ... }}` expression is interpolated directly into a `run:` shell script body. Even values that "should" be safe (job outputs, `needs.*.outputs.*`, `steps.*.outputs.*`, `secrets.*`) are flagged because the raw string is spliced into the script before bash parses it, which is a real injection vector if the value can ever contain shell metacharacters (definitely true for `inputs.*` from `workflow_dispatch`, and defense-in-depth for everything else).

Fix: move every such expression into the step's `env:` block and reference it as a shell variable instead:

```yaml
# Before
- run: |
    tag="${{ inputs.tag }}"

# After
- env:
    INPUT_TAG: ${{ inputs.tag }}
  run: |
    tag="$INPUT_TAG"
```

Apply this to every `${{ }}` that appears inside a `run: |` block — including ones referencing `needs.*.outputs.*`, `steps.*.outputs.*`, and `secrets.*` — not just `with:`/`env:` values (those are not shell-interpreted and are fine as-is).

**`excessive-permissions`** — a job (or the whole workflow) has no explicit `permissions:` block, so it inherits the repository's default token permissions, which are broader than the job needs. Fix: add a `permissions:` block to every job scoped to the minimum needed, e.g. `contents: read` for a job that only checks out code and calls external (non-GitHub) APIs.

**`artipacked`** — an `actions/checkout` step doesn't set `persist-credentials: false`. By default, checkout persists the job's Git credentials in the local git config, which can leak if that state is later uploaded as an artifact.

**Preferred fix (repo policy): disable the rule in zizmor's own config** rather than editing every checkout step — see section 4. Only fall back to the per-step `persist-credentials: false` edit if the user explicitly asks for the inline fix instead of a rule suppression:
```yaml
- uses: actions/checkout@v7
  with:
    persist-credentials: false
```
**Exception (if doing the inline fix):** if the job genuinely needs those persisted credentials later (e.g. it runs `git push` using the checkout-provided credential helper, as a release-tagging job typically does), leave that one checkout as-is rather than breaking the push — don't blanket-apply the fix without checking whether the job pushes to git afterward.

**`unpinned-uses`** — a `uses:` reference is pinned to a tag (`@v4`) rather than a commit SHA. Whether to fix this by pinning to a SHA or to suppress the rule is a project-level policy call (see the rule-suppression section below) — SHA-pinning trades convenience for supply-chain safety, and interacts with Dependabot config (see the `dependabot-maintenance` skill: SHA-pinned actions should NOT have their update-types ignored, since every bump is meaningful there).

### 4. Configuring zizmor's own rule set

If this repo runs zizmor via a `zizmor.yaml` (or similar) workflow using `its-me/action.zizmor` or the upstream action, rules are disabled/scoped via the action's inline `config:` input:

```yaml
- uses: its-me/action.zizmor@v1
  with:
    config: |
      rules:
        artipacked:
          disable: true
        unpinned-uses:
          disable: true
```

- Keep rule keys **alphabetically sorted** for readability as the list grows.
- `disable: true` turns a rule off everywhere.
- A rule can also be scoped to specific files instead of disabled globally:
  ```yaml
  rules:
    artipacked:
      ignore:
        - check-release.yaml
  ```
  Use this narrower form when the finding is a legitimate one-off (e.g. the checkout-then-push exception above) rather than something that applies repo-wide.
- **Always test a config change locally before trusting it**, since a typo in the rule id or path silently does nothing:
  ```bash
  cat > /tmp/zizmor-test.yml <<'EOF'
  rules:
    <rule-id>:
      disable: true
  EOF
  zizmor --config /tmp/zizmor-test.yml .github/workflows/*.yaml
  ```
  Look for the finding count changing and an `(N ignored)`/`(N suppressed)` note in the summary line before applying the same config to the real workflow file.

## Examples

- **User:** "fix the alert at github.com/org/repo/security/code-scanning/55 across all workflows"
- **Agent:** Fetches alert 55 via `gh api`, identifies it as `template-injection` on one file/line, greps for the same `${{ inputs.tag }}`-style pattern in every workflow file, applies the `env:` indirection fix to all of them, verifies with `zizmor --format json`, then reports back before committing.

- **User:** "is it possible to run zizmor rule X but only warn instead of failing the build"
- **Agent:** Explains that zizmor's config supports `disable`/`ignore` per rule but not a "warn only" severity override at the config level; suggests using `ignore` scoped to specific files if the goal is narrowing rather than downgrading.
