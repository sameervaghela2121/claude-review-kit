# Writing CLAUDE.md

CLAUDE.md is loaded into **every** Claude Code session in this repo, every local `/code-review`, and
every CI review run. It is the reviewer's idea of "our rules": code that breaks a rule in it gets
flagged. Every line costs attention in every session, so it must earn its place.

The test for every line: **"Would removing this cause Claude to make a mistake?"** If not, delete it.

## Include

- Commands Claude can't guess (exact test / single-test / lint / typecheck / build commands).
- The **Verification** section: what to run before calling a change done, and the instruction to
  show the output as evidence.
- Conventions that differ from the language/framework defaults.
- Architectural decisions that aren't obvious from the code ("all DB access goes through
  `app/adapters/`, never from route handlers").
- Environment quirks (required env vars by name only, services that must be running, ports).
- Gotchas: things that look wrong but are intentional, things that look fine but break prod.
- Sensitive areas and who owns them (paths only; details live in REVIEW.md/CODEOWNERS).
- Repo etiquette: branch naming, commit style, "run /pre-pr before opening a PR".

## Exclude

- Anything Claude can learn by reading the code (folder-by-folder tours, what each file does).
- Standard language conventions ("use type hints", "write clean code").
- Long API docs or tutorials - link to them instead.
- Information that changes often (current sprint, version numbers).
- Secrets, hostnames of production databases, credentials of any kind.
- Review-only rules (severity, what not to flag) - those go in REVIEW.md.

## Shape

Keep it short and scannable. Use `@path/to/file.md` imports for detail that only some tasks need, or
better, a skill in `.claude/skills/` (loaded on demand). Add a subdirectory `CLAUDE.md` only when a
sub-project has genuinely different commands or rules; it is loaded when Claude works in that folder,
and CI review reads it when the PR touches files under it.

If one rule keeps being ignored, mark that single line `IMPORTANT:`. Emphasising many lines
emphasises none.

## Skeleton

Fill from Phase 1-3 findings. Delete any section that would be empty.

```markdown
# <Project name>

<One or two sentences: what this is and who uses it.>

## Commands
- Install: `<cmd>`
- Run locally: `<cmd>`
- Tests (all): `<cmd>`  - single test: `<cmd path::name>`
- Lint / format / typecheck: `<cmd>` / `<cmd>` / `<cmd>`
- Build: `<cmd>`

## Verification - do this before saying a change is done
1. `<typecheck/lint cmd>`
2. `<targeted test cmd>` for the code you touched, then `<full suite cmd>` if it runs in < N min
3. <UI/API check if relevant, e.g. "hit the endpoint with curl and show the response">
Show the commands you ran and their output. Do not claim success without evidence.
<Note anything that cannot be verified locally, e.g. "integration tests need docker compose up db">

## Architecture
- <non-obvious decision 1>
- <non-obvious decision 2>

## Conventions
- <rule that differs from the default, with the canonical example file>

## Gotchas
- <thing that bit someone before>

## Sensitive areas - changes here need the owner's review (see CODEOWNERS)
- `<path>` - <why: auth / billing / migrations / CI / release>

## Workflow
- Branches: `<pattern>`. Commits: `<style>`.
- Before opening a PR run `/pre-pr`. Fill every section of the PR template.
- Review rules for PRs live in REVIEW.md.
- When compacting, keep the list of modified files and the verification commands.
```
