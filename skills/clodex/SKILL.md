---
name: clodex
description: |
  Autonomous dev loop: plan → branch → implement → ship PR → /codex:adversarial-review → fix → repeat. Halts on approve, threshold-satisfied, or max-iter hit. Never auto-merges, never applies migrations. Resumable + extensible across sessions via .clodex/state.json (v2 schema).

  FLAGS
    --max-iter N           iteration cap                                   (default: 5)
    --threshold X          highest acceptable severity: approve|low|medium|high|critical  (default: medium)
    --skip-brainstorm      skip Phase 1 plan + brainstorm; use forge inline
    --context-tight        recommend session split at every Phase 4 iteration boundary
    --resume               continue an interrupted run (no config overrides)
    --continue             extend a finished/stalled run (accepts override flags)
    --reset-review-iter    with --continue, reset iteration counter to 0

  THINKING LEVELS  think | think hard | think harder | ultrathink           (default: think hard)
  Controls Phase 1 plan depth and Phase 4f fix-synthesis depth. Pass as bare token.

  EXAMPLE
    /clodex Add delete option for submissions in draft state think harder --max-iter 5 --threshold medium

  HELP  /clodex --help   (full reference in-session if this tooltip is truncated)
  TRIGGERS  /clodex <task>, "ralph this", "run the codex loop on X", "automate the review-fix cycle for X".
argument-hint: "<task description> [think|think hard|think harder|ultrathink] [--max-iter N] [--threshold approve|low|medium|high|critical] [--skip-brainstorm] [--context-tight] [--resume] [--continue [--reset-review-iter]]"
---

# Clodex — Autonomous Plan → Ship → Review → Fix Loop

You are running clodex. Your job: drive a task from initial description to a codex-approved (or threshold-satisfying) PR, by composing existing skills and orchestrating a review-fix loop. You are the loop controller, not the implementer or the reviewer — those are existing skills and agents that already work. Glue, don't reinvent.

## Hard rules — NEVER violate these

**Authoritative source as of v0.2.6: `skills/clodex/HARD_RULES.md` in this plugin.** At the start of every run, **before any other work** (and immediately after the version banner in pre-flight), run `Bash(cat <plugin>/skills/clodex/HARD_RULES.md)` and treat its output as the authoritative hard-rules text — even if it conflicts with the rules reproduced below. Your loaded SKILL.md may be stale (Claude Code agent sessions cache skill content at session-start; `/reload-plugins` does not re-inject it), so a fresh read from disk is required.

The rules reproduced below mirror HARD_RULES.md and exist for reviewers reading SKILL.md. **If HARD_RULES.md is missing or unreadable, halt and surface** — do not fall back to these inlined rules silently, since "no fresh hard-rules read" is itself a structural problem the user should know about.

These rules exist because of real failures in v0.1. Read them once at the start and treat them as inviolable for the rest of the run. If a rule would force you to halt, halt — do not improvise around it.

1. **Codex is the only review pass.** The codex plugin's adversarial-review code path is the ONLY sanctioned way to invoke a review. Since codex plugin v1.0.4 (Apr 2026), all `/codex:*` slash commands are flagged `disable-model-invocation: true`, so they cannot be auto-dispatched from a model session — that's a deliberate codex policy, not a bug to route around. The sanctioned model-side path is the same code, just invoked one layer down: `node "${COMPANION_SCRIPT}" adversarial-review … | status … | result … | cancel …` via Bash. Resolve `${COMPANION_SCRIPT}` once in pre-flight (see "Pre-flight checks") and reuse for every subsequent call.

   If the companion script errors, returns no job ID, returns invalid JSON, or codex stalls past the polling cap: halt the loop, write `verdict='stalled'` (or `'error'`) to state, surface to the user with the job ID and current iteration, and stop.
   - Do NOT invoke `codex` / `codex review` / `codex exec` directly via Bash. (That's the OpenAI Codex CLI binary, separate from the codex Claude-Code plugin — clodex uses the plugin, not the bare CLI.)
   - Do NOT write your own wrapper around `codex-companion.mjs` or shell out via an intermediate script. The only sanctioned form is the direct `node "${COMPANION_SCRIPT}" <subcommand> …` call.
   - Do NOT dispatch Claude subagents (general-purpose or specialized) as a substitute review pass. Claude reviewing Claude's own work is not the independent perspective clodex exists to provide.
   - Do NOT write your own focus text inline and pass it to any entry point other than what `clodex-focus-runner` produces. (Hard rule 3.)
   - If the `/codex:adversarial-review` slash command ever flips back to `disable-model-invocation: false`, switching back to slash-command dispatch is fine — it's functionally identical to the companion-script call. But until then, the companion-script path IS the canonical one for this orchestrator.

2. **Never write to plugin internal state.** Specifically: `~/.claude/plugins/data/`, `~/.claude/plugins/cache/`, plugin `state.json` / `broker.json` / job log files, or any path under another plugin's install tree. If a codex job is stuck and `node "$COMPANION_SCRIPT" cancel <job-id>` does not clear it within 30 seconds, halt and surface — let the user clean up manually. Forcing the state via direct edits can corrupt future codex runs.

   **Positive form (so the boundary is unambiguous):** clodex writes ONLY to `.clodex/state.json` in the current repository, and to any files the implementation phase legitimately edits. Anything under `~/.claude/plugins/`, `~/AppData/Local/Temp/`, or any other plugin's data directory is OFF-LIMITS for writes. If you find yourself drafting a `Set-Content`, `Out-File`, or any other write that touches one of those paths, stop — that's the rule firing.

   Specific anti-patterns observed in v0.1 incidents:
   - `Stop-Process -Id <codex_pid> -Force; ...; Set-Content "...\codex-openai-codex\state\...\state.json"` — DO NOT do this. The state file belongs to the codex plugin's broker.
   - Manually editing job-record JSON files under `~/.claude/plugins/data/codex-openai-codex/state/<workspace>/jobs/` — DO NOT do this. Surface to user instead.

3. **Always dispatch the `clodex-focus-runner` agent for Phase 4a.** Never write the focus text inline in your own context. The agent enforces the focus-content discipline (≤3 named failure modes, trail-heads only, no boilerplate re-statement of the adversarial frame). If you find yourself drafting `(1) Trigger… (2) Pre-submit… (3) TOCTOU…` in your own response, you are off-spec — dispatch the agent instead.

4. **Halt and surface, do not improvise.** When any prescribed step fails in an unexpected way (slash command returns an error, an expected file does not exist, a subagent returns malformed output), the correct action is ALWAYS halt and surface. Resist the temptation to "find another way to make it work" — improvisation is exactly how v0.1 went off the rails on PR #78.

If you violate any of these rules, the run is invalid regardless of how the report reads at the end.

## Context budget discipline

A clodex run on a moderate PR can easily span 5+ Phase 4 iterations on top of a Phase 2 implementation. Context burns are the second-most-common cause of v0.1 runs going sideways (after the off-spec failures the hard rules address). Real telemetry: on PR #78, ~66% of context was used before any successful codex review had completed — 30–40% of that burn came from off-spec improvisation, but the rest was structurally expensive Phase 2 work.

Follow these rules to keep the orchestrator's context lean enough that long runs are tractable.

### Persist artifacts to disk; pass paths in chat

Every large artifact produced during the run lives on disk. Only path references and short summaries flow through the orchestrator's chat context.

**Canonical file layout (v0.2.6 — adds `iter-NNN-review.md`):** all per-iteration artifacts live under `.clodex/runs/<branch-slug>/`. The branch slug is the current `state.branch` with any `/` characters replaced by `--` (so `clodex/fix-draft-resume-500-20260516-1430` becomes `clodex--fix-draft-resume-500-20260516-1430`). One subdirectory per branch; `--continue` on the same branch appends to the same directory; a fresh branch gets a fresh subdirectory. Prior runs' files are preserved.

```
.clodex/
├── state.json                                      # orchestrator state (top-level)
├── .gitignore                                      # auto-created
└── runs/
    └── <branch-slug>/                              # per-branch namespace
        ├── iter-001-focus.md                       # input: focus text sent to codex
        ├── iter-001-codex.json                     # output: raw codex result JSON
        ├── iter-001-review.md                      # output: human-readable rendering of codex.json
        ├── iter-001-fix.md                         # action: findings-fixer return summary (only when dispatched)
        ├── iter-002-focus.md
        ├── iter-002-codex.json
        ├── iter-002-review.md
        ├── iter-002-fix.md
        └── ...
```

**Per-iteration filenames are exactly these four canonical names, no others.** `focus.md`, `codex.json`, and `review.md` are written every iteration. `fix.md` is written only when the findings-fixer was dispatched (verdict = `needs-attention` with blocking findings). Iteration numbers are **zero-padded to 3 digits** (`iter-001`, not `iter-1`). Extensions are semantic: `.json` for JSON content, `.md` for markdown. **No `.txt`, no `-findings.md` companion file, no per-iteration variants.** If you find yourself naming a file `iter-3-codex-findings.md` or `iter-2-codex.txt`, you are off-spec — re-read this section.

| Artifact | Canonical path | Written by | What may appear in chat |
|---|---|---|---|
| Focus text sent to codex | `.clodex/runs/<branch-slug>/iter-NNN-focus.md` | Orchestrator Phase 4b (write the focus-runner's return value before launching codex) | Path string only |
| Codex JSON output (full) | `.clodex/runs/<branch-slug>/iter-NNN-codex.json` | Orchestrator Phase 4d (`node "$COMPANION_SCRIPT" result <job-id> --json > <path>`) | Path string + one-line surface summary (`[clodex iter <N>/<max>] verdict=… | Cc/Hh/Mm/Ll | "…"`) |
| Codex review rendering (human-readable) | `.clodex/runs/<branch-slug>/iter-NNN-review.md` | Orchestrator Phase 4d (render the codex.json into markdown per the "Review rendering" template below) | Path string only |
| Findings-fixer return summary | `.clodex/runs/<branch-slug>/iter-NNN-fix.md` | Orchestrator Phase 4f (write the structured summary the agent returned) — only when findings-fixer was dispatched | Path string + one-line "fix: <commit-sha> closed <N> findings" |
| Plan markdown | `<plan_file>` from Phase 1 (outside `.clodex/`) | `superpowers:writing-plans` | Path string only |
| State | `.clodex/state.json` | Orchestrator throughout | Specific fields read as needed; never whole file inline |

### Review rendering template (v0.2.6 — iter-NNN-review.md)

Write this file immediately after `iter-NNN-codex.json` in Phase 4d. The content is a markdown rendering of the codex JSON's verdict, summary, findings, and next_steps. Template:

````markdown
# iter-NNN — codex adversarial review

**Verdict:** <verdict>
**Counts:** <C> critical · <H> high · <M> medium · <L> low
**Threshold:** <threshold> (<blocking_count> blocking findings strictly above ceiling, or "no blocking findings")
**Job ID:** <job_id>
**Elapsed:** <elapsed from codex.json or status snapshot>

## Summary

<codex.summary verbatim>

## Findings

<if no findings:>
_None — codex returned `approve`._
<else for each finding in codex.findings, severity-sorted critical → high → medium → low:>
### [<severity>] <title>
**File:** `<file>:<line_start>-<line_end>` (or just `<file>:<line_start>` if same)
**Confidence:** <confidence>

<body verbatim>

**Recommendation:** <recommendation verbatim>

---
<end for>

## Next steps (from codex)

<for each item in codex.next_steps:>
- <item verbatim>
<end for>

<if findings-fixer was dispatched after this review (verdict = needs-attention with blocking findings):>
## Action

Findings-fixer dispatched — see [`iter-NNN-fix.md`](iter-NNN-fix.md) for triage and commit details.
<end if>
````

Do not editorialize the codex content — verdict, summary, finding bodies, recommendations, and next_steps are reproduced verbatim from `codex.json`. Severity badges, file:line formatting, and section headers are clodex's renderings to make the file scannable.

**Directory creation:** before writing the first per-iteration file in any iteration, ensure the run directory exists: `mkdir -p .clodex/runs/<branch-slug>` (PowerShell: `New-Item -ItemType Directory -Force -Path .clodex/runs/<branch-slug> | Out-Null`). Idempotent — safe to run every iteration.

**Legacy files (pre-v0.2.6):** the prior flat scheme `.clodex/iter-<N>-*.{txt,md,json}` is deprecated. Do not write new files under the flat scheme. Existing legacy files from prior runs are left in place as forensic artifacts — surface them in the Phase 5 report's "Pre-existing artifacts" line if any are found at the top level of `.clodex/` matching `iter-*`. Do not auto-delete or auto-migrate them.

When dispatching subagents, pass paths in the prompt rather than embedding content. The findings-fixer agent reads `codex_output_path` from disk — do NOT paste the codex JSON into the agent's prompt.

### Don't tail codex log files or job output

The log file under `~/.claude/plugins/data/codex-openai-codex/state/<workspace>/jobs/*.log` is verbose, uninformative for review purposes, and counts against context budget. v0.1 runs burned significant context on `tail -300` and `tail -200` of these logs.

- For job state: use `node "$COMPANION_SCRIPT" status <job-id>` — returns short formatted status
- For findings: use `node "$COMPANION_SCRIPT" result <job-id> --json` — returns parsed JSON to stdout (redirect to disk)
- For "is the job still progressing?": check the LOG FILE'S `LastWriteTime` (`Get-ChildItem ... | Select LastWriteTime`) without reading the log contents

If you must read a log file (rare — almost never required), cap at 50 lines.

### Don't re-read the diff

Implementation-phase reads of source files are unavoidable. Re-reading the diff during the review loop is not — the codex review covers the diff for you. If you find yourself running `git diff main...HEAD` in the orchestrator's chat after Phase 4 starts, that's wasted context.

The findings-fixer agent will read whatever specific files it needs to apply fixes; let it.

### No giant JSON blob inspections

PowerShell patterns like `... | ConvertTo-Json -Depth 8 | Format-Table -AutoSize` produce multi-screen outputs that flood chat. v0.1 runs burned context on these when inspecting codex broker state.

- Use the codex plugin's text status command: `node "<companion-path>" status` returns a markdown table, not a JSON blob
- If you need a specific field from `.clodex/state.json`, read the file and extract the field — don't pretty-print the whole structure to chat

### Bounded reads

Defaults for any deliberate file read inside an active loop iteration:

- Source files: ≤200 lines unless directly necessary for the work
- Log files: ≤50 lines
- Test output: ≤100 lines (use `--reporter=dot` to compress)
- JSON state: read by key, not whole file

These are guidelines, not hard caps — sometimes a 500-line file genuinely needs to be read whole. But default to the bounds and exceed them only with clear purpose.

### Context discipline interacts with the hard rules

Most context bloat in v0.1 came from improvisation that the hard rules now prevent: failed CLI fallback attempts, tailing logs to debug stuck jobs, 4-agent fallback transcripts, manual state-file edits. The halt-and-surface rule is also, incidentally, a context-budget rule.

## When to activate

- User types `/clodex <task>` (any variant of the slash command)
- User says "ralph this", "run the codex loop on …", "automate the review-fix cycle for …"
- User explicitly invokes the `clodex` skill

## Argument parsing

Raw input is in `$ARGUMENTS`. Parse in this order:

0. **Help / version short-circuit — emit via Bash, do not paraphrase.**

   Both branches below MUST use the Bash tool to produce the text. Do NOT generate the help/version output from your own context — `cat`/`printf` output is what gets shown. This guards against the VS Code Claude Code extension's tendency to paraphrase verbose markdown blocks (the terminal CLI emits verbatim correctly; the extension does not).

   - **`--version` / `-v` / `-V`** anywhere in `$ARGUMENTS`:
     Run exactly: `Bash(command: 'printf "clodex@lukas-local v0.2.6\n"', description: "Print clodex version")`. The tool output is the user-visible response. Emit it as-is — no preamble, no commentary, no surrounding markdown. Then STOP.

   - **`--help` / `--about` / `-h`** anywhere in `$ARGUMENTS`:
     1. Resolve the HELP.md path. The plugin install path is at `${CLAUDE_PLUGIN_ROOT}` when set; the help file is at `${CLAUDE_PLUGIN_ROOT}/skills/clodex/HELP.md`. If `$CLAUDE_PLUGIN_ROOT` is unset, fall back to `Glob: ~/.claude/plugins/**/clodex/skills/clodex/HELP.md` and pick the first match (there should be exactly one).
     2. Run `Bash(command: 'cat "<resolved-path>"', description: "Print clodex help")` — Bash's stdout is the user-visible response.
     3. **Do not transform, summarize, paraphrase, re-wrap, or rewrite the cat output in any way.** If the user sees anything other than the literal HELP.md contents, you violated this instruction. The HELP.md file is the canonical reference; SKILL.md merely renders it. Bump v-string in HELP.md when the plugin version bumps; the version string elsewhere in this file (e.g. the `--version` printf above) must stay in sync.
     4. STOP. Do not run pre-flight checks, do not read state, do not start any phase.

   This short-circuit is a pure documentation/version lookup and must complete in 1–2 tool calls (Bash, optionally a Glob if `$CLAUDE_PLUGIN_ROOT` is unset). If you find yourself reading state, dispatching agents, or generating the help block from memory, you are off-spec — re-read this section.

1. Flags (extract and remove):
   - `--max-iter N` (default: `5`) — maximum review→fix iterations in Phase 4
   - `--threshold {approve|low|medium|high|critical}` (default: `medium`) — highest severity that is **acceptable** (non-blocking). Findings strictly above this severity block the ship. Special value `approve` means "nothing is acceptable" — the loop only exits cleanly on a true codex `approve` verdict.
   - `--skip-brainstorm` — skip Phase 1 brainstorming, go straight to plan
   - `--context-tight` — make the orchestrator aggressive about session-split recommendations. With this set, the iteration-start self-check (see Phase 4) recommends `/clear` + `/clodex --resume` at every iteration boundary, regardless of estimated context use. Useful for large PRs where the user knows in advance they'll span multiple sessions. Persists in state so `--resume` continues honouring it without needing to be re-passed.
   - `--resume` — read `.clodex/state.json` and continue an **interrupted** prior run (verdict still null / in-progress). Does NOT accept overrides; restores the prior config as-is.
   - `--continue` — explicitly extend a **finished or off-rails** prior run (verdict is one of `approve`, `threshold-satisfied`, `max-iter-hit`, `stalled`, `error`, `no-diff`, or null but iteration > 0 with an existing branch+PR). Accepts overrides via `--max-iter` / `--threshold` / thinking-level. Reset verdict to null, archive prior `findings_history` into `prior_runs`, and re-enter Phase 4.
   - `--reset-review-iter` — only valid with `--continue`. Resets `iteration` to 0 and starts the review counter fresh under the new `max_iter`. Without it, `--continue` resumes at `state.iteration + 1` (capped at the new `max_iter`).
2. Thinking level token (anywhere in remaining args): `think` | `think hard` | `think harder` | `ultrathink`. Default: `think hard`. Match longest first (`think harder` before `think hard` before `think`).
3. Everything else, in order, joined with single spaces = the **task description**.

If the task description is empty and neither `--resume` nor `--continue` is set, stop and ask the user for the task.

`--resume` and `--continue` are mutually exclusive. If both are passed, halt with an error explaining the distinction.

Threshold is **inclusive** — the value names the highest severity that is acceptable as non-blocking. A finding blocks iff its severity rank is strictly greater than the threshold rank.

Severity ranks: `critical=4, high=3, medium=2, low=1`. Threshold ranks add `approve=0` at the bottom: `approve=0, low=1, medium=2, high=3, critical=4`.

- `--threshold approve` — nothing acceptable. Loop exits ONLY on a true codex `approve` verdict (or `max-iter-hit`/`stalled`/`error`/`no-diff`). Never produces `threshold-satisfied`.
- `--threshold low` — low findings are acceptable; medium/high/critical block.
- `--threshold medium` *(default)* — medium and low acceptable; high/critical block.
- `--threshold high` — high, medium, and low acceptable; only critical blocks.
- `--threshold critical` — any finding is acceptable. First `needs-attention` verdict with findings produces `threshold-satisfied`.

## Help / About output

**Canonical source as of v0.2.6:** `skills/clodex/HELP.md` in this plugin. The `--help` short-circuit in parse step 0 above MUST `cat` that file via Bash; do not render this fallback block from your own context.

The block reproduced below mirrors HELP.md and exists as a SKILL-internal reference (so reviewers reading SKILL.md can see what users get) and as a fallback if HELP.md is somehow missing. When parse step 0 cannot resolve HELP.md (Glob returns no match), emit the following markdown block verbatim to the user and STOP. Do NOT prepend chatter ("Here's the help…"), do NOT append commentary. Keep HELP.md and this block in sync on every edit.

````markdown
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
```

### Flags

| Flag | Default | Description |
|---|---|---|
| `--max-iter N` | `5` | iteration cap for the review→fix loop |
| `--threshold X` | `medium` | highest acceptable severity: `approve` \| `low` \| `medium` \| `high` \| `critical` |
| `--skip-brainstorm` | off | skip Phase 1 plan + brainstorm; use forge inline |
| `--context-tight` | off | recommend session split at every Phase 4 iteration boundary |
| `--resume` | — | continue an interrupted run (no config overrides accepted) |
| `--continue` | — | extend a finished/stalled run; accepts `--max-iter`, `--threshold`, thinking-level overrides |
| `--reset-review-iter` | — | with `--continue`, reset iteration counter to 0 |
| `--help` / `--about` / `-h` | — | print this help and exit (no run started) |

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

### State

`.clodex/state.json` (gitignored, schema v2) records task, branch, PR, iteration, findings_history, prior_runs, pending_migrations. Schema upgrade from v1 happens automatically on first read.

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

Full README + design rationale: `README.md` in the plugin install directory (`~/.claude/plugins/marketplaces/lukas-local/clodex/README.md` or the source clone). Also on GitHub: https://github.com/lukaskucinski/clodex
````

End of the verbatim help block. After emitting it, stop.

## State file

After Phase 1 (and after every later phase that changes state), write `.clodex/state.json` at the repo root:

```json
{
  "version": 2,
  "task": "<original task description>",
  "thinking_level": "think hard",
  "max_iter": 5,
  "threshold": "medium",
  "context_tight": false,
  "started_at": "<ISO timestamp>",
  "branch": null,
  "plan_file": null,
  "pr_url": null,
  "pr_number": null,
  "iteration": 0,
  "last_codex_job_id": null,
  "verdict": null,
  "findings_history": [],
  "prior_runs": [],
  "pending_migrations": []
}
```

`findings_history[i]` records iteration `i`'s codex output (compact): `{ iteration, job_id, verdict, summary, counts: {critical, high, medium, low}, blocking_count, committed_fix_sha, review_method, focus_path, codex_json_path, review_path, fix_path }`. The four `*_path` fields point at the canonical per-iteration files under `.clodex/runs/<branch-slug>/` (see "Persist artifacts to disk" in Context budget discipline). `fix_path` is `null` when no findings-fixer was dispatched (verdict was `approve` or `threshold-satisfied`); the other three paths are always populated.

`review_method` is required (see "State file: per-iteration `review_method`" in Phase 4c).

Top-level `verdict` values, in increasing severity of "this didn't work":
- `approve` — codex returned `approve`
- `threshold-satisfied` — codex returned `needs-attention` but no findings have severity strictly above the threshold ceiling (all remaining findings are acceptable)
- `needs-attention` — interim state during the loop; never a terminal value
- `no-diff` — focus-runner reported empty diff; nothing for codex to review
- `max-iter-hit` — loop exhausted `max_iter` with blocking findings still present
- `stalled` — one codex job didn't complete within the polling cap
- `codex-reproducible-hang` — two consecutive iterations stalled at the same phase; the diff itself is the suspect
- `error` — codex slash command errored, returned no job ID, or returned malformed output

`prior_runs[j]` archives an earlier run that was extended via `--continue`. Each entry: `{ extended_at: <ISO>, prior_verdict, prior_max_iter, prior_threshold, findings_history: [...the prior history...] }`. The current `findings_history` always reflects the active run's iterations only (under the current `max_iter`); `prior_runs` preserves the full timeline for the Phase 5 report.

`version: 2` distinguishes from v0.1 state files. If you read a state file with `version: 1` or missing `version`, treat it as v1 — copy `findings_history` into `prior_runs[0].findings_history`, clear the current `findings_history`, and upgrade. Document the upgrade in the Phase 5 report.

Create `.clodex/` if it doesn't exist. Add `.clodex/` to `.gitignore` if not already present (it's local working state, not source). Do this with a single appending Edit; do not rewrite the file.

## Pre-flight checks (run once at start, in parallel)

**Before the first pre-flight tool call,** emit a single banner line to chat so the user can verify which plugin version is active without reading the help block:

```
[clodex v0.2.6] starting <run-type>: <task description, truncated to 80 chars>
```

`<run-type>` is one of `new task`, `--resume`, `--continue`. For `--resume` / `--continue`, the task description comes from `state.json`. Update this version string whenever the plugin version bumps — it's the user's only quick verification that the new version actually loaded.

**Immediately after the banner, before any other tool call**, do a fresh hard-rules read:

1. Resolve `<plugin>` via `Glob ~/.claude/plugins/**/clodex/skills/clodex/HARD_RULES.md` (pick the first/only match).
2. Run `Bash(command: 'cat "<resolved-path>"', description: "Read fresh hard rules")` and treat its stdout as the authoritative hard-rules text for this run (see "Hard rules — NEVER violate these" section above). If the rules in your loaded SKILL.md context conflict with this fresh read, the fresh read wins — your loaded SKILL.md may be stale.
3. Append the HARD_RULES.md version line (last line of the file) to the banner: `[clodex v0.2.6 / hard-rules v0.2.5] starting ...` (re-emit the banner with both versions; this lets the user spot the case where SKILL.md and HARD_RULES.md are out of sync. Plugin version (v0.2.6) and hard-rules version (v0.2.5) can legitimately differ — they're bumped independently. Plugin version bumps whenever any file changes; hard-rules version bumps only when the rules themselves change.).

Then run these in parallel:

- `git rev-parse --is-inside-work-tree` — must be a git repo
- `git remote -v` — must have a remote (need it to push and open a PR)
- `gh auth status` — `gh` must be authenticated
- `git status --short` — record uncommitted/untracked changes
- Confirm the codex plugin is installed: `Glob ~/.claude/plugins/**/codex/commands/adversarial-review.md` (one or more matches; pick the first)
- Resolve and pin the **codex companion script path** for the rest of the run: `Glob ~/.claude/plugins/**/codex/scripts/codex-companion.mjs` — there should be exactly one match. Store it as `$COMPANION_SCRIPT` (in your own state, not on disk — it's stable across the run). Every codex invocation in Phase 4b/c/d/cancel uses this exact path.
- Confirm `/codex-focus` exists: `Glob ~/.claude/commands/codex-focus.md` or `~/.claude/plugins/**/codex-focus*`

If any check fails, stop and surface a one-line explanation to the user. Do not try to "fix" missing prerequisites silently.

If `Glob` returns multiple `codex-companion.mjs` matches (rare — would only happen with multiple codex plugin installs), pick the one whose path matches the same plugin install as the `adversarial-review.md` you just confirmed. The two MUST come from the same plugin root, or the companion-script invocation won't match the slash-command's framing.

If there are uncommitted changes on a non-main branch: this is the user's in-progress work. Default to including those changes in the clodex run (treat them as the starting point of the implementation). Only stash/discard with explicit user instruction.

## Phase 1 — Brainstorm + plan

**Skip this phase entirely if `--skip-brainstorm` is set.** Otherwise:

1. Invoke the `superpowers:brainstorming` skill via the `Skill` tool, passing the task description. Brainstorming will Q/A with the user about intent, requirements, design.
2. After brainstorming concludes (or if the user declined Q/A), invoke `superpowers:writing-plans` to produce a plan markdown file. Capture the plan file path.

If `--skip-brainstorm`:
- Use `TodoWrite` to write a brief inline task list — no separate plan file.
- Set `plan_file` to `null` in state.

Update state with `plan_file` path.

## Phase 2 — Branch + implement

1. **Create branch.** Invoke the `new-branch` skill, or directly run:
   ```bash
   git fetch origin main
   git checkout -b clodex/<short-slug>-<YYYYMMDD-HHmm> origin/main
   ```
   Choose `<short-slug>` from the task (3–5 kebab-case words). Update state with the branch name. If the user already started on a non-main feature branch with uncommitted changes (see pre-flight), instead reuse that branch and skip the checkout — record the existing branch name in state.

2. **Implement.** Pick exactly one path based on whether a plan file exists:
   - **Plan file exists:** Invoke `superpowers:executing-plans` with the plan file path.
   - **No plan file:** Invoke the `forge` skill with `$THINKING_LEVEL` and the task description. `forge` will create its own branch slug if asked, but since we already made a branch in step 1, tell forge to "continue on current branch — do not create a new one" so it skips its branching step.

3. **Discipline.** For new behavior, follow `superpowers:test-driven-development`. For bug fixes, follow `superpowers:systematic-debugging`. These are procedural guards — invoke them at the start of implementation if relevant.

4. **Local verification.** Read project CLAUDE.md if present and run the project's standard verification (tests, build, typecheck, lint). Fix anything red before proceeding. Per `superpowers:verification-before-completion`: do not advance to ship until verification is green.

   **Migration exception.** If a verification step requires applying migrations to a remote database (`supabase db push`, `npm run db:push`, etc), skip that specific step — see the "Migration safety" section. Run every other verification step normally. Note skipped steps so they appear in the Phase 5 report.

## Phase 3 — Ship

Invoke the `ship` skill (`/ship`). It stages explicitly-named files, commits with an imperative message, pushes, and opens a PR via `gh pr create`. Capture from its output:

- The PR URL → `pr_url` in state
- The PR number → `pr_number` in state (parse from URL: `gh pr view --json number -q .number` if needed)

If `/ship` reports the PR already exists, capture the existing PR URL and proceed. Do **not** open a duplicate.

## Phase 4 — Review loop (the core of clodex)

Reset `iteration` to `0` in state if not resuming. Then loop:

```
while iteration < max_iter:
  iteration += 1
  iteration_start_self_check()
  run_one_review_round(iteration)
```

### Iteration-boundary self-check (context budget)

At the start of each iteration, before dispatching the focus-runner, run this self-check:

1. **Estimate current context use** — impressionistically (there's no precise API). Indicators:
   - Has Phase 2 implementation produced more than ~10 file reads / writes since this session started?
   - Have you run more than ~30 tool calls so far?
   - Does your scrollback feel "long"?
2. **Apply the decision rule:**
   - If `--context-tight` is set: ALWAYS recommend session split before this iteration (see below)
   - If context feels tight (you would answer "yes" to 2+ of the above): recommend session split
   - If context feels fine: proceed normally with this iteration

3. **Session split recommendation format** — surface this to the user as a clear pause, then halt the loop cleanly:

```
[clodex iter <N>/<max>] context check: this session has done <brief summary> already.
Recommend splitting here:
  1. /clear              — start a fresh session
  2. /clodex --resume    — pick up at iteration <N>

State is fully captured in .clodex/state.json. Branch: <branch>. PR: <pr_url>.
Halting. Run the two commands above to continue.
```

Write `verdict: null` (loop still in-progress) and stop. The user runs `/clear` then `/clodex --resume` and a fresh session resumes at the same iteration.

### Multi-session runs are first-class

A clodex run on a moderate PR (10+ files, 1000+ lines of diff) commonly spans 2–3 sessions. This is NORMAL, not a degraded fallback:

- Phase 2 implementation typically fills one session on its own for non-trivial diffs
- Phase 4 iterations are atomic — each starts and ends with state fully captured in `.clodex/state.json`
- `/clodex --resume` jumps to the right phase based on state and continues with fresh context

For users who know in advance they're running a large task, the `--context-tight` flag (see argument parsing) makes the session-split recommendation fire at every iteration boundary, producing a clean per-iteration session split pattern.

### 4a. Generate focus (dispatch `clodex-focus-runner` agent — MANDATORY)

Dispatch the `clodex-focus-runner` agent (defined in `agents/clodex-focus-runner.md` in this plugin) with description `"Generate codex focus for iteration N"` and a prompt that tells it:

- Scope: `branch` (we are on the feature branch vs `main`)
- Iteration number, and (if iteration > 1) a compact prior_findings_summary
- Return ONLY the exact `/codex:adversarial-review …` invocation line that `/codex-focus branch` emits — strip preamble, fences, and trailing instructions
- Cap of ≤3 named failure modes in the focus text (v0.2 — see "Why" below)
- Do not run the review; do not propose fixes

**You MUST dispatch this agent.** Drafting the focus text inline in your own context is a v0.1 failure mode (cf. PR #78 incident — the orchestrator skipped the agent, wrote a 6.5KB / 10-failure-mode focus inline, and codex ran out of synthesis budget chasing all 10). The agent exists to enforce the focus-content discipline; bypassing it defeats the design.

You receive back exactly one of:

- A single line starting with `/codex:adversarial-review --background …` — proceed to Phase 4b (you will translate the `/codex:adversarial-review` prefix to `node "$COMPANION_SCRIPT" adversarial-review` there; the agent always returns the slash-command form because that's the canonical user-pasteable form `/codex-focus` emits)
- The literal string `NO_DIFF` — the loop is done, no changes for codex to review. Set `verdict='no-diff'` and skip to Phase 5
- A single line starting with `ERROR:` — halt the loop, surface the error to the user. Do NOT improvise an alternative focus

If the returned line does NOT start with `/codex:adversarial-review`, halt — the agent is misbehaving and you must not paper over it by invoking codex some other way.

### 4b. Run adversarial review (companion-script via Bash)

The focus-runner returns the canonical `/codex:adversarial-review --background --base main "FOCUS TEXT"` form. **Translate** this into the companion-script invocation before running — replace the `/codex:adversarial-review` prefix with `node "$COMPANION_SCRIPT" adversarial-review`. Preserve everything else (`--background`, `--base main`, the quoted focus text) verbatim.

So a focus-runner output of:
```
/codex:adversarial-review --background --base main "Application scale: ... Trail-heads: ... Named failure modes: ..."
```

becomes the Bash invocation:
```bash
node "$COMPANION_SCRIPT" adversarial-review --background --base main "Application scale: ... Trail-heads: ... Named failure modes: ..."
```

Always include `--background` regardless of what codex-focus picked (the loop's coordinator must not block on a foreground codex run). If the captured command has `--wait`, replace with `--background` before translating. The `--background` flag on `adversarial-review` is a no-op inside the companion script for this subcommand; the actual detachment comes from the Bash tool's `run_in_background: true` (next step).

**Before launching:** ensure the run directory exists and persist the focus text to disk for forensics. Replace `<branch-slug>` with the current branch with `/` → `--` (e.g. `clodex--fix-draft-resume-500-20260516-1430`):

```
mkdir -p .clodex/runs/<branch-slug>
# Write the focus-runner's return value to: .clodex/runs/<branch-slug>/iter-<padded-N>-focus.md
# (use the Write tool; the file's content is the literal /codex:adversarial-review … line)
```

Then run the translated command via `Bash` with `run_in_background: true`. Capture the returned `shell_id`. The Bash tool returns immediately; the node process continues running detached, writing job state to the codex broker.

Immediately after launching, **recover the job ID** by querying the broker (the launching Bash call returned no stdout because it's detached):

```bash
node "$COMPANION_SCRIPT" status --all --json
```

The output is a JSON array of all known jobs. Find the most-recent entry where `kind == "adversarial-review"` and `state` is one of `queued` / `running` (or `created_at` is within the last 10 seconds). Its `id` field is the job ID. Record it as `last_codex_job_id` in state. Also persist the per-iteration `review_method` (see Phase 4c) — set to `codex-adversarial-review` on success.

**If `node "$COMPANION_SCRIPT" adversarial-review …` errors immediately (non-zero exit before backgrounding), if `status --all --json` returns no matching job after a 2s retry, or if the recovered job's state is `error` / `failed`:** halt the loop, set `verdict='error'`, surface the error to the user. Do NOT fall back to the OpenAI `codex` CLI, do NOT dispatch a Claude subagent as a substitute, do NOT write to broker state to "fix" a stuck job. (See Hard rule 1.)

### 4c. Wait for completion (active poll — do not trust notifications alone)

On this user's machine, adversarial-review completions historically take **2–4 minutes** (median ~2:30, longest observed ~4:10). The harness notification for the background bash process is unreliable under VS Code, so **actively poll** rather than wait passively. Schedule explicit status checks at concrete elapsed times.

Polling protocol (replace any reference to `/codex:status <job-id>` with the companion-script equivalent: `node "$COMPANION_SCRIPT" status <job-id>`):

| Elapsed | Action |
|---|---|
| t=0 | Codex launched (Phase 4b). Record start time. |
| t≈120s | Sleep 120s (`Start-Sleep -Seconds 120` on PowerShell; `sleep 120` on bash), then run `node "$COMPANION_SCRIPT" status <job-id>`. ~Half of historical runs are done by now. |
| t≈210s | If still running, sleep 90s, then re-check. |
| t≈300s | If still running, sleep 90s, then re-check. Most runs finish here. |
| t≈360s | If still running, **print a user-visible heartbeat**: `"[clodex iter N] codex still running at 6m — historical max was 4m10s, will keep checking every 90s up to 10m total"`. Sleep 90s, re-check. |
| t≈450s | If still running, sleep 90s, re-check. |
| t≈540s | If still running, sleep 60s, re-check. |
| t≈600s | If still running after 10 min total, **halt this iteration** (see "Stalled handling" below). |

Use a single short sleep per cycle (≤120s); never chain a long sleep, never use `ScheduleWakeup` (that's a `/loop`-mode tool, not a session-pacing primitive). The Bash tool's per-sleep block applies only to leading sleeps — a sleep followed by a status check is fine.

You may also check the background bash launched in Phase 4b via `BashOutput` periodically — if the node process has exited cleanly, the review JSON is in stdout and you can skip straight to Phase 4d. Either signal (broker `state=completed` or background bash exited) is sufficient to proceed.

#### Transient-error recovery (v0.2 — model outages, broker disconnects)

If `node "$COMPANION_SCRIPT" status <job-id>` returns an error (e.g., broker pipe disconnect, 5xx from a model provider, network blip) rather than a "running" or "completed" status, this is likely a transient outage — codex itself may have hit a 529 or similar mid-synthesis. Do NOT immediately halt.

Retry once: wait 60s, re-run `node "$COMPANION_SCRIPT" status <job-id>`. If the second check also errors OR returns "running" but the job's underlying log file has not been written to in the last 5 minutes (check `Get-ChildItem ... -LastWriteTime` against `~/.claude/plugins/data/codex-openai-codex/state/<workspace>/jobs/*.log` — read-only metadata access only; do NOT read the log contents and do NOT write to that directory), proceed to "Stalled handling" below. Otherwise resume the normal poll cadence.

Do this retry at most once per iteration. If you retried earlier this iteration and the new check errors again, halt immediately — repeated transient errors are no longer transient.

#### Stalled handling (10-min cap or repeated errors)

When you hit either the 10-min cap or repeated transient errors:

1. Invoke `node "$COMPANION_SCRIPT" cancel <job-id>` once. Give it up to 30 seconds.
2. Re-check `node "$COMPANION_SCRIPT" status <job-id>`. If the job is no longer running, good.
3. If the job is STILL running after the cancel: surface to the user with the job ID, the elapsed time, and explicit copy-pasteable PowerShell to investigate. Do NOT attempt to kill the process directly, do NOT edit any plugin state file, do NOT touch `~/.claude/plugins/data/`. (See Hard rule 2.)
4. Decide which verdict to set:
   - **`codex-reproducible-hang`** if this is the 2nd consecutive iteration where codex stalled at substantially the same phase (e.g., both iterations ran diff-exploration commands for ~1 min then went silent, never produced synthesis output). Check `state.findings_history` for the prior iteration's `verdict` — if it was `stalled` and the failure shape matches, escalate to `codex-reproducible-hang`. This is a stronger signal than a one-off stall: the diff itself is exceeding codex's synthesis budget on this machine, and more iterations on the same branch will hang the same way.
   - **`stalled`** if this is the first such failure or the failure shape differs from the prior iteration.
5. Write `last_codex_job_id`, the per-iteration `review_method` (see "State file: per-iteration review_method"), and a Phase-5-equivalent stalled report.

Different verdicts surface different next-step suggestions in the report:

- `stalled` → suggest `/clodex --continue` after the user clears the stuck job
- `codex-reproducible-hang` → do NOT suggest a plain `--continue` (codex will hang again on the same diff). Suggest one of: split the PR into smaller pieces; re-run with a tighter focus targeting a single commit (`--scope commit:<sha>`); run `/codex:adversarial-review --base main` manually from the user's chat (no custom focus — the default template often succeeds where a custom focus stalls). The slash command runs fine from a user-initiated turn — it's only blocked from model auto-invocation.

#### State file: per-iteration `review_method`

Every `findings_history[i]` entry carries `review_method` recording how the review actually executed for that iteration:

- `codex-adversarial-review` — canonical path; what every legitimate iteration should report
- `codex-stalled` — codex job hung; no findings produced
- `codex-reproducible-hang` — codex hung at the same phase as the prior iteration
- `codex-error` — slash command errored or returned malformed output

The value `parallel-claude-reviewers-substitute` (or any other Claude-as-reviewer surrogate) must NEVER appear. It's named here only so the field's audit purpose is explicit: any tool reading `findings_history` can tell at a glance whether a given iteration's verdict came from codex or not.

### 4d. Retrieve result (and ALWAYS surface a summary line)

Run `node "$COMPANION_SCRIPT" result <job-id> --json` and capture stdout. The output conforms to `review-output.schema.json`:

```json
{
  "verdict": "approve" | "needs-attention",
  "summary": "<one-line ship/no-ship assessment>",
  "findings": [
    { "severity": "critical|high|medium|low", "title": "...", "body": "...",
      "file": "...", "line_start": 0, "line_end": 0, "confidence": 0.0,
      "recommendation": "..." }
  ],
  "next_steps": ["..."]
}
```

**Immediately after retrieving the result, before any decision logic, print a single user-visible line to the conversation.** This is non-negotiable — the user runs clodex in VS Code where codex output occasionally fails to surface to chat, so the orchestrator must echo the result itself every iteration. Format:

```
[clodex iter <N>/<max>] verdict=<verdict> | <C>c / <H>h / <M>m / <L>l | "<summary up to 200 chars>"
```

Example:
```
[clodex iter 2/5] verdict=needs-attention | 0c / 1h / 2m / 0l | "Fix L closes prior static path but a TOCTOU window remains between the new status-check and storage.remove."
```

Persist the full result JSON to `.clodex/runs/<branch-slug>/iter-<padded-N>-codex.json` for the findings-fixer subagent to consume — use shell redirection in the same Bash call:

```
node "$COMPANION_SCRIPT" result <job-id> --json > .clodex/runs/<branch-slug>/iter-<padded-N>-codex.json
```

(`<padded-N>` is zero-padded to 3 digits — `001`, `002`, etc. `<branch-slug>` is the current branch with `/` → `--`.)

**Then, in the same Phase 4d step, render the codex JSON to a human-readable `iter-<padded-N>-review.md` per the "Review rendering template" above.** Read the codex JSON, transform per template, Write to the canonical path. Do not skip this step on `approve` verdicts — the rendering exists so the user can navigate the runs/ directory and read what happened in each iteration without needing to `jq` the JSON. (See "Persist artifacts to disk; pass paths in chat" for the full file layout rationale.)

Append a compact entry to `findings_history` in state, including the paths as `codex_json_path` and `review_path`.

### 4e. Decide

```
if verdict == "approve":
  set state.verdict = "approve"; break loop; go to Phase 5.

# threshold-rank: approve=0, low=1, medium=2, high=3, critical=4
# severity-rank: low=1, medium=2, high=3, critical=4
# A finding blocks if its severity rank is STRICTLY GREATER than the threshold rank.
# When threshold='approve' (rank 0), every severity rank ≥ 1 blocks → all findings block.
# When threshold='critical' (rank 4), no severity rank > 4 → no findings block (always threshold-satisfied).
blocking = [f for f in findings if rank(f.severity) > threshold_rank(threshold)]

if len(blocking) == 0:
  set state.verdict = "threshold-satisfied"; break loop; go to Phase 5.

if iteration >= max_iter:
  set state.verdict = "max-iter-hit"; break loop; go to Phase 5.

# else: fix and loop
```

### 4f. Fix (dispatch `clodex-findings-fixer` agent)

Dispatch the `clodex-findings-fixer` agent (defined in `agents/clodex-findings-fixer.md`) with:

- The full codex JSON output (or a path to it written to `.clodex/iter-<N>-codex.json`)
- The thinking level (`$THINKING_LEVEL`)
- The blocking-findings subset (so it knows what's required vs. nice-to-have)
- Instruction to: triage findings → form a minimal fix plan → implement → run local verification → commit on the current branch → push

The fixer reports back: the new commit SHA, a one-line summary of what was fixed, and any findings it explicitly chose not to fix (e.g. it disagreed with codex). Record the commit SHA on the iteration's findings_history entry.

**Persist the fixer's structured return to disk** for forensics and to keep the orchestrator's chat context lean: write the fixer's full structured return (COMMIT_SHA, FIXED, DISAGREED, DEFERRED, CI_STATUS, NOTES blocks) to `.clodex/runs/<branch-slug>/iter-<padded-N>-fix.md`. Once written, the orchestrator only carries forward the one-line summary (`fix: <sha> closed <N> findings, deferred <M>`) — the full content lives on disk.

After the fixer returns:
- Verify the branch was pushed: `git rev-list @{u}..HEAD` should be empty
- Verify CI hasn't immediately failed: `gh pr checks <pr_number>` — if a check has failed, halt and surface to user. Codex will not approve a broken build; iterating is wasted budget.

Then loop back to 4a for the next iteration.

## Phase 5 — Report

Print a concise terminal-friendly summary:

```
## Clodex Complete

Task: <task>
Branch: <branch>
PR: <pr_url>
Iterations: <n> of <max_iter>   (threshold: <t>, thinking: <level>)
Final verdict: <approve | threshold-satisfied | max-iter-hit | no-diff | stalled | error>

Findings per iteration:
  1: 0c / 2h / 3m / 0l   (committed fix: <sha>)
  2: 0c / 0h / 1m / 1l   (committed fix: <sha>)
  3: 0c / 0h / 0m / 0l   (approve)

[If state.prior_runs is non-empty, append a "Prior runs (via --continue):" block
 summarizing each archived run: its verdict, iter count, threshold.]

Remaining blocking findings: <none | list of file:line + severity + title>

Pending migrations (NOT APPLIED — review and apply manually after merge):
  - supabase/migrations/20260513120000_add_draft_delete.sql
  - supabase/migrations/20260513120500_backfill_status.sql

Apply commands (run after merge, only if you've reviewed them):
  supabase db push
  # or per-file via MCP:
  # mcp__plugin_supabase_supabase__apply_migration with the file's contents

Next steps (pick one):
  • Merge — verdict is approve/threshold-satisfied and you've reviewed the diff + any pending migrations
  • /clodex --continue [--max-iter N] [--threshold ...] — push for more iterations (use when verdict is max-iter-hit and you want more rounds, OR when verdict is stalled/error and the codex broker is clear)
  • /clodex --continue --reset-review-iter --max-iter N — re-start the review counter from scratch on the existing branch+PR (use when prior orchestrator went off the rails)
  • Manual triage of N remaining findings, then /clodex --continue
  • Review + apply pending migrations (post-merge only)
```

**Never auto-merge the PR.** Merging is a shared-state action requiring explicit user authorization (see the global executing-actions-with-care rules). Leave the PR open with a passing verdict; the user merges.

If verdict is `max-iter-hit`, `needs-attention`, `stalled`, `codex-reproducible-hang`, or `error`, list the unresolved blocking findings (or the stalled job ID + last-known status) so the user can decide whether to bump `--max-iter`, accept the remaining findings, re-launch codex manually, or hand off to a human reviewer.

For `stalled` / `error` / `codex-reproducible-hang` verdicts, include explicit copy-pasteable PowerShell to check codex broker state (`node "$COMPANION_SCRIPT" status --all`) and clear stuck jobs — but the user runs these themselves. (See Hard rule 2.)

### Mandatory "Did NOT get codex approval" disclosure block

Whenever final `verdict != "approve"` AND `verdict != "threshold-satisfied"`, the report MUST include a clearly-marked disclosure block:

```
## ⚠ Did NOT receive codex approval

Final verdict: <verdict>
Codex completed for iterations: <list, or "none">
Codex stalled/errored for iterations: <list>

This run did NOT produce an independent codex sign-off on the current branch state. Do not treat this report as equivalent to a codex-approved PR. Options:

  • Run codex manually from a fresh user turn (the slash command works for human-initiated turns — it's only blocked from model auto-invocation):
      /codex:adversarial-review --base main     # default template — usually robust
  • Split the branch into smaller PRs and run /clodex on each
  • Accept the risk and merge with documented justification
  • Re-run /clodex --continue [--reset-review-iter] after the underlying issue is cleared
```

This block exists because of the v0.1 PR #78 incident: the orchestrator produced a report that said "approved" when no actual codex review had succeeded. The user (correctly) didn't trust it. v0.2's contract is that anything other than `approve` / `threshold-satisfied` carries this honest disclosure — no exceptions, no softer language.

The disclosure block is in ADDITION to the Phase 5 summary, not a replacement for it. Both appear in the same report.

## Resumability

Two distinct restart modes. Pick the right one — they have different semantics.

### `--resume`: continue an interrupted run

Use when the prior session died mid-loop (session crash, terminal close, network drop). The verdict is still null / in-progress.

1. Read `.clodex/state.json`. If missing, error: "No prior clodex run found in .clodex/state.json. Pass a task description to start fresh."
2. If state `version` is `1` (v0.1) or missing, upgrade per the v0.2 migration described in the "State file" section.
3. Refuse to proceed if the prior verdict is in any terminal-ish set (`approve`, `threshold-satisfied`, `max-iter-hit`, `stalled`, `error`, `no-diff`). Direct the user to `--continue` instead.
4. Restore: task, thinking_level, max_iter, threshold, context_tight, branch, plan_file, pr_url, iteration. Do NOT accept any of these as CLI overrides (use `--continue` for that).
5. Check out the branch: `git checkout <branch>` (only if not already on it).
6. Jump to phase based on state:
   - `branch == null` → Phase 1
   - `branch != null && pr_url == null` → Phase 3 (skip Phase 2 — implementation is already done if a branch exists; if it isn't, the user can re-run without --resume)
   - `pr_url != null && verdict in (null, "needs-attention") && iteration < max_iter` → Phase 4, next iteration is `iteration + 1`

### `--continue`: extend a finished or stalled run

Use when the prior run reached a terminal verdict (`approve`, `threshold-satisfied`, `max-iter-hit`, `stalled`, `error`, `no-diff`) but the user wants to push further with new budget or a different threshold. Also use when a prior orchestrator went off the rails and the user wants a clean review re-entry on the existing branch + PR.

1. Read `.clodex/state.json`. If missing, error like `--resume`.
2. Upgrade state version if needed.
3. Refuse to proceed if `branch == null` (no implementation exists yet — start fresh instead) or `pr_url == null` (PR was never opened — re-run without `--continue` after a fresh task, since clodex needs a PR to review).
4. Validate that the branch and PR still exist:
   - `git rev-parse --verify <branch>` — branch is still locally present
   - `gh pr view <pr_number> --json state -q .state` — PR exists. If it's `MERGED` or `CLOSED`, refuse with "PR #N is no longer open; start a fresh clodex run."
5. Check codex broker isn't stuck on the prior `last_codex_job_id`. If `node "$COMPANION_SCRIPT" status <job-id>` still reports `running` for the prior job, surface to the user and halt — do NOT auto-cancel; the user must clear it (this is the v0.1 PR #78 lesson).
6. Archive the prior run: append `{ extended_at: <now>, prior_verdict: state.verdict, prior_max_iter: state.max_iter, prior_threshold: state.threshold, findings_history: [...state.findings_history] }` to `state.prior_runs`. Clear `state.findings_history`.
7. Apply CLI overrides:
   - `--max-iter N` if passed, else default to `state.max_iter + 5` (give the run some real headroom when extending)
   - `--threshold ...` if passed, else preserve
   - thinking-level token if passed, else preserve
   - `--context-tight` flips the persisted value to true (cannot be cleared via CLI; manually edit state to unset)
8. Reset `state.verdict` to `null`, `state.last_codex_job_id` to `null`.
9. If `--reset-review-iter` is set: `state.iteration = 0`. Otherwise: `state.iteration = min(prior_iteration, new_max_iter - 1)` (so the next iteration is `prior_iteration + 1`, capped at the new `max_iter`).
10. Surface a one-line summary of the extension before re-entering the loop: `"[clodex --continue] extending iter <N> of <new_max_iter> (was <prior_iter>/<prior_max_iter>, verdict=<prior_verdict>); threshold=<t>; thinking=<level>"`.
11. Check out the branch if not already on it, then jump to Phase 4 at `iteration + 1`.

If both `--resume` and `--continue` were passed, halt with: `"--resume and --continue are mutually exclusive. Use --resume for an interrupted run (no verdict yet), --continue to extend a finished/stalled run."`

## Thinking-level handling

The thinking level affects two specific decision points; insert the corresponding phrase into your own reasoning before the work:

| Level | Phrase to insert before Phase 1 plan and Phase 4f fix synthesis |
|---|---|
| `think` | "Think about this." |
| `think hard` | "Think hard about this. Think hard." |
| `think harder` | "Think harder about this. Think harder." |
| `ultrathink` | "Ultrathink about this." |

This shapes how deep your own analysis goes before dispatching work. It does **not** change how codex reviews (codex has its own depth settings) and it does **not** change how subagents reason (they have their own system prompts).

When dispatching subagents (`clodex-focus-runner`, `clodex-findings-fixer`, or any `forge`/`executing-plans` invocation), pass the level forward as a string argument so they can use it the same way.

## Migration safety (Supabase and equivalents)

**The loop may author, edit, and commit migration files. The loop must never apply them.** Migrations modify shared state on a remote database — applying them is exactly the class of destructive, hard-to-reverse, multi-tenant-visible action that requires explicit user authorization, not autonomous-loop authorization.

### What counts as a migration

By default, treat the following as migration files (do not apply, but do let codex review them on the branch):

- `supabase/migrations/*.sql`
- `db/migrate/*.{rb,sql}`
- `prisma/migrations/**/*.sql`
- `migrations/*.sql`, `**/migrations/**/*.sql`
- Any file with frontmatter or first-line comment matching `-- migration:` / `-- supabase migration`

If the project's CLAUDE.md or `supabase/config.toml` indicates a different path, prefer that signal.

### Allowed

- Writing new migration `.sql` files.
- Editing or splitting pending migration files on the feature branch.
- Committing migration files (so they appear in the branch diff codex reviews).
- Reading the database **schema** (`supabase db dump --schema-only`, `\d` via local CLI, read-only `select` queries against a local stack) for context.
- Generating regenerated TypeScript types from a *local* Supabase stack (e.g. `supabase gen types typescript --local`) **only** if the local stack is the project default and the user's workflow already runs it.

### Forbidden inside the loop

- `supabase db push`
- `supabase migration up`, `supabase migration run`, `supabase migration repair`
- `supabase db reset` against a remote
- `psql` / `mcp__plugin_supabase_supabase__execute_sql` with write statements against a remote project
- `mcp__plugin_supabase_supabase__apply_migration`
- `mcp__plugin_supabase_supabase__create_branch`, `merge_branch`, `reset_branch` (these mutate the project)
- Any project script that wraps the above (e.g. `npm run db:push`, `pnpm migrate:deploy`) — read package.json's scripts before invoking any verification step

If a verification step (Phase 2 or Phase 4f) appears to require applying a migration to pass, **skip that step and note it**. Codex can review the migration file's SQL statically without it being applied. The loop is allowed to ship a PR whose migrations have not been applied — that's the entire point of the human-approval gate at Phase 5.

### Tracking pending migrations

On every iteration, compute the set of migration files added or modified on the feature branch vs `main`:

```bash
git diff --name-only --diff-filter=AM main...HEAD -- 'supabase/migrations/*.sql' 'db/migrate/*' 'prisma/migrations/**/*.sql' 'migrations/*.sql'
```

Persist the result in state as `pending_migrations: [<path>, ...]`. Phase 5 reads this and surfaces it.

## Failure modes and guards

- **codex background job errors.** If `node "$COMPANION_SCRIPT" result <job-id> --json` returns an error/failure status instead of valid JSON, retry the same review invocation **once**. If it fails again, halt the loop, mark verdict `needs-attention`, surface the error.
- **codex job stalls past 10 min.** Set verdict `stalled`, surface the job ID and last-known status, do not auto-relaunch — the user decides whether to investigate or retry.
- **CI fails after a fix push.** Halt loop. Codex won't approve broken builds and iterating burns budget. Surface to user with the failing check name and the latest commit SHA.
- **Push rejected (e.g. branch protection).** Halt. Surface the rejection reason from `git push`.
- **Plan vs. implementation mismatch.** If `superpowers:executing-plans` reports the plan can't be executed as written (missing files, broken assumptions), halt and surface — don't paper over.
- **Diff regrowth across iterations.** If iteration N+1 introduces files outside the original task scope, that's drift. Halt and surface — the user can decide whether to rescope.

Never:
- Use `--no-verify` to bypass hooks.
- Force-push.
- Merge the PR.
- Delete the branch.
- Discard uncommitted user changes without explicit permission.
- Apply database migrations (see "Migration safety" section).
- **Invoke the OpenAI `codex` CLI binary (`codex` / `codex review` / `codex exec`).** That's a separate tool from the codex Claude-Code plugin clodex uses. Stick to `node "$COMPANION_SCRIPT" <subcommand>` for codex calls. (Hard rule 1.)
- **Wrap `codex-companion.mjs` in a custom launcher or shell out via an intermediate script.** Call the companion script directly with `node`; nothing else in between. (Hard rule 1.)
- **Substitute Claude subagents for the codex review pass.** If codex is unavailable, halt — do not dispatch Claude agents to "fill in." (Hard rule 1.)
- **Write to `~/.claude/plugins/data/` or any other plugin's internal state.** If codex is stuck and `node "$COMPANION_SCRIPT" cancel <job-id>` doesn't clear it, surface to the user and halt. (Hard rule 2.)
- **Write the focus text inline.** Always dispatch the `clodex-focus-runner` agent for Phase 4a. (Hard rule 3.)
- **Improvise an alternative path when a prescribed step fails.** Halt and surface. (Hard rule 4.)

## Reuse map

clodex is glue. The actual work happens in these existing skills/commands — do not reimplement their logic:

| Phase | Tool |
|---|---|
| 1 (brainstorm) | `superpowers:brainstorming` |
| 1 (plan) | `superpowers:writing-plans` |
| 2 (branch) | `new-branch` (or inline `git checkout -b`) |
| 2 (implement, plan-driven) | `superpowers:executing-plans` |
| 2 (implement, plan-less) | `forge` with thinking level |
| 2 (discipline) | `superpowers:test-driven-development`, `superpowers:systematic-debugging` |
| 2 (verify) | `superpowers:verification-before-completion` |
| 3 | `ship` |
| 4a | `codex-focus` (via `clodex-focus-runner` subagent) |
| 4b/c/d | `node "$COMPANION_SCRIPT" adversarial-review \| status \| result` (codex plugin's `codex-companion.mjs`) |
| 4f | `clodex-findings-fixer` subagent |
| 5 | (own report) |

If any of these is unavailable in the current environment, surface that during pre-flight rather than mid-loop.
