# Hooks and `.claude/settings.json`

CLAUDE.md is advice; hooks are guarantees. Use a hook for anything that must happen every time.
Project hooks live in `.claude/settings.json` (committed, shared with the team). Personal overrides
go in `.claude/settings.local.json` (git-ignored).

Hook scripts receive a JSON payload on stdin. For `Edit`/`Write` tools the edited path is
`.tool_input.file_path`. Exit code 0 = fine. On `PreToolUse`, exit code 2 blocks the tool call and
stderr is shown to Claude. On `PostToolUse`, exit 2 cannot undo anything; it only shows stderr to Claude.

Docs: https://code.claude.com/docs/en/hooks

## 1. Format on edit (PostToolUse) - recommended

Only include branches for formatters that exist in this repo (found in Phase 1 and run in Phase 2).
Use the repo's own invocation (`npx prettier`, `poetry run ruff`, `uv run ruff`, `dart format`, ...).
Never fail the hook: a formatter error must not block Claude.

`.claude/hooks/format-file.sh`:

```bash
#!/usr/bin/env bash
# PostToolUse hook: format the file Claude just edited. Never blocks.
file=$(jq -r '.tool_input.file_path // empty' 2>/dev/null)
[ -n "$file" ] && [ -f "$file" ] || exit 0
cd "$CLAUDE_PROJECT_DIR" || exit 0

case "$file" in
  *.py)                      uv run ruff format --quiet "$file" ;;      # example - use this repo's tool
  *.ts|*.tsx|*.js|*.jsx|*.json|*.css|*.md) npx --no-install prettier --log-level warn --write "$file" ;;
  *.dart)                    dart format "$file" >/dev/null ;;
  *.go)                      gofmt -w "$file" ;;
esac 2>/dev/null
exit 0
```

If `jq` is not installed on the team's machines, read the path with
`python3 -c 'import json,sys;print(json.load(sys.stdin).get("tool_input",{}).get("file_path",""))'`.

## 2. Protect paths (PreToolUse) - recommended when the repo has generated or secret files

Block edits to generated code, lockfiles edited by hand, and env/secret files.

`.claude/hooks/protect-paths.sh`:

This hook must not fail open: read the path with python3 (always present where Claude Code runs on
macOS/Linux dev machines) rather than jq, and resolve symlinks so `/tmp` vs `/private/tmp` style
aliases can't slip past the patterns.

```bash
#!/usr/bin/env bash
# PreToolUse hook: block edits to files that must not be hand-edited.
file=$(python3 -c 'import json,os,sys;p=json.load(sys.stdin).get("tool_input",{}).get("file_path","");print(os.path.realpath(p) if p else "")')
[ -n "$file" ] || exit 0
root=$(python3 -c 'import os,sys;print(os.path.realpath(sys.argv[1]))' "$CLAUDE_PROJECT_DIR")
rel="${file#"$root"/}"

case "$rel" in
  .env|.env.*|*.pem|*.p12|*.key)
    echo "Blocked: $rel holds secrets. Ask the developer to change it." >&2; exit 2 ;;
  # Generated code - regenerate instead (replace with this repo's paths and generator command)
  src/gen/*)
    echo "Blocked: $rel is generated. Run <generator command> instead." >&2; exit 2 ;;
esac
exit 0
```

Keep `.env.example`-style files editable if the repo uses them (put a `.env.example) exit 0 ;;`
branch first).

## 3. `.claude/settings.json`

Merge into any existing file; keep the user's existing keys.

```json
{
  "permissions": {
    "deny": [
      "Read(./.env)",
      "Read(./.env.*)",
      "Read(./**/*.pem)",
      "Read(./**/*.p12)",
      "Read(./**/*.key)"
    ]
  },
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          { "type": "command", "command": "\"$CLAUDE_PROJECT_DIR\"/.claude/hooks/format-file.sh" }
        ]
      }
    ],
    "PreToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          { "type": "command", "command": "\"$CLAUDE_PROJECT_DIR\"/.claude/hooks/protect-paths.sh" }
        ]
      }
    ]
  }
}
```

If the repo has `.env.example` (or similar) files that Claude should read, keep the deny rules
specific to real env files.

`Read(...)` deny rules stop the Read tool only; a shell command such as `cat .env` is not covered.
Add `"Bash(cat .env*)"`-style deny rules if you want, but say in the summary that this is
best-effort, and that real secrets should not live in the working tree at all.

## 4. Optional: a Stop hook that runs fast checks

Only when the project has a check that finishes in well under a minute (e.g. typecheck). A Stop hook
that exits 2 makes Claude keep working until the check passes. Skip it for slow suites - `/pre-pr`
covers those. Read the Stop section of the hooks docs (input field `stop_hook_active`) before writing
one, so it cannot loop forever.

## Verify

```bash
chmod +x .claude/hooks/*.sh
python3 -m json.tool .claude/settings.json > /dev/null && echo "settings ok"
echo '{"tool_input":{"file_path":"'"$PWD"'/<some source file>"}}' | CLAUDE_PROJECT_DIR=$PWD .claude/hooks/format-file.sh; echo "exit $?"
echo '{"tool_input":{"file_path":"'"$PWD"'/.env"}}' | CLAUDE_PROJECT_DIR=$PWD .claude/hooks/protect-paths.sh; echo "exit $? (expect 2)"
```
