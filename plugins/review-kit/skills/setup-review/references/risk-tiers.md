# Risk tiers, sensitive paths and CODEOWNERS

Risk is decided by **what the change touches**, not by who wrote it. Agent approvals never count as
human approvals. The person who ran the agent is the author and is responsible for the change.

## Tiers

| Tier | Typical changes | Before PR | Human approvals | Notes |
|---|---|---|---|---|
| Low | docs, copy, comments, tests only, internal tooling, small refactors with full test coverage | `/pre-pr` | 1 (can be any teammate) | CI review + CI checks are enough evidence |
| Medium | features and bug fixes in application code, new endpoints, dependency bumps | `/pre-pr` incl. security review if it touches input handling | 1 | Reviewer must read the PR's Testing evidence, not just the diff |
| High | auth, permissions, billing/licensing, PII, crypto, DB migrations, infra/IaC, deploy and release scripts, CI workflows, installers/updaters, `CLAUDE.md` / `REVIEW.md` / `.claude/` | `/pre-pr` + `/security-review` + `/code-review high` | 1 code owner (2 people for billing/auth/migrations) | Small PRs only. PR must state the rollback plan. Prefer a feature flag |

Start strict and relax tiers only when the revert/incident numbers show it is safe.

## Detecting sensitive paths

Look for these, and confirm with the developer:

- Auth: `auth`, `login`, `session`, `token`, `jwt`, `oauth`, `cognito`, `permission`, `rbac`, `middleware`
- Money/licensing: `billing`, `payment`, `invoice`, `subscription`, `license`, `entitlement`, `stripe`
- Data: `migrations`, `schema`, `models` with PII fields, backup/restore scripts
- Infra/release: `deploy`, `infra`, `terraform`, `*.tf`, `cdk`, `k8s`, `helm`, `Dockerfile`, `provision*.sh`, `release`, `fastlane`, signing config
- CI: `.github/workflows/`, `.gitlab-ci.yml`
- Agent config: `CLAUDE.md`, `REVIEW.md`, `.claude/` (these steer every future review)

## CODEOWNERS

GitHub requests review from code owners automatically, and branch protection can require their
approval. Only list people the developer confirmed. Use GitHub handles or teams, never emails you
guessed. Later lines win, so put the broad default first.

```text
# Default: any teammate may approve (leave empty or set a team)
# *                       @org/team

# High-risk areas
/.github/workflows/       @owner-handle
/CLAUDE.md                @owner-handle
/REVIEW.md                @owner-handle
/.claude/                 @owner-handle
/<auth path>/             @owner-handle @second-handle
/<migrations path>/       @owner-handle
/<deploy or infra path>/  @owner-handle
```

If the developer cannot name owners yet, write the file with the high-risk paths commented out and
call it out in the summary.

## Branch protection

A repo admin runs this once. Fill in OWNER/REPO, the default branch, and the names of the CI jobs
that already run on pull requests.

```bash
gh api -X PUT repos/OWNER/REPO/branches/DEFAULT_BRANCH/protection --input - <<'JSON'
{
  "required_status_checks": { "strict": true, "contexts": ["security-review", "CI-JOB-1", "CI-JOB-2"] },
  "enforce_admins": true,
  "required_pull_request_reviews": {
    "required_approving_review_count": 1,
    "dismiss_stale_reviews": true,
    "require_code_owner_reviews": true,
    "require_last_push_approval": true
  },
  "required_conversation_resolution": true,
  "restrictions": null,
  "allow_force_pushes": false,
  "allow_deletions": false
}
JSON
```

- `required_conversation_resolution`: every Claude finding must be fixed or resolved with a reason.
- Do **not** add `claude-review` as a required check: it runs when the PR opens, not on every push, so
  it would block PRs after new commits. Conversation resolution enforces its findings instead.
- `dismiss_stale_reviews` + `require_last_push_approval`: an approval is only valid for the commit it
  was given on.
