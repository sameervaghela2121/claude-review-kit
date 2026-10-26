---
name: setup-review
description: Set up (or refresh) the team's AI-assisted code review process in the current repository - inspects the code, then writes CLAUDE.md, REVIEW.md, Claude Code hooks, a pre-PR skill, reviewer subagents, a PR template, CODEOWNERS and GitHub workflows tailored to this project.
disable-model-invocation: true
argument-hint: "[audit]"
---

# Set up the AI-assisted code review process in this repository

You are setting up the review process described in this kit's GUIDE.md for the repository in the
current working directory. Everything you write must come from what you **observe in this repo**,
not from generic templates. A short file that is true beats a long file that guesses.

Arguments: `$ARGUMENTS`. If it is `audit`, run phases 1-2, then compare the existing files against
the references and report what is missing, stale or bloated, and propose edits instead of writing new files.

Reference files (read each one only when you reach the phase that needs it):

- `${CLAUDE_SKILL_DIR}/references/claude-md.md` - how to write CLAUDE.md
- `${CLAUDE_SKILL_DIR}/references/review-md.md` - how to write REVIEW.md
- `${CLAUDE_SKILL_DIR}/references/risk-tiers.md` - risk tiers, sensitive paths, CODEOWNERS
- `${CLAUDE_SKILL_DIR}/references/hooks.md` - hooks and `.claude/settings.json`
- `${CLAUDE_SKILL_DIR}/templates/` - skeletons for the files below. Fill them in; never copy placeholders through.

## Ground rules

- Never overwrite an existing file blindly. If a target file exists, read it, keep everything that is
  still true, and merge your additions in. Show the user what you changed.
- Never read or copy secrets. Do not open `.env`, `*.pem`, `*.p12`, `*.key`, credential JSON files or
  anything that looks like it holds keys. You only need to know such files exist.
- Do not commit, push, create branches remotely, change GitHub settings or add secrets. You write files;
  the developer reviews and commits them. Print the admin steps at the end instead.
- Prefer running a command over guessing what it does. A command you list in CLAUDE.md must be one you
  actually ran (or one the user confirmed), with its result noted.
- If you cannot ask the user questions (non-interactive run), make the conservative choice, and list
  every assumption in the final summary.

## Phase 1 - Inspect the repository

Use a subagent (Explore) for the broad sweep so the file dumps stay out of this conversation. Ask it to
report back, with file paths as evidence:

1. **Shape**: languages, frameworks, package managers, monorepo or single project, main entry points,
   top-level directories and what each is for (one line each).
2. **Commands**: install, run/dev, test (whole suite and a single test), lint, format, type check, build,
   migrations. Look in package.json scripts, Makefile, justfile, pyproject.toml, tox.ini, noxfile,
   pubspec.yaml, build.gradle, Cargo.toml, go.mod, Dockerfile, docker-compose, README, CONTRIBUTING,
   and existing CI config (`.github/workflows/`, `.gitlab-ci.yml`, etc.).
3. **CI**: what runs on pull requests today, job names, and which checks are already enforced.
4. **Conventions that differ from defaults**: anything a newcomer would get wrong - error-handling
   pattern, layering rules, naming, how config/env vars are loaded, where tests live, fixtures/mocks
   policy, API response shape. Only things backed by repeated evidence in the code.
5. **Generated / vendored / lock files**: paths that should never be hand-edited or reviewed.
6. **Sensitive areas**: auth, sessions/tokens, permissions, payments/billing/licensing, PII handling,
   crypto, database migrations, infrastructure-as-code, deploy/release scripts, CI workflows,
   installers/updaters, secrets handling. Paths only.
7. **Existing Claude/agent setup**: CLAUDE.md (any level), REVIEW.md, `.claude/`, AGENTS.md,
   `.cursorrules`, CODEOWNERS, PR templates.
8. **Git facts**: default branch (`git symbolic-ref refs/remotes/origin/HEAD` or `gh repo view --json defaultBranchRef`),
   remote host (GitHub, GitLab, other), and the top contributors per sensitive area
   (`git shortlog -sne --since=1.year -- <path> | head -5`) as CODEOWNERS candidates.
9. **Footguns**: anything in git history or comments that suggests past incidents - reverts
   (`git log --oneline -i --grep=revert | head -30`), "hotfix", "DO NOT", "HACK", "WARNING" comments.

## Phase 2 - Verify the commands

Run each discovered test / lint / type-check / build command once (single-test form first if the suite
is slow; use a timeout; run long ones in the background). Record for each: the exact command, pass or
fail, and roughly how long it took. If a command needs services (database, docker) or secrets you cannot
access, do not try to work around it - note it as "needs X" and ask the user in Phase 3.

These verified commands become the **Verification** section of CLAUDE.md. That section is the most
valuable thing this skill produces: it is how every future Claude session proves its work.

## Phase 3 - Ask only what you could not find out

Use AskUserQuestion, at most two rounds, only for gaps. Typical questions:

- Which commands are the "definition of done" for a change (from the verified list)?
- Which of the sensitive paths you found really need a named human owner, and who? (Offer the
  contributor candidates from Phase 1 as options.)
- Anything about this project that Claude keeps getting wrong today?
- Which optional pieces to install: CI PR review workflow, CI security review workflow, `@claude`
  mention workflow, format-on-edit hook, protected-paths hook, `/explain-area` skill.

Skip questions whose answers are already clear from the code.

## Phase 4 - Write the files

Read the matching reference before writing each file. Create only what applies to this repo.

| File | Source | Notes |
|---|---|---|
| `CLAUDE.md` | `references/claude-md.md` | Target under 120 lines. Subdirectory CLAUDE.md only for truly different sub-projects (e.g. `frontend/`, `infra/`). |
| `REVIEW.md` | `references/review-md.md` | Root file; add a subdirectory REVIEW.md only for a high-risk area with its own rules. |
| `.claude/settings.json` | `references/hooks.md` | Merge with any existing file. Validate JSON afterwards. |
| `.claude/hooks/*.sh` | `references/hooks.md` | Only for tools that are actually installed in this repo. `chmod +x`. |
| `.claude/skills/pre-pr/SKILL.md` | `templates/claude/pre-pr.SKILL.md` | Fill in the verified commands. |
| `.claude/skills/review-tune/SKILL.md` | `templates/claude/review-tune.SKILL.md` | Copy as is unless the repo is not on GitHub. |
| `.claude/skills/explain-area/SKILL.md` | `templates/claude/explain-area.SKILL.md` | Optional. |
| `.claude/agents/diff-reviewer.md` | `templates/claude/diff-reviewer.agent.md` | Copy; adjust tool list only if needed. |
| `.claude/agents/security-reviewer.md` | `templates/claude/security-reviewer.agent.md` | Add this repo's sensitive areas to its checklist. |
| `.github/pull_request_template.md` | `templates/github/pull_request_template.md` | Merge with an existing template. |
| `.github/CODEOWNERS` | `references/risk-tiers.md` | Only confirmed owners. Always include `.github/workflows/`, `CLAUDE.md`, `REVIEW.md`, `.claude/`. |
| `.github/workflows/claude-pr-review.yml` | `templates/github/claude-pr-review.yml` | Set the default branch name. GitHub only. |
| `.github/workflows/claude-security-review.yml` | `templates/github/claude-security-review.yml` | Fill `exclude-directories` from Phase 1 item 5. |
| `.github/claude-security-instructions.txt` | - | Optional: 5-15 lines naming this repo's sensitive areas and threat model. |
| `.github/workflows/claude.yml` | `templates/github/claude.yml` | Optional `@claude` mention workflow. |

If the remote is GitLab or something else, skip the `.github/` files and say in the summary that the CI
part needs the GitLab CI/CD integration instead (https://code.claude.com/docs/en/gitlab-ci-cd).

## Phase 5 - Check your own work

Before you report:

1. `wc -l CLAUDE.md REVIEW.md` - if CLAUDE.md is over ~150 lines, cut it. For each line ask "would
   removing this cause Claude to make a mistake?"; delete lines that fail the test.
2. Every command in CLAUDE.md appears in your Phase 2 results (or was confirmed by the user).
3. No template placeholders left in any file you wrote:
   `grep -rnE '<[A-Z][A-Z_]+|filled in by /setup-review|<one |<rule |<path>|TBD' CLAUDE.md REVIEW.md .claude .github`
   must print nothing.
4. JSON parses: `python3 -m json.tool .claude/settings.json > /dev/null`.
5. YAML parses: `python3 -c "import sys,yaml;[yaml.safe_load(open(f)) for f in sys.argv[1:]]" .github/workflows/claude-*.yml`
   or, if PyYAML is missing, `ruby -ryaml -e 'ARGV.each{|f| YAML.load_file(f)}' .github/workflows/claude-*.yml`
   (if neither is available, say so).
6. Each hook script runs on a sample payload without error, e.g.
   `echo '{"tool_input":{"file_path":"<a real source file>"}}' | .claude/hooks/format-file.sh; echo $?`
   and leaves `git diff` unchanged for an already-formatted file.
7. Hand the diff to the `diff-reviewer` subagent you just created with this instruction: "Review this
   setup diff for statements that are not true for this repository, commands that do not exist, and
   rules that contradict each other." Fix what it finds once; do not loop.

## Phase 6 - Report

End with a short summary for the developer:

- **Files written / updated** - one line each, with what was merged rather than created.
- **Verified commands** - the table from Phase 2.
- **Assumptions** - anything you decided without asking.
- **Next steps for the developer**: review the diff, then commit on a branch such as
  `chore/ai-review-setup` and open a PR (the new PR template applies to it).
- **Admin steps (a repo admin must do these once)** - print them, do not run them:
  1. Org-level Actions secret `ANTHROPIC_API_KEY` (Claude Console API key) available to this repo.
  2. Claude GitHub App installed on this repo (https://github.com/apps/claude) - needed by the
     PR review and `@claude` workflows.
  3. Branch protection on the default branch. Print the `gh api` command from the "Branch
     protection" section of `references/risk-tiers.md` with this repo's owner/name, default branch
     and existing CI job names filled in.
- **What this setup does not cover** - be honest (e.g. "tests need a local MongoDB, so Claude cannot
  run the full suite unattended").
