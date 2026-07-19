# skills

Claude Code skills for hardening and maintaining GitHub Actions workflows.

## Skills

- **[zizmor](zizmor/SKILL.md)** — set up [zizmor](https://woodruffw.github.io/zizmor/) GitHub Actions security scanning using a `default` or `strict` approach, and investigate/fix zizmor findings from code-scanning alerts or the CLI.
- **[dependabot](dependabot/SKILL.md)** — configure and maintain Dependabot for GitHub Actions workflows, including cooldown periods and suppressing redundant minor/patch bump PRs for major-version-tagged actions.

## Usage

Drop this repository into `~/.claude/skills` (or a project's `.claude/skills`) so Claude Code can discover and invoke each skill by name, e.g. `/zizmor`, `/zizmor strict`, `/zizmor fix`, `/dependabot`.

## License

[MIT](LICENSE)
