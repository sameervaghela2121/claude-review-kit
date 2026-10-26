# AI-assisted code review with Claude Code

**Who this is for:** every developer who writes code with Claude Code or Claude, and every repo admin.

**What you get:** a review process modelled on how the Claude Code team reviews its own code. Claude
does most of the reading, before and after the PR is opened. Human time goes to the decisions only
humans can make. One command, `/review-kit:setup-review`, sets it up in any repository by reading
that repository's code.

Sources: the article "How the Claude Code team does code review" (shared internally), [Claude Code best practices](https://code.claude.com/docs/en/best-practices),
[Code Review](https://code.claude.com/docs/en/code-review), [Code Review setup](https://code.claude.com/docs/en/code-review-setup),
[Customize Code Review](https://code.claude.com/docs/en/code-review-customize), [GitHub Actions](https://code.claude.com/docs/en/github-actions).

---

## 1. The idea in one page

Code review has two jobs: **catch bugs before they ship** and **share understanding of the codebase**.
When agents write most of the code, nobody can read every line. So we stop asking "did someone read
the code?" and ask "what could go wrong with this change, and which check, automated or human, is the
right one to catch it?"

Human time counts twice: time spent reviewing **and** time spent fixing bugs that review missed.
Cutting the first without the second only moves the work. Our goal:

> Most PRs merge with **at most one human in the loop** (who may be the author), **without** a rise in
> reverts or incidents.

We get there with several layers of checks. Each one catches some bugs and misses others, few bugs get
through all of them, and a fast revert handles the rest.

| # | Layer | What runs | Blocks merge? |
|---|---|---|---|
| 1 | **Before the PR** | Claude runs the project's verification commands and shows evidence; `/pre-pr` runs a fresh-context review against REVIEW.md, `/code-review` and `/security-review`; hooks format code on every edit | You don't open the PR until it passes |
| 2 | **AI review on every PR** | `Claude PR Review` workflow posts inline findings (Important / Nit / Pre-existing) using REVIEW.md from the base branch | Every thread must be fixed or resolved with a reason |
| 3 | **Security review** | `Claude Security Review` workflow | Required check + threads must be resolved |
| 4 | **CI** | Your existing lint, type check, tests and build | Required checks |
| 5 | **Human approval by risk** | 1 approval; code owners for high-risk paths; approvals dismissed on new commits | Yes |
| 6 | **Rollout** | Feature flags for risky features; internal users first where possible | - |
| 7 | **Fast revert** | Every PR says how to revert it; the author fixes what they land | - |

## 2. Rules everyone follows

1. **You land it, you own it.** The person who ran the agent is the author and answers for the change.
   If it breaks, you fix it or revert it. This is not about blame; you simply have the most context.
2. **Show, don't tell.** "Done" means test output, command results or a screenshot in the PR, not "tested locally".
3. **Bot approvals never count** as human approvals. Claude comments; people approve.
4. **Every review thread is fixed or resolved with a reason** before merge, Claude's included.
5. **Risk is set by the change, not the author** (section 5).
6. **Review must converge.** At most 3 review rounds per PR. After that, rethink the approach instead
   of fixing one more line. Re-reviews look at the new commits only. Fix every instance of a bug class, not one line.
7. **Small PRs.** Easy to review, debug and revert.
8. **Pivot early.** If someone in the channel says "we shouldn't build this" or "there's a better path",
   close or change the PR. Agent-written code is cheap to throw away.
9. **Repeated finding → move it left.** When the reviewer keeps flagging the same thing, fix it at the
   source: a lint rule, a hook, a test, or a CLAUDE.md/REVIEW.md line.

## 3. Setup

### 3.1 Once for the organisation (an admin)

1. **Host this kit.** It is hosted at `sameervaghela2121/claude-review-kit`. It is a
   Claude Code plugin marketplace that contains one plugin, `review-kit`.
2. **API key.** Create a Claude Console API key for CI (own workspace, with a monthly spend limit). Add it
   as an **organisation Actions secret** named `ANTHROPIC_API_KEY`, available to the repos that use this process.
3. **GitHub App.** Install the [Claude GitHub App](https://github.com/apps/claude) on the organisation
   (all repos or selected repos). The PR review and `@claude` workflows use it to post comments.

> We are not on a Claude Team/Enterprise plan, so we use the GitHub Action rather than the managed
> Code Review service. If we upgrade, an Owner can switch on managed Code Review. It reads the same
> REVIEW.md and CLAUDE.md, and we can then delete `claude-pr-review.yml`.

### 3.2 Once per developer

```bash
claude plugin marketplace add sameervaghela2121/claude-review-kit
```

```bash
claude plugin install review-kit@claude-review-kit
```

Also install the [GitHub CLI](https://cli.github.com) and run `gh auth login`. Claude uses `gh` for PRs and review comments.

### 3.3 Once per repository (any developer, then an admin for step 4)

1. Open Claude Code at the repo root on a fresh branch (`git switch -c chore/ai-review-setup`).
2. Run `/review-kit:setup-review`. Claude:
   - inspects the code: stack, commands, CI, conventions, generated files, sensitive areas, history of reverts;
   - **runs** the test/lint/build commands it finds, so CLAUDE.md only lists commands that work;
   - asks you only what it couldn't work out (definition of done, owners of sensitive paths, optional pieces);
   - writes the files below, merging into any that already exist;
   - checks its own work (file length, JSON/YAML validity, hook dry-runs, a fresh-context review of the setup).
3. **Read the diff.** You own these files. Cut anything that isn't true or isn't needed. Commit and open
   a PR. The new PR template applies to that PR.
4. A repo admin applies branch protection (section 8) and checks that the secret and app cover the repo.

To check an existing setup later, run `/review-kit:setup-review audit`.

### What gets created in your repo

| File | Purpose |
|---|---|
| `CLAUDE.md` | Project context loaded into every session and every review: commands, the **Verification** section, conventions that differ from defaults, gotchas, sensitive areas. Kept short; every line must prevent a mistake. |
| `REVIEW.md` | Rules for the PR reviewer only: what Important means here, Nit cap, what not to report, what to always check, evidence format. |
| `.claude/settings.json` + `.claude/hooks/` | Format-on-edit hook, a hook that blocks edits to secrets and generated files, deny rules for reading `.env` files. |
| `.claude/skills/pre-pr/` | `/pre-pr`: verification, then reviews, then the PR description with evidence. |
| `.claude/skills/review-tune/` | `/review-tune`: tunes the reviewer from what humans did with its comments, and records incidents. |
| `.claude/skills/explain-area/` | `/explain-area <path>`: an up-to-date explainer of one part of the code. |
| `.claude/agents/diff-reviewer.md` | Fresh-context reviewer that applies REVIEW.md locally. |
| `.claude/agents/security-reviewer.md` | Security reviewer that knows this repo's sensitive areas. |
| `.github/pull_request_template.md` | What and why, Testing (evidence), Risk, How to revert, Review notes. |
| `.github/CODEOWNERS` | Human owners for high-risk paths, CI workflows and the agent config files. |
| `.github/workflows/claude-pr-review.yml` | AI review on every non-draft PR. |
| `.github/workflows/claude-security-review.yml` | Security review on every PR. |
| `.github/workflows/claude.yml` (optional) | `@claude` in PR/issue comments, e.g. "@claude review the new commits". |

Because these files live in the repo, everyone who clones it gets `/pre-pr`, the hooks and the
subagents. Only `setup-review` itself needs the plugin.

## 4. Everyday workflow

1. **Plan non-trivial work first.** Use plan mode (`Shift+Tab`) to explore and plan; for big features,
   ask Claude to interview you and write a spec. Skip the plan if you could describe the diff in one sentence.
2. **Give Claude a way to verify.** Name the test or check in the prompt ("…write a failing test, fix it,
   run the suite"). CLAUDE.md's Verification section is the default.
3. **Manage context.** `/clear` between unrelated tasks; "use subagents to investigate X" for wide
   searches; after two failed corrections, `/clear` and write a better prompt.
4. **Run `/pre-pr`** before opening or updating a PR. It verifies, reviews (max 3 rounds), and drafts the
   PR description with evidence.
5. **Open the PR.** CI review, security review and CI checks run. Fix or resolve every thread.
6. **Human review** at the level the risk tier asks for. Reviewers read the evidence and look at
   intent, design and risk. They don't re-read lines the machines already checked.
7. **Merge, watch, own.** If something breaks, revert first, then fix forward. Run `/review-tune incident <PR>`.

## 5. Risk tiers

| Tier | Typical changes | Before the PR | Human approvals |
|---|---|---|---|
| **Low** | docs, copy, comments, tests only, small covered refactors | `/pre-pr` | 1 teammate |
| **Medium** | features, bug fixes, new endpoints, dependency bumps | `/pre-pr` (+ security review if it handles input/auth/data) | 1 teammate |
| **High** | auth, permissions, billing/licensing, PII, crypto, DB migrations, infra/IaC, deploy/release, CI workflows, installers/updaters, CLAUDE.md / REVIEW.md / `.claude/` | `/pre-pr` + `/security-review` + `/code-review high` | Code owner (2 people for auth, billing, migrations); rollback plan in the PR; feature flag where possible |

Start strict. Relax a tier only when the revert and incident numbers say it is safe.

**What humans are for:** deciding whether to build it at all; product taste; architecture (it sets the
patterns others copy); auditing dangerous code; checking the evidence (if you can't say what the
checks would miss, do a real review rather than a 👍); and owning the result.

## 6. Making the reviewer better

- **Incidents are the evaluation set.** Every revert or incident gets a line in
  `docs/review-incidents.md` via `/review-tune incident <PR>`: where the bug was, and which layer should
  have caught it. Judge every change to CLAUDE.md, REVIEW.md, hooks or workflows against that list.
- **Measure noise.** Every couple of weeks run `/review-tune`. It sorts Claude's recent comments into
  fixed / rejected / intentional / didn't matter, and proposes edits to cut noise and close misses.
- **Prefer deterministic fixes.** A lint rule, a test or a hook beats another sentence in REVIEW.md.
- **Track** (a spreadsheet is fine to start): approvals per PR, human comments per PR, time a PR waits
  for a person, review rounds per PR. Use the revert rate and incident count as guardrails: if they rise,
  we haven't saved time, only moved it.
- **Keep files short.** If Claude ignores a rule, the file is probably too long. Prune before adding.

## 7. Learning the codebase without reading every diff

Reading a one-line diff teaches little. Before working in an unfamiliar area, run
`/explain-area <path or feature>` for an explainer that is always up to date: purpose, key files, flow,
data, invariants, and how to change it safely. Keep working in open channels so others see what is
being built and can say early if it shouldn't be.

## 8. Branch protection (repo admin)

`/setup-review` prints this with your repo's values filled in. Replace the check names with your real CI job names.

```bash
gh api -X PUT repos/OWNER/REPO/branches/main/protection --input - <<'EOF'
{
  "required_status_checks": { "strict": true, "contexts": ["security-review", "YOUR-CI-JOB-1", "YOUR-CI-JOB-2"] },
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
EOF
```

- `required_conversation_resolution` is what forces every Claude finding to be fixed or resolved with a reason.
- `claude-review` is **not** a required check on purpose: it runs when a PR opens, not on every push, so
  requiring it would block PRs after a new commit. Its findings are enforced through conversation resolution.
- `dismiss_stale_reviews` + `require_last_push_approval` means an approval is only good for the commit
  it was given on, so an agent can't push something new after a human approved.
- Add a merge queue if your GitHub plan supports it, so checks run against main with the change applied.

## 9. FAQ and limits

- **Why both CLAUDE.md and REVIEW.md?** CLAUDE.md is read by every session and every review, including
  the built-in local `/code-review`. REVIEW.md is read only by PR review (CI workflow, `diff-reviewer`,
  managed Code Review). Rules that must apply everywhere go in CLAUDE.md.
- **Why read REVIEW.md from the base branch?** So a PR can't loosen the rules it's reviewed against.
  Your REVIEW.md changes apply after they merge. CLAUDE.md and `.claude/` are read from the PR itself,
  which is why all three are in CODEOWNERS and count as High risk.
- **Cost.** Each PR review and security review uses API tokens on the org key plus GitHub Actions
  minutes. The PR review runs once per PR by default; add `synchronize` to the workflow triggers to
  review every push. Set a spend limit in the Claude Console.
- **Forks and drafts** are not reviewed automatically. Secrets aren't available to fork PRs.
- **A review with no findings doesn't mean the code is correct.** It's one layer.
- **GitLab repos:** the Claude Code files work as they are; for CI use [Claude Code GitLab CI/CD](https://code.claude.com/docs/en/gitlab-ci-cd).
- **Updating the kit:** change the files in this repo and bump `version` in
  `plugins/review-kit/.claude-plugin/plugin.json`. Developers get it with `claude plugin marketplace update claude-review-kit`.
