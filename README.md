# codex-flow

Claude + Codex dual-model development, layered on [superpowers](https://github.com/obra/superpowers) **without forking it**. Claude orchestrates and reviews; Codex (`gpt-5.5`) is the independent adversarial reviewer and the fast implementer. The skills call the superpowers skills as sub-skills and weave Codex in at four seams.

## Requirements

- The **superpowers** plugin installed — these skills call `superpowers:brainstorming`, `superpowers:writing-plans`, `superpowers:subagent-driven-development`.
- The **OpenAI Codex CLI** installed and authenticated — `command -v codex`, `codex login`. Default model `gpt-5.5`. Each skill degrades to plain superpowers if Codex is absent.

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
| `writing-plans-codex` | `superpowers:writing-plans` + Codex adversarial **plan** critique (autofix) → chains to `sdd-with-codex-implementer` |
| `sdd-with-codex-implementer` | the superpowers SDD loop with Codex as the per-task **implementer** (fast/low) + Claude reviewer kept + a final Codex **challenge** on the branch diff |
| `codex-lanes` | shared reference: the exact `codex exec` invocation — guards, fast/xhigh lanes, result schema, prompts. Loaded by the other three. |

## Kick off

> "brainstorm \<feature\> with the codex flow"

`brainstorming-codex` runs; the chain carries through plan → implementation, with human approval gates between phases.

## Model lanes

| lane | model | effort | fast_mode | sandbox |
|---|---|---|---|---|
| implementer | `gpt-5.5` | low | on (1.5× speed, 2.5× credits) | workspace-write |
| review | `gpt-5.5` | xhigh | off | read-only (doc autofix: workspace-write) |

Built and verified with `superpowers:writing-skills` (RED→GREEN→REFACTOR).
