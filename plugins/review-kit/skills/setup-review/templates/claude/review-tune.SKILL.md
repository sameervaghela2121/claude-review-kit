---
name: review-tune
description: Tune the AI reviewer for this repository from evidence - classify how humans handled recent Claude review comments and record misses (reverts, incidents), then propose edits to REVIEW.md, CLAUDE.md, lint rules or hooks. Run every couple of weeks or after an incident.
disable-model-invocation: true
argument-hint: "[number-of-merged-PRs, default 20] | incident <PR-or-commit>"
---

# Tune the reviewer

Arguments: `$ARGUMENTS`.

The goal is fewer human minutes per change **without** more reverts or incidents. Find real bugs first;
cut review noise second.

## Mode A - incident: `incident <PR number or commit>`

A bug reached main or production. Find out which layer should have caught it.

1. Read the change that introduced the bug (`gh pr view <n> --json title,body,files,comments,reviews`,
   `gh pr diff <n>`, or `git show <commit>`) and the fix or revert.
2. Identify the exact `file:line` of the bug and the class of bug.
3. Decide which layer should have caught it: CLAUDE.md verification, a test, a lint rule, a hook,
   REVIEW.md "Always check", security review, or human review of a High-risk path.
4. Append one entry to `docs/review-incidents.md` (create it if missing):
   `| date | PR | file:line | bug class | layer that should have caught it | change made |`.
   This file is the reviewer's evaluation set - every reviewer change is judged against it.
5. Propose the smallest change to that layer. Prefer a deterministic check (test, lint rule, hook)
   over another sentence in REVIEW.md.

## Mode B - noise and miss review (default)

1. List the last N merged PRs (default 20):
   `gh pr list --state merged --limit N --json number,title,mergedAt`.
2. For each, fetch Claude's review comments:
   `gh api repos/{owner}/{repo}/pulls/<n>/comments --paginate` and the PR conversation comments;
   keep the ones authored by the Claude app or bot.
3. Classify each comment by what the humans did:
   - **fixed** - a later commit changed the flagged lines,
   - **rejected** - a reply disagreed, or the thread was resolved with no change,
   - **intentional** - a reply explained the code is deliberate,
   - **didn't matter** - ignored.
4. Also list reverts merged in the same period (`git log --oneline -i --grep=revert`).
5. Report a small table: counts per class, the top 3 noisy finding types (rejected / didn't matter),
   and any revert whose bug the reviewer did not flag.
6. Propose edits, each with the evidence that motivated it:
   - noisy finding type -> add it to REVIEW.md "Do not report", or narrow "What Important means here";
   - repeated real finding -> move it left: a lint rule, a test, a hook, or a CLAUDE.md rule, so it
     never reaches review again;
   - a miss -> an "Always check" rule in REVIEW.md, plus a Mode A entry.
7. Keep REVIEW.md short. If you add a rule, look for one to remove.

Show the proposed diff. Don't apply it unless the developer agrees. Changes to REVIEW.md take effect
in CI only after they are merged to the default branch.
