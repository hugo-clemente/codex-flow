---
name: pr-triage-codex
description: Use when a PR has review comments, bot findings, or failing CI and the goal is to get it fully green. Triages every open item with the user in batched AskUserQuestion calls, then Codex implements the accepted fixes and adversarially reviews the diff, then pushes and watches CI until green. Triggers - "triage my PR", "get CI green", "handle the review comments", "fix the PR feedback", "clear the CI blockers", a PR URL plus "make it pass". Optional arg `--fast` - all codex exec calls run in fast mode (codex-lanes §3).
---

# PR Triage (Codex-augmented)

## Overview

One pass from red PR to green PR: gather every open item (unresolved review threads, bot
findings, failing checks) → triage them with the user in a few batched AskUserQuestion calls →
Codex implements the accepted fixes → Codex adversarially reviews the diff → reply/resolve
threads, push, watch CI → loop on new failures until green or blocked.

**REQUIRED SUB-SKILL: `codex-flow:codex-lanes`** — implementer lane (§5) writes fixes, review
lane (§4) challenges the diff. `NO_CODEX` (§1) → Claude subagents take both roles; say so once.

## When to Use

- a PR exists and has unresolved comments, bot findings, or red checks
- NOT: no PR yet (→ `superpowers:finishing-a-development-branch`) · deep feature work hiding
  behind one failing check (→ fix it as its own task, this skill is for triage breadth)

## Flow

### 1. Gather (no questions yet)

- PR: `gh pr view --json number,url,headRefName,baseRefName`. Not on the PR branch → check it out.
- Unresolved review threads (REST doesn't expose resolution — GraphQL only):
  ```bash
  gh api graphql -f query='query($o:String!,$r:String!,$n:Int!){repository(owner:$o,name:$r){
    pullRequest(number:$n){reviewThreads(first:100){nodes{id isResolved isOutdated path line
    comments(first:10){nodes{author{login} body url}}}}}}' \
    -f o=<owner> -f r=<repo> -F n=<pr#> \
    --jq '.data.repository.pullRequest.reviewThreads.nodes[] | select(.isResolved==false)'
  ```
  Humans and bots (CodeRabbit, Copilot, linters) both count — one triage list.
- Failing checks: `gh pr checks <pr#>` → for each failure `gh run view <run-id> --log-failed`
  (read enough log to name the actual error, not the step name).
- Build ONE numbered triage list: `#, source (reviewer/bot/CI), file:line or check name,
  one-line gist, your read (legit / disputed / flaky / needs-decision)`. Show it to the user
  as a table before asking anything — the questions reference these numbers.
- Under the table, explain each item **ELI5**: 2-3 plain sentences per item — what the comment
  or failure actually means, why it matters (or doesn't), and what fixing it would involve.
  No jargon from the log, no reviewer shorthand. The user decides every item's fate in step 2;
  they can only decide well if they understood the item without opening the PR. The question
  text then stays short — it references the explanation, not repeats it.

### 2. Triage — batched AskUserQuestion

The whole point is killing ping-pong: **batch up to 4 items per AskUserQuestion call**, never
one question per message. Each item is one question — header = `#N <short label>`, question text
= the gist + your recommendation. Options:

- **Fix** — accept, goes to the implementer queue
- **Won't fix — reply** — you draft the pushback from the user's stated reason, post it, leave
  the thread open for the reviewer
- **Rerun (flaky)** — CI-only: `gh run rerun <id> --failed`, no code change
- **Defer** — acknowledge in a reply, out of scope for this pass

Obvious mechanical CI breakage (lint, format, a typo the log names exactly) → put it in the
list marked "auto-fix" and fold it into ONE confirmation question ("auto-fix items #2 #5 #7?")
instead of one question each. Judgment calls always get their own question.

### 3. Fix — Codex implementer lane

Cluster accepted items by file/theme; ONE `codex exec` implementer call (`codex-lanes` §5) per
cluster, not per comment. Prompt = the cluster's items verbatim (comment body, file:line, CI
error) + the repo's REAL verify commands. Taste gate applies (see `sdd-with-codex-implementer`):
UI/copy/public-API items → Claude subagent (`model: sonnet`) instead. `--fast` → fast-mode flags
on every codex call. After all clusters: run the repo's lint + tests locally — Codex's sandbox
self-report is not verification.

### 4. Challenge — Codex review lane

One review-lane pass (`codex-lanes` §4, read-only) on the accumulated diff:
`Target: the diff from <merge-base> to HEAD`. This is the second model checking the fixes
actually address the comments without breaking anything else. Adjudicate findings; material
ones → back to step 3 for that cluster.

### 5. Close the loop on GitHub

- Fixed items: reply on each thread ("fixed in <sha> — <one line>") then resolve it:
  `gh api graphql -f query='mutation($t:ID!){resolveReviewThread(input:{threadId:$t}){thread{id}}}' -f t=<threadId>`
- Won't-fix / deferred: post the drafted reply, do NOT resolve — the reviewer closes it.
- Commit (message references what was addressed, not ticket numbers) + push.

### 6. Watch CI, loop

`gh pr checks <pr#> --watch --fail-fast` in the **background** (`run_in_background: true` — the
foreground Bash cap is 600s and CI is slower). On exit:

- all green → report: items fixed / replied / deferred, checks green, done.
- new failures → gather ONLY the new items, back to step 2. **Max 3 rounds** — a third red round
  means something structural; stop and report instead of churning.
- blocked (missing secret, infra, required reviewer) → name the blocker, stop.

## Common Mistakes

| Baseline behavior | Do instead |
|---|---|
| one AskUserQuestion per comment — 15 round-trips | batch 4 per call; auto-fix obvious CI items in one confirmation |
| REST comment list treated as triage list — resolution state invisible | GraphQL `reviewThreads` with `isResolved==false` |
| resolved threads without replying | reply with the sha, THEN resolve; never resolve won't-fix |
| foreground `gh pr checks --watch` — killed at 600s | background launch, read result on exit |
| trusted Codex "tests pass" self-report | re-run the repo's real commands locally |
| bare `codex exec "fix the comments"` | codex-lanes recipes verbatim, one call per cluster |
| endless fix-push-fail loop | 3 rounds max, then stop and report |
