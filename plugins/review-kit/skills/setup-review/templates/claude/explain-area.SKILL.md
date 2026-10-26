---
name: explain-area
description: Write an up-to-date explainer for one part of this codebase (a directory, module or feature) - how it works, how data flows, what to watch out for - so a developer can learn it before changing it. Use when someone asks how an area works or before working in unfamiliar code.
argument-hint: "<path or feature name>"
---

# Explain an area of the codebase

Target: `$ARGUMENTS`.

Code review used to be how people learned the codebase. Now most reading is done by Claude, so learn
an area by asking for an explainer of exactly that slice, right before you change it.

1. Use a subagent to read the target and the code it calls and is called by. Read the recent history:
   `git log --oneline -20 -- <path>`.
2. Write the explainer in the conversation (not a file, unless the developer asks) with:
   - **Purpose** - what this area is for, in two sentences.
   - **Map** - the 5-10 files that matter, one line each, as `path:line` links.
   - **Flow** - how a typical request/event/job moves through it. A small ASCII or Mermaid diagram
     when there is more than one hop.
   - **Data** - the main types/tables/collections it reads and writes.
   - **Invariants and gotchas** - what must stay true, what has broken before (from history), and
     relevant rules from CLAUDE.md and REVIEW.md.
   - **How to change it safely** - which tests cover it and how to run them.
3. Only describe what you read in the code. Mark guesses as guesses.
