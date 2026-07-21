---
name: sdd-with-codex-implementer
description: Use when executing an implementation plan task-by-task in the current session with Codex writing the code. Triggers - invoked by writing-plans-codex, "run the plan with codex", "sdd with codex implementer", "execute the plan, codex implements", or when an approved plan is ready and Codex should do the per-task implementation while Claude reviews. Optional arg `--fast` - all codex exec calls (implementer + final challenge) run in Codex fast mode (codex-lanes §3).
---

# Subagent-Driven Development — Codex Implementer

## Overview

The `superpowers:subagent-driven-development` loop with ONE substitution: each task's IMPLEMENTER
is a `codex exec` call (terra/low lane), not a Claude subagent. Claude stays controller + reviewer.
Everything else in SDD is unchanged.

**REQUIRED SUB-SKILL: `codex-flow:codex-lanes`** — load it for the implementer-lane invocation.

## REQUIRED BACKGROUND

You MUST follow `superpowers:subagent-driven-development` for the whole loop — controller duties,
`task-brief` / `review-package` file handoffs, the per-task **two-verdict review (spec compliance
+ code quality)**, the ledger, the final whole-branch review, and
`superpowers:finishing-a-development-branch`. This skill overrides ONLY the implementer dispatch
and adds a final cross-model challenge.

## When to Use

- executing an approved plan, Codex implementing each task, Claude reviewing
- NOT: tightly-coupled tasks needing one mind · Codex CLI absent (→ plain `superpowers:subagent-driven-development`)

## What changes vs base SDD

| step | base SDD | here |
|---|---|---|
| implementer | Claude subagent | `codex exec` implementer lane (`codex-flow:codex-lanes` §5); **taste-gated tasks stay Claude** (see below) |
| task reviewer | Claude subagent, two verdicts | **unchanged** flow — pin `model: opus` (`codex-flow:codex-lanes` §3b) |
| final review | Claude code-reviewer | pin `model: opus` **+ Codex challenge (Seam 4)** |
| everything else | — | unchanged |

## Taste gate — route BEFORE dispatching

terra transcribes fast but its taste is below the bar for anything user-facing. Per task,
before step 1 below: does the task touch UI/UX, user-visible copy, or public API shape?
→ Claude implementer subagent (base SDD dispatch, `model: sonnet`), skip the Codex lane for
that task. Mechanical tasks (clear-spec implementation, migrations, plumbing) → Codex lane.

## Per-task implementer dispatch (replaces base SDD's implementer step)

1. Record BASE: `git rev-parse HEAD`.
2. `scripts/task-brief PLAN N` → brief file (base SDD).
3. Build the Codex prompt file: START with the fixed preamble from this skill's
   `codex-implementer-prompt.md` (verbatim — it carries the working rules, the no-questions
   status mapping, and the self-review pass), THEN the XML sections from the brief —
   `<task><files><patterns><approach><constraints><testing><verify><output_contract>`. In `<verify>`, give Codex **THIS repo's real commands** — `pnpm --filter @roundtable/<pkg> typecheck` and `pnpm --filter @roundtable/<pkg> test:no-migrate --run <files>` — never a generic `bun test`/`npm test` (baseline Codex guessed `bun test` in a pnpm/vitest repo).
4. Run the **implementer lane** (`codex-flow:codex-lanes` §5): background launch (`run_in_background: true`) + poll the `-o` result file. Invoked with `--fast` (or user asked for fast mode) → add the fast-mode flags (§3) to every `codex exec` call, Seam 4 included.
5. Read the result JSON; map `status` to base SDD's handling:
   - `completed` → proceed to review
   - `partial` → keep the diff, finish locally, then review
   - `failed` / missing or malformed JSON → roll back to BASE, follow **Escalation** below
6. Codex's `status:completed` + `verification_summary` are the implementer's **self-report, not the
   review.** Continue to base SDD's task-reviewer (fresh Claude subagent, both verdicts, fed the
   `review-package` diff) exactly as normal, and re-run verification with the repo's pnpm commands
   — don't accept Codex's sandbox test run as proof.

## Seam 4 — final whole-branch challenge (after all tasks)

Base SDD's final review stays (Claude code-reviewer on the most capable model). ADD a Codex
challenge on the branch diff: **review lane** (`codex-flow:codex-lanes` §4) with the code-challenge
prompt and `Target: the diff from <merge-base> to HEAD` (plain `codex exec` has no `--base` flag —
name the range in the prompt; Codex runs `git diff` itself in the read-only sandbox). Two-model final
review; **Claude adjudicates and applies fixes autonomously** — no human gate. Summarize what both
reviews found and what you changed for the record, then proceed to
`superpowers:finishing-a-development-branch`.

## Escalation

Judge the output, not the price tag. A Codex task result that misses the bar (review rejects it,
`failed`, or the diff is off-spec): tighten the brief and re-dispatch **once**. Still short →
roll back to BASE and hand THAT task to a Claude implementer subagent (`model: opus`) —
**without asking the user**. Escalating costs less than shipping mediocre work. Never crank
Codex above `low`; a task needing more effort isn't clean transcription.

## Common Mistakes (from baseline testing)

| Baseline / risk | Do instead |
|---|---|
| used a Claude implementer subagent (the Codex seam never emerged) | implementer = `codex exec` implementer lane |
| Codex prompt said `bun test` / `npm test` | pass THIS repo's pnpm commands in `<verify>` |
| ad-hoc "read files + run tests" instead of the structured review | route the diff through base SDD's task-reviewer (both verdicts) |
| skipped the final Codex challenge | run Seam 4 on the branch diff before finishing |
