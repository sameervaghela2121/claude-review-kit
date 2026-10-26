# claude-review-kit

A Claude Code plugin marketplace. It contains one plugin, **review-kit**, which sets up our
AI-assisted code review process in any repository by reading the repository's own code.

Read **[GUIDE.md](GUIDE.md)** for the process, the rules, and the setup steps.

## Quick start

```bash
claude plugin marketplace add sameervaghela2121/claude-review-kit
```

```bash
claude plugin install review-kit@claude-review-kit
```

Then, at the root of a project repository, on a new branch:

```text
/review-kit:setup-review
```

Run `/review-kit:setup-review audit` later to check an existing setup.

## Layout

```text
.claude-plugin/marketplace.json           marketplace manifest
plugins/review-kit/
  .claude-plugin/plugin.json              plugin manifest (bump version on every change)
  skills/setup-review/SKILL.md            the setup procedure Claude follows
  skills/setup-review/references/         how to write CLAUDE.md, REVIEW.md, hooks, risk tiers
  skills/setup-review/templates/claude/   pre-pr, review-tune, explain-area skills; reviewer subagents
  skills/setup-review/templates/github/   PR template and GitHub workflows
GUIDE.md                                  the team guide
```

## Developing the kit

```bash
claude plugin validate .
```

To try local changes without publishing, start Claude Code in a test repository with
`claude --plugin-dir /path/to/claude-review-kit/plugins/review-kit`.
