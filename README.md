# codex-flow

Claude + Codex dual-model development, layered on [superpowers](https://github.com/obra/superpowers) **without forking it**. Claude orchestrates and reviews; Codex (gpt-5.6: `sol`/`terra`) is the independent adversarial reviewer and the fast implementer. The skills call the superpowers skills as sub-skills and weave Codex in at four seams.

## Requirements

- The **superpowers** plugin installed — these skills call `superpowers:brainstorming`, `superpowers:writing-plans`, `superpowers:subagent-driven-development`.
- The **OpenAI Codex CLI** installed and authenticated — `command -v codex`, `codex login`. Models `gpt-5.6-sol` / `gpt-5.6-terra`. Each skill degrades to plain superpowers if Codex is absent.

## Install

```
/plugin marketplace add hugo-clemente/codex-flow
/plugin install codex-flow@codex-flow
```

(Private repo — fetched with your `gh` / git credentials.)

## Skills

| skill | role |
|---|---|
| `brainstorming-codex` | `superpowers:brainstorming` + Codex adversarial **spec** critique → chains to `writing-plans-codex` |
| `writing-plans-codex` | `superpowers:writing-plans` + Codex adversarial **plan** critique (findings JSON, Claude adjudicates) → chains to `sdd-with-codex-implementer` |
| `sdd-with-codex-implementer` | the superpowers SDD loop with Codex as the per-task **implementer** (terra/low) + Claude reviewer kept + a final Codex **challenge** on the branch diff |
| `pr-triage-codex` | red PR → green: triage review comments + CI blockers with the user (batched AskUserQuestion, ELI5 per item), Codex implements the fixes + challenges the diff, push, watch CI until green |
| `codex-lanes` | shared reference: the exact `codex exec` invocation — guards, terra/sol lanes, result schema, prompts. Loaded by the others. |

## Kick off

> "brainstorm \<feature\> with the codex flow"

`brainstorming-codex` runs; the chain carries through plan → implementation, with human approval gates between phases.

## Model lanes

| lane | model | effort | sandbox |
|---|---|---|---|
| implementer | `gpt-5.6-terra` | low | workspace-write |
| review | `gpt-5.6-sol` | xhigh (escalate max/ultra) | read-only — findings JSON, Claude adjudicates + applies |

Fast mode: pass `--fast` to any of the three flow skills → every `codex exec` call in the whole
flow (reviews included) runs with `service_tier="fast"` + `features.fast_mode=true`
(~1.5× speed, ~2.5× credits). Sticky across hand-offs, opt-in.

Claude lanes (Agent dispatches, see `codex-lanes` §3b): reviewers (per-task + final) `opus` ·
user-facing implementer `sonnet` · escalated implementer `opus` · mechanical chores
`sonnet`/low. Defaults, not limits — bad output gets rerun on a smarter model, no asking.

Built and verified with `superpowers:writing-skills` (RED→GREEN→REFACTOR).
