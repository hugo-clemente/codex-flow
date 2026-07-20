---
name: codex-lanes
description: Reference recipe for invoking the Codex CLI (`codex exec`) from the codex-flow skills. Use when composing any codex exec call - it carries the mandatory guards, the implementer (terra/medium) and review (sol/xhigh) lanes, the result schema, and the adversarial prompts. Loaded by brainstorming-codex, writing-plans-codex, and sdd-with-codex-implementer.
---

# Codex `codex exec` lanes (shared recipe)

Shared invocation recipe for the codex-flow skills. Read this before composing any
`codex exec` call. The guards below are not optional; each maps to a real Codex-CLI failure.

## 1. Availability gate (run once, before any Codex step)

```bash
command -v codex >/dev/null 2>&1 && echo HAVE_CODEX || echo NO_CODEX
```

`NO_CODEX` → skip the Codex seam, run the plain `superpowers:*` skill instead, tell the user
once ("Codex CLI not found — running plain superpowers"). Never fabricate Codex output or pass
off a Claude self-review as the Codex pass.

## 2. Guards — on EVERY `codex exec` call

- **stdin must be fed or closed.** Either pipe the prompt in (`- < prompt.md`) OR add
  `< /dev/null`. Never leave stdin open — in a non-TTY (the Claude Code Bash tool) Codex hangs
  forever waiting for EOF. The #1 failure. Pick exactly one per call:
  - short prompt → positional arg + `< /dev/null`
  - long/structured prompt → `- < prompt.md` (the file IS stdin; do NOT also add `</dev/null`)
- **`-C "$_REPO_ROOT"`** — resolve eagerly: `_REPO_ROOT="$(git rev-parse --show-toplevel 2>/dev/null || pwd)"`.
- **`-m` — pin the model per lane (§3).** The two lanes use different models; never rely on the config-default model.
- **`--skip-git-repo-check`** — always pass it. No-op inside a git repo; lets codex run when the cwd / `-C` dir isn't a trusted git repo (e.g. reviewing a spec/plan doc under `~/Documents/claude-plans`, or running outside any repo). Without it codex errors "Not inside a trusted directory".
- **effort is explicit.** `~/.codex/config.toml` defaults to `high`; an unset effort is neither
  the fast lane nor the xhigh lane. Always pass `-c 'model_reasoning_effort="…"'`.
- **bound the run.** Prefix `gtimeout 3600` (macOS: `brew install coreutils`; Linux: `timeout 3600`) —
  a generous hang-guard, not a work limit. Omit only if neither binary exists.
- **long runs go in the background.** Claude Code's foreground Bash tool hard-caps at 600s (10 min); xhigh
  reviews and implementer tasks routinely exceed that. Launch codex with the Bash tool's
  `run_in_background: true` and read the output when the process exits — never a plain foreground codex
  call. `gtimeout` is only the stuck-process backstop.

## 3. Lanes

| lane            | when                          | model         | effort | sandbox          |
|-----------------|-------------------------------|---------------|--------|------------------|
| **implementer** | writing code (SDD task)       | gpt-5.6-terra | medium | workspace-write  |
| **review**      | critiquing a spec/plan/diff   | gpt-5.6-sol   | xhigh  | read-only*       |

\*doc autofix (rewriting a spec/plan in place) uses `workspace-write`; code-diff review stays `read-only`.
5.6 adds `max` and `ultra` above xhigh — explicit escalation for the review lane when xhigh
comes back shallow; never the default.

**Fast mode (opt-in, flow-wide).** Any codex-flow skill invoked with `--fast` (or the user asks
for fast mode) → add `-c 'service_tier="fast"' -c 'features.fast_mode=true'` to **every**
`codex exec` call, both lanes, reviews included. ~1.5× speed at ~2.5× credits. The flag is
sticky: carry it through every hand-off (brainstorming → writing-plans → sdd) until the flow
ends or the user turns it off. Never the default.

## 3b. Claude lanes (Agent-tool dispatches)

The Codex lanes above cover gpt-5.6; Claude subagents need a policy too — never let them
silently inherit the session model:

| lane                                  | model    | effort | why                                        |
|---------------------------------------|----------|--------|--------------------------------------------|
| per-task reviewer (two verdicts)      | `opus`   | —      | real review needs intelligence             |
| final whole-branch code-reviewer      | `opus`   | —      | one shot, highest stakes                   |
| taste-gated implementer (user-facing) | `sonnet` | —      | taste meets the user-facing bar            |
| escalated implementer (Codex missed)  | `opus`   | —      | the miss was quality — go up in intelligence |
| mechanical chores (briefs, ledger)    | `sonnet` | low    | clear-spec transcription                   |

**Defaults, not limits.** Judge the output, not the price tag: if a cheaper lane's output misses
the bar, rerun on a smarter model without asking. When axes conflict on anything that ships:
intelligence > taste > cost. Cost is a tie-breaker only.

Inside `Agent`/`Workflow` fan-outs the model param takes Claude models only — to reach the Codex lanes
there, wrap it: a `sonnet`/low agent whose prompt says "compose a self-contained codex prompt,
run the lane's `codex exec` via Bash, return the parsed `-o` JSON".

## 4. Review lane — invocation (xhigh, standard tier)

xhigh reviews routinely run past 10 min, so run this in the **background** — never a plain foreground
call (Claude Code's foreground Bash tool hard-caps at 600s). `-o` isolates the final findings message;
the full transcript (startup banner, reasoning, echoed tool output, token footer) goes to `$LOG` and is
read only on failure — never fold it into context:

```bash
_REPO_ROOT="$(git rev-parse --show-toplevel 2>/dev/null || pwd)"
OUT="$(mktemp)"; LOG="$(mktemp)"
gtimeout 3600 codex exec -m gpt-5.6-sol \
  -c 'model_reasoning_effort="xhigh"' \
  -C "$_REPO_ROOT" -s read-only --skip-git-repo-check \
  -o "$OUT" \
  "<adversarial prompt — §6>  Target: <path or 'the working diff'>." \
  < /dev/null > "$LOG" 2>&1
```

Fast mode (§3): insert `-c 'service_tier="fast"' -c 'features.fast_mode=true'` after the effort line.
Launch that with the Bash tool's `run_in_background: true`; when the process exits (the harness notifies
you), read **`$OUT`** for the findings. `$OUT` empty or the run errored → consult `$LOG`. Doc autofix
(spec/plan): swap `-s read-only` → `-s workspace-write` and end the prompt with "…then rewrite the file
in place, resolving the material findings." (`-o` then holds just the change summary — the value is the
rewritten file.)

## 5. Implementer lane — invocation (terra medium, structured result)

The result schema is not shipped on disk — write it to a throwaway file first:

```bash
_REPO_ROOT="$(git rev-parse --show-toplevel 2>/dev/null || pwd)"
SCHEMA="$(mktemp)"; cat > "$SCHEMA" <<'JSON'
{ "type":"object","additionalProperties":false,
  "required":["status","files_modified","issues","summary","verification_summary"],
  "properties":{
    "status":{"enum":["completed","partial","failed"]},
    "files_modified":{"type":"array","items":{"type":"string"}},
    "issues":{"type":"array","items":{"type":"string"}},
    "summary":{"type":"string"},
    "verification_summary":{"type":"string"}}}
JSON
gtimeout 3600 codex exec -m gpt-5.6-terra \
  -c 'model_reasoning_effort="medium"' \
  -C "$_REPO_ROOT" -s workspace-write --skip-git-repo-check \
  --output-schema "$SCHEMA" \
  -o "$(mktemp -u).json" \
  - < "$PROMPT_FILE"
```

(`- < "$PROMPT_FILE"` feeds the prompt via stdin, so no `< /dev/null` here.) Launch with the
Bash tool's `run_in_background: true` and poll for the `-o` result file — see
`sdd-with-codex-implementer`.

Fast mode (§3): insert `-c 'service_tier="fast"' -c 'features.fast_mode=true'` after the
effort line.

## 6. Prompts

- **Spec / plan adversarial review:** "You are an adversarial reviewer. Challenge every
  assumption in <file>. Surface unstated assumptions, missing edge cases, failure modes, and
  anything that contradicts this repo's conventions. Findings only, ranked by severity — no
  praise. [autofix variant: …then rewrite <file> in place resolving the material ones.]"
- **Final code challenge (branch diff):** "Assume this diff is broken. Find the strongest reasons
  it should not ship: auth, data loss, races, rollback, empty-state, schema drift. Ground every
  finding in a file:line. Findings only."
