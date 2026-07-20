---
name: writing-plans-codex
description: Use when turning an approved spec into an implementation plan in the Claude+Codex flow. Triggers - invoked by brainstorming-codex, "write the plan with codex", "plan with codex review", or when a spec is approved and you want Codex to adversarially review the plan before execution. Optional arg `--fast` - all codex exec calls run in fast mode (codex-lanes §3), carried into the sdd hand-off.
---

# Writing Plans (Codex-augmented)

## Overview

`superpowers:writing-plans` produces the plan; a different model (Codex) attacks it before
execution. Claude plans, Codex critiques — **no human gate here.** The only approval in the
whole flow is the design doc back in `brainstorming-codex`; once that's approved, planning and
execution run autonomously. Chains into `sdd-with-codex-implementer`. Superpowers is called,
never forked.

**REQUIRED SUB-SKILL: `codex-flow:codex-lanes`** — load it for the exact `codex exec` invocation
(guards + review lane).

## When to Use

- an approved spec exists and you need the implementation plan, with a second-model critique before execution
- NOT: no spec yet (use `brainstorming-codex`) · Codex CLI absent (→ plain `superpowers:writing-plans`)

## Flow (do not reorder)

1. **Codex availability** — check `codex-flow:codex-lanes` §1. `NO_CODEX` → run plain
   `superpowers:writing-plans`, tell the user once, stop here.
2. **REQUIRED SUB-SKILL: superpowers:writing-plans** — produce and save the plan. High-taste,
   main-loop stage — if the session runs on sonnet, suggest `/model opus` once first. When the plan
   is saved, **control returns HERE.** Do NOT present writing-plans' execution-choice menu, and
   do NOT launch plain `superpowers:subagent-driven-development` — this skill owns the hand-off
   (step 5).
3. **Seam 2 — Codex plan critique** — run the **review lane** (`codex-flow:codex-lanes` §4, doc
   autofix, `workspace-write`) against the plan file: task ordering, missing rollback steps,
   untestable acceptance criteria, cross-package sequencing, tasks that span files unsafely.
   Codex critiques and revises the plan in place.
4. **No gate — autonomous.** Note the hardened plan and a one-line summary of what Codex changed
   for the record, then proceed. Do NOT pause for approval; the design doc was already gated.
5. **Hand off — REQUIRED SUB-SKILL: sdd-with-codex-implementer** (NOT plain
   `superpowers:subagent-driven-development` — that path drops the Codex implementer lane and the
   final cross-model challenge). Invoked with `--fast` → pass it along; fast mode is flow-wide
   (codex-lanes §3).

## Order contract

The Codex critique fires AFTER the plan is saved and BEFORE execution. Never start executing
tasks until the plan has cleared Seam 2 — but Seam 2 is autonomous, no human gate.

## Common Mistakes (from baseline testing — every one was observed)

| Baseline behavior | Do instead |
|---|---|
| bare `codex exec "Read <plan>…"` — hangs, no xhigh, no repo root | use the review-lane recipe in `codex-flow:codex-lanes` verbatim |
| picked writing-plans' "Subagent-Driven (recommended)" menu → plain SDD | chain to `sdd-with-codex-implementer` |
| Claude "incorporated the feedback" itself | doc autofix: Codex revises the plan file in place (`workspace-write`) |
| started executing before the plan critique | critique first, then execute — no gate |
| paused for user approval of the plan | autonomous after the design-doc gate; don't re-gate |
