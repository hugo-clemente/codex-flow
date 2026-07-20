---
name: brainstorming-codex
description: Use when starting a new feature or change you want taken through the Claude+Codex flow, beginning with brainstorming. Triggers - "brainstorm with codex", "use our codex flow", "dual-model brainstorm", "codex flow", or any feature kickoff where Codex should independently review the spec before planning. Optional arg `--fast` - all codex exec calls in the whole flow run in fast mode (codex-lanes §3), carried through hand-offs.
---

# Brainstorming (Codex-augmented)

## Overview

`superpowers:brainstorming` produces the spec; then a different model (Codex) attacks that spec
before any planning; then you approve the hardened result. Claude designs, Codex critiques, you
gate. Superpowers is called, never forked.

**This human gate (step 4) is the ONLY approval in the entire Claude+Codex flow.** Once the
design doc is approved here, planning (`writing-plans-codex`) and execution
(`sdd-with-codex-implementer`) run autonomously to the end of development — no further gates.

**REQUIRED SUB-SKILL: `codex-flow:codex-lanes`** — load it for the exact `codex exec` invocation
(guards + review lane). Every Codex call below uses it.

## When to Use

- kicking off a feature/change and you want an independent second-model critique of the spec
- NOT: trivial changes · mid-implementation · when the Codex CLI is absent (→ plain superpowers)

## Flow (do not reorder)

1. **Codex availability** — check `codex-flow:codex-lanes` §1. `NO_CODEX` → run plain
   `superpowers:brainstorming`, tell the user once, stop here.
2. **REQUIRED SUB-SKILL: superpowers:brainstorming** — run the full design dialogue and write +
   commit the spec. This is a high-taste, main-loop stage — the skill can't pin the model; if the
   session runs on sonnet, suggest `/model opus` once before starting. When the spec is committed, **control returns HERE.** Do NOT follow
   brainstorming's built-in hand-off into writing-plans — this skill owns the hand-off (step 5).
   Use whatever spec path brainstorming chose; don't relocate it.
3. **Seam 1 — Codex spec critique** — run the **review lane** (`codex-flow:codex-lanes` §4, doc
   autofix, `workspace-write`) against the committed spec file. Codex critiques and revises the
   spec in place.
4. **Human gate (the only one)** — show the user the hardened spec plus a one-line summary of
   what Codex changed; get approval. Changes requested → revise → re-gate. This is the sole
   approval in the flow; after it, planning and execution are autonomous.
5. **Hand off — REQUIRED SUB-SKILL: writing-plans-codex** (NOT plain
   `superpowers:writing-plans` — that path silently drops the plan-stage Codex review).
   Invoked with `--fast` → pass it along; fast mode is flow-wide (codex-lanes §3).

## Order contract

The Codex critique fires AFTER the spec is committed and BEFORE any planning. Never enter
planning until the spec has been through Seam 1 and the human gate.

## Common Mistakes (from baseline testing — every one was observed)

| Baseline behavior | Do instead |
|---|---|
| bare `codex exec "Read <file>…"` — hangs, no xhigh, no repo root | use the review-lane recipe in `codex-flow:codex-lanes` verbatim |
| chained to plain `superpowers:writing-plans` | chain to `writing-plans-codex` |
| ran writing-plans before the Codex critique | critique + gate first, then plan |
| dropped the user approval gate | keep the human gate (step 4) |
| invented a spec path / Claude "self-reviewed" as the Codex pass | use brainstorming's spec path; the critique is a real `codex exec` call |
