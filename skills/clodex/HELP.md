`clodex@lukas-local` **v0.2.6** — per-iteration `iter-NNN-review.md` (human-readable rendering of codex.json)

## /clodex — Autonomous Plan → Ship → Review → Fix Loop

### What it does

Take a task from "described" to "codex-approved + PR open" without babysitting. Plans, branches, implements, ships a PR, then runs the codex adversarial-review code path (via `codex-companion.mjs`, since the slash command itself is no longer model-invocable as of codex plugin v1.0.4) in a loop, fixing findings each round.

**Halts on:**
- `approve` — codex passed cleanly
- `threshold-satisfied` — only findings at-or-below the threshold ceiling remain (acceptable findings)
- `max-iter-hit` — iteration cap reached; you decide whether to extend or merge as-is
- `stalled` / `codex-reproducible-hang` / `error` — codex unavailable or repeatedly failing
- `no-diff` — nothing for codex to review

**Never:** auto-merges the PR, applies database migrations, force-pushes, invokes codex outside the sanctioned `codex-companion.mjs` adversarial-review path, writes to plugin internal state, or improvises an alternative path when a step fails.

### Usage

```
/clodex <task description> [thinking-level] [flags]
/clodex --resume
/clodex --continue [override flags]
/clodex --help | --about | -h
/clodex --version | -v | -V
```

### Flags

| Flag | Default | Description |
|---|---|---|
| `--max-iter N` | `5` | iteration cap for the review→fix loop |
| `--threshold X` | `medium` | highest **acceptable** severity: `approve` \| `low` \| `medium` \| `high` \| `critical` |
| `--skip-brainstorm` | off | skip Phase 1 plan + brainstorm; use forge inline |
| `--context-tight` | off | recommend session split at every Phase 4 iteration boundary |
| `--resume` | — | continue an interrupted run (no config overrides accepted) |
| `--continue` | — | extend a finished/stalled run; accepts `--max-iter`, `--threshold`, thinking-level overrides |
| `--reset-review-iter` | — | with `--continue`, reset iteration counter to 0 |
| `--help` / `--about` / `-h` | — | print this help and exit (no run started) |
| `--version` / `-v` / `-V` | — | print plugin version line and exit |

### Thinking levels

Pass as a bare token (no `--` prefix). Controls Phase 1 plan depth and Phase 4f fix-synthesis depth. Does NOT affect codex's own review depth.

| Level | Phrase inserted before deep work |
|---|---|
| `think` | "Think about this." |
| `think hard` *(default)* | "Think hard about this. Think hard." |
| `think harder` | "Think harder about this. Think harder." |
| `ultrathink` | "Ultrathink about this." |

### Severity threshold (inclusive — "highest acceptable severity")

| `--threshold` | Acceptable (non-blocking) | Blocks |
|---|---|---|
| `approve` *(strictest)* | nothing | any finding — loop only ends on a true codex `approve` verdict |
| `low` | low | medium + high + critical |
| `medium` *(default)* | low + medium | high + critical |
| `high` | low + medium + high | critical |
| `critical` *(loosest)* | any finding | nothing — first `needs-attention` verdict ends loop |

A finding blocks iff its severity rank is strictly greater than the threshold rank. Threshold names the **ceiling** of what's acceptable.

### Examples

```
# Defaults (think hard, max 5, threshold medium)
/clodex Add delete option for submissions in draft state

# Deeper planning + stricter threshold (only low findings acceptable) + 8 iterations
/clodex Refactor upload pipeline to use signed URLs ultrathink --max-iter 8 --threshold low

# Production-critical — only stop on a true codex approve verdict
/clodex Rewrite auth session store ultrathink --max-iter 10 --threshold approve

# Quick fix, no brainstorm
/clodex Fix the timezone bug in the daily digest think --skip-brainstorm

# Large PR, plan for multi-session up front
/clodex Implement OIDC auth ultrathink --max-iter 8 --context-tight

# Extend a finished run with more budget, loosening threshold (accept high+medium+low)
/clodex --continue --max-iter 10 --threshold high

# Re-enter review loop on existing branch+PR after an off-rails iteration
/clodex --continue --reset-review-iter --max-iter 5

# Resume an interrupted run as-is
/clodex --resume
```

### State and disk layout

`.clodex/state.json` (gitignored, schema v2) records task, branch, PR, iteration, findings_history, prior_runs, pending_migrations. Schema upgrade from v1 happens automatically on first read.

Per-iteration artifacts (as of v0.2.6) live under `.clodex/runs/<branch-slug>/` — one subdirectory per feature branch — with up to four files per iteration:

- `iter-NNN-focus.md` (always) — input: focus text sent to codex
- `iter-NNN-codex.json` (always) — output: codex's raw return, machine-readable
- `iter-NNN-review.md` (always, v0.2.6+) — output: human-readable rendering of codex.json
- `iter-NNN-fix.md` (only when findings-fixer was dispatched) — action: triage + commit summary

Iteration numbers are zero-padded to 3 digits. Extensions are semantic: `.json` for JSON, `.md` for markdown. The branch slug replaces `/` in the branch name with `--`.

Legacy flat files (`.clodex/iter-<N>-*.{txt,md}`) from pre-v0.2.6 runs are not auto-migrated — they remain as forensic artifacts. New runs write only under `runs/<branch-slug>/`.

### Phase outline

1. **Brainstorm + plan** — `superpowers:brainstorming`, `superpowers:writing-plans`. Skip with `--skip-brainstorm`.
2. **Branch + implement** — `new-branch`, then `superpowers:executing-plans` (plan-driven) or `forge` (plan-less).
3. **Ship** — `/ship` opens the PR.
4. **Review loop** (×max-iter): focus-runner agent → `node "$COMPANION_SCRIPT" adversarial-review` → poll via `status` → fetch via `result --json` → triage → findings-fixer agent → push.
5. **Report** — includes "did NOT receive codex approval" disclosure for non-approve verdicts, plus copy-pasteable next-step commands.

### Hard rules (v0.2.1+)

1. The codex plugin's adversarial-review code path is the only sanctioned review pass. Since codex plugin v1.0.4 all `/codex:*` commands are `disable-model-invocation: true`, so the sanctioned model-side entry point is `node "$COMPANION_SCRIPT" adversarial-review|status|result|cancel` via Bash. No OpenAI `codex` CLI, no custom wrappers around the companion script, no Claude-subagent substitutions.
2. No writes to plugin internal state (`~/.claude/plugins/data/`).
3. Always dispatch the `clodex-focus-runner` agent for Phase 4a (≤3 named failure modes).
4. Halt and surface on unexpected failures — never improvise an alternative path.

### Triggers

`/clodex <task>`, "ralph this", "run the codex loop on X", "automate the review-fix cycle for X".

### Docs

Full README + design rationale: `README.md` in the plugin install directory. Also on GitHub: https://github.com/lukaskucinski/clodex
