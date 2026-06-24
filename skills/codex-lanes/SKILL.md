---
name: codex-lanes
description: Reference recipe for invoking the Codex CLI (`codex exec`) from the codex-flow skills. Use when composing any codex exec call - it carries the mandatory guards, the implementer (fast/low) and review (xhigh) lanes, the result schema, and the adversarial prompts. Loaded by brainstorming-codex, writing-plans-codex, and sdd-with-codex-implementer.
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
- **`-m gpt-5.5`** — pin the model.
- **`--skip-git-repo-check`** — always pass it. No-op inside a git repo; lets codex run when the cwd / `-C` dir isn't a trusted git repo (e.g. reviewing a spec/plan doc under `~/Documents/claude-plans`, or running outside any repo). Without it codex errors "Not inside a trusted directory".
- **effort is explicit.** `~/.codex/config.toml` defaults to `high`; an unset effort is neither
  the fast lane nor the xhigh lane. Always pass `-c 'model_reasoning_effort="…"'`.
- **bound the run.** Prefix `gtimeout 600` (macOS: `brew install coreutils`; Linux: `timeout 600`).
  Omit only if neither binary exists.

## 3. Lanes

| lane            | when                          | model   | effort | fast_mode | sandbox          |
|-----------------|-------------------------------|---------|--------|-----------|------------------|
| **implementer** | writing code (SDD task)       | gpt-5.5 | low    | on        | workspace-write  |
| **review**      | critiquing a spec/plan/diff   | gpt-5.5 | xhigh  | off       | read-only*       |

\*doc autofix (rewriting a spec/plan in place) uses `workspace-write`; code-diff review stays `read-only`.
`fast_mode` = 1.5× speed at 2.5× credits — implementer lane ONLY, never on reviews.

## 4. Review lane — invocation (xhigh, standard tier)

```bash
_REPO_ROOT="$(git rev-parse --show-toplevel 2>/dev/null || pwd)"
gtimeout 600 codex exec -m gpt-5.5 \
  -c 'model_reasoning_effort="xhigh"' \
  -C "$_REPO_ROOT" -s read-only --skip-git-repo-check \
  "<adversarial prompt — §7>  Target: <path or 'the working diff'>." \
  < /dev/null
```

Doc autofix (spec/plan): swap `-s read-only` → `-s workspace-write` and end the prompt with
"…then rewrite the file in place, resolving the material findings."

## 5. Implementer lane — invocation (low + fast, structured result)

The result schema (§6) is not shipped on disk — write it to a throwaway file first:

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
gtimeout 600 codex exec -m gpt-5.5 \
  -c 'model_reasoning_effort="low"' \
  -c 'service_tier="fast"' -c 'features.fast_mode=true' \
  -C "$_REPO_ROOT" -s workspace-write --skip-git-repo-check \
  --output-schema "$SCHEMA" \
  -o "$(mktemp -u).json" \
  - < "$PROMPT_FILE"
```

(`- < "$PROMPT_FILE"` feeds the prompt via stdin, so no `< /dev/null` here.) Launch with the
Bash tool's `run_in_background: true` and poll for the `-o` result file — see
`sdd-with-codex-implementer`.

## 6. Result schema (implementer) — content for §5's `$SCHEMA`

```json
{ "status": "completed|partial|failed",
  "files_modified": ["..."],
  "issues": ["..."],
  "summary": "...",
  "verification_summary": "..." }
```

## 7. Prompts

- **Spec / plan adversarial review:** "You are an adversarial reviewer. Challenge every
  assumption in <file>. Surface unstated assumptions, missing edge cases, failure modes, and
  anything that contradicts this repo's conventions. Findings only, ranked by severity — no
  praise. [autofix variant: …then rewrite <file> in place resolving the material ones.]"
- **Final code challenge (`--base`):** "Assume this diff is broken. Find the strongest reasons
  it should not ship: auth, data loss, races, rollback, empty-state, schema drift. Ground every
  finding in a file:line. Findings only."
