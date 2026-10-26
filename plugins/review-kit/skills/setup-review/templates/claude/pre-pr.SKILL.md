---
name: pre-pr
description: Run the pre-PR checks for this repository - verification commands, a fresh-context review against REVIEW.md, a correctness review and (for risky changes) a security review - then draft the PR description with evidence. Use before opening or updating a pull request.
disable-model-invocation: true
argument-hint: "[base-branch]"
---

# Pre-PR checks

Base branch: `$ARGUMENTS` if given, otherwise <DEFAULT_BRANCH>.

Work through the steps in order. Stop and report if a step fails and you cannot fix it at the root cause.

## 1. Scope

Run `git diff --stat <base>...HEAD` and `git status`. If the change mixes unrelated work, say so and
suggest splitting it before going further. Note which risk tier the change is (see the table at the
end) - this decides which reviews run.

## 2. Verify

Run the verification commands from CLAUDE.md:

<VERIFICATION_COMMANDS - filled in by /setup-review, e.g.
- `uv run ruff check .`
- `uv run mypy app`
- `uv run pytest tests/<area touched> -q`, then `uv run pytest -q`>

Fix failures at the root cause. Keep the exact commands and the tail of their output - they go into
the PR description as evidence.

## 3. Review against REVIEW.md (fresh context)

Use the `diff-reviewer` subagent on `<base>...HEAD`. It reads REVIEW.md and CLAUDE.md itself.

## 4. Correctness review

Run `/code-review` (use `/code-review high` for High-risk changes).

## 5. Security review (Medium touching input handling / auth / data, and all High)

Run `/security-review`. For High-risk changes, also use the `security-reviewer` subagent.

## 6. Act on findings - and converge

- Fix Important findings. For each Nit, fix it or note why not.
- When one class of bug shows up, fix every instance, not just the reported line.
- Re-run steps 2-5 **on the new changes only**. Do at most **3 review rounds** in total. If findings
  are still appearing after round 3, stop and tell the developer the approach may need rethinking -
  do not keep cycling.

## 7. Draft the PR description

Fill `.github/pull_request_template.md`:

- **What / Why** - one paragraph, plus the issue link if there is one.
- **Testing** - the commands from step 2 and their result, plus any manual check (curl output,
  screenshot). Evidence, not "tested locally".
- **Risk** - tier and why; list sensitive paths touched.
- **How to revert** - "revert this commit" is fine only if there is no migration, data change or
  config change; otherwise say what else must happen.
- **Review notes** - findings you chose not to fix, with the reason.

Show the draft to the developer. Do not open the PR or push unless they ask.

## Risk tiers

| Tier | Touches | Reviews in this skill |
|---|---|---|
| Low | docs, tests only, comments, small refactors | steps 2-4 |
| Medium | application code, endpoints, dependencies | steps 2-4, plus 5 if it handles input/auth/data |
| High | <HIGH_RISK_PATHS - filled in by /setup-review> | steps 2-5, `/code-review high`, small PR, rollback plan |
