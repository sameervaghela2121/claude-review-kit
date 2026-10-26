# Writing REVIEW.md

REVIEW.md tells the **PR reviewer** what matters in this repository: what counts as Important, what
to ignore, and what to always check. It changes what the review finds and reports; general project
context stays in CLAUDE.md.

Who reads it in this setup:

- The CI review workflow (`.github/workflows/claude-pr-review.yml`) reads the copy on the **base
  branch**, so a PR cannot weaken the rules it is reviewed against. Changes to REVIEW.md take effect
  after they are merged.
- The `diff-reviewer` subagent and `/pre-pr` read it locally.
- Claude's managed Code Review service (Team/Enterprise plans) reads it natively, so this file keeps
  working if the organisation upgrades.
- The built-in local `/code-review` command does **not** read it - only CLAUDE.md. Put any rule that
  must apply everywhere in CLAUDE.md.

## What good REVIEW.md files do

1. **Define failure for this project.** What would actually hurt: wrong data, leaked PII, broken
   rollback, a misrouted config, docs that lie. Those are Important. Everything else is Nit at most.
2. **Cap nits.** "Report at most 5 Nits per review." Agent authors pay a full review cycle per nit.
3. **List what not to report.** Anything CI already enforces (lint, format, types), generated code,
   lockfiles, vendored code, test fixtures that intentionally break prod rules.
4. **List what to always check.** The 3-8 rules this team keeps getting wrong. Write each as a
   checkable statement.
5. **Demand evidence.** Every finding cites `file:line`; behaviour claims cite the source, not a guess
   from naming.
6. **Fix the summary format.** A top-level summary with the count of findings by severity and an
   explicit "No issues found" when there are none.
7. **Respect history.** Label issues that existed before the PR as Pre-existing, and don't block the
   author on them.
8. **Converge.** On re-review, look only at the new commits; don't re-report issues already raised;
   when the same class of bug appears in several places, report the class once with every location.

Keep it under ~80 lines. If it grows, split it: `@review-security.md` lines in the root REVIEW.md
include sibling files at the repo root (managed Code Review supports up to 8 includes; the CI workflow
in this kit simply reads them too).

## Severity vocabulary (used by the CI reviewer and the managed service)

- **Important** - should be fixed before merge.
- **Nit** - optional improvement.
- **Pre-existing** - real issue, but not introduced by this PR.

## Skeleton

```markdown
# Review instructions

## What Important means here
Reserve Important for findings that would <project-specific failures: break behaviour, corrupt or
leak data, break backward compatibility of the API/DB, block a rollback, misconfigure deploys>.
Style, naming and refactoring suggestions are Nit at most.
<Optional escalation: any violation of a CLAUDE.md rule is Important.>

## Cap the nits
Report at most 5 Nits per review. Skip Nits entirely on PRs labelled `hotfix`.

## Do not report
- Anything CI already enforces: <lint tool>, <formatter>, <type checker>
- Generated files: <paths>
- Lockfiles and vendored code: <paths>
- <test-only code that intentionally violates production rules>

## Always check
- <rule 1, e.g. "Every new API route has an integration test in tests/api/">
- <rule 2, e.g. "Every Mongo query is scoped to the caller's workspace_id">
- <rule 3, e.g. "No email, user id or request body in log lines">
- <rule 4, e.g. "DB migrations are backward compatible with the previous release">

## Higher bar for some paths
- In `<scripts/, docs/, prototypes/>`, only report if near-certain and severe.

## Evidence and format
- Cite `file:line` for every finding. Behaviour claims need a citation in the source, not an
  inference from a name.
- Start with a summary: number of findings by severity, or "No issues found."
- Mark issues that existed before this PR as Pre-existing.
- On re-review, review only the new commits and do not repeat earlier findings.
- When a class of bug appears in several places, report it once and list every location.
```

## Writing "Always check" rules from this repo

Good sources found in Phase 1: reverts and hotfixes in git history, `DO NOT` / `WARNING` comments,
sensitive areas, conventions the code follows consistently, and answers the developer gave in Phase 3.
Prefer 4 sharp rules over 15 vague ones.
