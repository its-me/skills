---
name: zizmor
description: Set up zizmor GitHub Actions security scanning in a repository using one of two approaches (default or strict), and investigate/fix zizmor findings from GitHub code-scanning alerts or the zizmor CLI. Use this skill when asked to add zizmor to a repo, harden or security-review Actions workflows, fix a code-scanning alert produced by zizmor, or tune which zizmor rules run/are suppressed.
argument-hint: "[strict]"
---

## Instructions

### 1. Pick the approach

Use the **default** approach unless the user passes `strict` (or explicitly asks for strict/SHA-pinning):

- **default** — actions stay pinned to major-version tags (`@v7`); the `artipacked` and `unpinned-uses` rules are disabled via zizmor's inline config, so no per-step edits are needed for them.
- **strict** — every rule stays enabled. Every `uses:` reference is pinned to a full commit SHA (with the tag kept as a trailing comment), and every `actions/checkout` step sets `persist-credentials: false`.

### 2. Install the zizmor workflow

Write `.github/workflows/zizmor.yaml` from the template matching the approach.

**default:**

```yaml
name: zizmor

on:
  push:
    branches: ['**']
  pull_request:
  workflow_dispatch:

permissions:
  security-events: write
  contents: read
  actions: read

jobs:
  zizmor:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - uses: its-me/action.zizmor@v1
        with:
          config: |
            rules:
              artipacked:
                disable: true
              unpinned-uses:
                disable: true
```

**strict:**

```yaml
name: zizmor

on:
  push:
    branches: ['**']
  pull_request:
  workflow_dispatch:

permissions:
  security-events: write
  contents: read
  actions: read

jobs:
  zizmor:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@9c091bb21b7c1c1d1991bb908d89e4e9dddfe3e0 # v7.0.0
        with:
          persist-credentials: false
      - uses: its-me/action.zizmor@52c918723b0bd2be8f1d275ad85b4a0b7ee91e03 # v1.0.0
```

The SHAs above are the known-good pins for `actions/checkout@v7.0.0` and `its-me/action.zizmor@v1.0.0`; reuse them as-is unless the user asks for a newer release.

### 3. Bring every other workflow in line with the approach

Go through each file in `.github/workflows/` and edit every action to match the selected approach:

**default** — leave tag-pinned `uses:` references and checkout steps as they are (`artipacked` and `unpinned-uses` are suppressed by the config). Still fix `template-injection` and `excessive-permissions` findings (section 5) — those are never suppressed.

**strict** — for every `uses:` reference pinned to a tag, swap the tag for the commit hash it currently resolves to, keeping the tag as a comment:

```bash
gh api repos/<owner>/<repo>/commits/<tag> --jq .sha
```

```yaml
# Before
- uses: actions/setup-node@v4
# After
- uses: actions/setup-node@2028fbc5c25fe9cf00d9f06a71cc4710d4507903 # v4.4.0
```

Prefer resolving the most specific release tag (e.g. `v4.4.0` rather than the floating `v4`) so the comment stays truthful; `gh api repos/<owner>/<repo>/releases/latest --jq .tag_name` finds it. Also add `persist-credentials: false` to every `actions/checkout` step:

```yaml
- uses: actions/checkout@9c091bb21b7c1c1d1991bb908d89e4e9dddfe3e0 # v7.0.0
  with:
    persist-credentials: false
```

**Exception:** if a job genuinely needs the persisted credentials later (e.g. it runs `git push` using the checkout-provided credential helper, as a release-tagging job typically does), leave that one checkout as-is and scope an `ignore` for it in the config (section 6) rather than breaking the push.

### 4. Verify locally

Run zizmor against all workflows with a config that mirrors the approach:

```bash
# strict: no suppressions
zizmor .github/workflows/*.yaml

# default: replicate the inline config
cat > /tmp/zizmor-default.yml <<'EOF'
rules:
  artipacked:
    disable: true
  unpinned-uses:
    disable: true
EOF
zizmor --config /tmp/zizmor-default.yml .github/workflows/*.yaml
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

### 5. Fixing individual findings

Given a code-scanning alert URL like `https://github.com/<owner>/<repo>/security/code-scanning/<N>`, fetch the exact finding before guessing at a fix:

```bash
gh api repos/<owner>/<repo>/code-scanning/alerts/<N>
```

This returns the rule id (`rule.id`, e.g. `zizmor/template-injection`), severity, and the exact `most_recent_instance.location` (`path`, `start_line`, `end_line`). Always read the flagged file at that location before editing — don't pattern-match from memory. A single alert is often just one high-confidence instance of a broader pattern that also shows up at lower severity elsewhere; when asked to fix an alert "across all workflows," search for every instance of the underlying pattern.

**`template-injection`** — a `${{ ... }}` expression is interpolated directly into a `run:` shell script body. Even values that "should" be safe (job outputs, `needs.*.outputs.*`, `steps.*.outputs.*`, `secrets.*`) are flagged because the raw string is spliced into the script before bash parses it. Fix: move every such expression into the step's `env:` block and reference it as a shell variable:

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

`with:`/`env:` values are not shell-interpreted and are fine as-is.

**`excessive-permissions`** — a job (or the whole workflow) has no explicit `permissions:` block, so it inherits the repository's default token permissions. Fix: add a `permissions:` block scoped to the minimum needed, e.g. `contents: read` for a job that only checks out code.

**`artipacked`** — an `actions/checkout` step doesn't set `persist-credentials: false`. Fix depends on the approach: under **default** the rule is already disabled via config; under **strict** add `persist-credentials: false` per step (section 3, including the git-push exception).

**`unpinned-uses`** — a `uses:` reference is pinned to a tag rather than a commit SHA. Under **default** the rule is disabled via config; under **strict** swap the tag for the hash (section 3). SHA-pinning interacts with Dependabot config — see the `dependabot-maintenance` skill: SHA-pinned actions should NOT have their update-types ignored, since every bump is meaningful there.

### 6. Tuning the rule config

Rules are disabled/scoped via the action's inline `config:` input:

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
- `disable: true` turns a rule off everywhere. A rule can instead be scoped to specific files when the finding is a legitimate one-off (e.g. the checkout-then-push exception):
  ```yaml
  rules:
    artipacked:
      ignore:
        - check-release.yaml
  ```
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

- **User:** "/zizmor" (in a repo without scanning)
- **Agent:** Writes `.github/workflows/zizmor.yaml` from the default template, leaves tag-pinned actions in other workflows alone, fixes any `template-injection`/`excessive-permissions` findings, verifies with `zizmor --config /tmp/zizmor-default.yml`.

- **User:** "/zizmor strict"
- **Agent:** Writes the strict template, resolves every tag-pinned `uses:` across all workflows to a commit SHA with a `# vX.Y.Z` comment, adds `persist-credentials: false` to every checkout (scoping an `ignore` for any job that must push with persisted credentials), verifies with plain `zizmor`.

- **User:** "fix the alert at github.com/org/repo/security/code-scanning/55 across all workflows"
- **Agent:** Fetches alert 55 via `gh api`, identifies it as `template-injection` on one file/line, greps for the same pattern in every workflow file, applies the `env:` indirection fix to all of them, verifies with `zizmor --format json`, then reports back before committing.
