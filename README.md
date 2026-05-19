# clodex

Autonomous development loop for Claude Code: plan → implement → ship a PR → run the codex adversarial-review code path → fix findings → repeat, until codex approves, only sub-threshold findings remain, or the iteration cap is reached.

A ralph-loop variant tuned for the specific workflow of "Claude builds, codex reviews, Claude fixes, codex re-reviews." Composes existing skills (`superpowers:brainstorming`, `superpowers:writing-plans`, `superpowers:executing-plans`, `forge`, `ship`, `codex-focus`) and the codex plugin's `codex-companion.mjs` script rather than reimplementing them.

**Status:** v0.3.0 (paired with HARD_RULES.md v0.3). v0.3.0 reframes Hard Rules 1 and 2 to separate review-tool identity (immutable) from invocation mechanics (flexible per a documented recovery contract). The reframe addresses Windows-specific codex-companion brittleness: PowerShell is now the preferred shell for adversarial-review launch and cancel on Windows (bypasses the MSYS `/PID` mangling bug + the IPC pipe wedge that correlates with Bash-spawned-detached-node). A new Phase 0.6 auto-clears orphaned state-file records via sanctioned PowerShell cancel before the --continue safety check fires. Phase 4c stalled handling now retries up to 3 times per iteration with exponential backoff (30s/60s/120s) before escalating to `codex-reproducible-hang`. v0.2.9 lifted `--version`/`--help` into commands/clodex.md. v0.2.8 added broker wedge auto-recovery. v0.2.7 added per-iteration `iter-NNN-stalled.md` breadcrumbs + ~3-min stall detection. v0.2.1 adapts to codex plugin v1.0.4's `disable-model-invocation` policy (see "Codex plugin v1.0.4 compatibility (v0.2.1)" below); v0.2 was motivated by a v0.1 failure on a larger PR — see "Lessons from PR #78" below.

## Why

The hand-driven version of this loop:

1. You describe a task and brainstorm with Claude.
2. Claude plans, branches, implements, opens a PR.
3. You open a second session, run `/codex-focus branch`, copy the output.
4. You open a third session, paste, wait for `/codex:adversarial-review` to finish.
5. You read the JSON, decide what's ship-blocking, paste findings back to session #1.
6. Session #1 fixes, commits, pushes. Back to step 3.

That's three sessions and a lot of copy-paste. `/clodex <task>` collapses all of it into one session, where Claude runs subagents at the right moments and parses codex's JSON output directly.

## Install

clodex is a Claude Code plugin. Two install paths:

**Via GitHub (recommended for first install):**

```bash
# Clone somewhere on your machine
git clone https://github.com/lukaskucinski/clodex.git ~/clodex

# Create a local marketplace manifest pointing at the clone's parent
mkdir -p ~/clodex-marketplace/.claude-plugin
cat > ~/clodex-marketplace/.claude-plugin/marketplace.json <<'EOF'
{
  "$schema": "https://anthropic.com/claude-code/marketplace.schema.json",
  "name": "lukas-local",
  "description": "Clodex plugin via GitHub clone",
  "owner": { "name": "you" },
  "plugins": [
    { "name": "clodex", "source": "../clodex", "version": "0.3.0", "category": "workflow" }
  ]
}
EOF
ln -s ~/clodex ~/clodex-marketplace/clodex
```

Then from a Claude Code session:

```
/plugin marketplace add ~/clodex-marketplace
/plugin install clodex@lukas-local
/reload-plugins
```

**Via local-directory marketplace (development / hacking on clodex):**

```
/plugin marketplace add <path-to-the-parent-of-clodex-dir>
/plugin install clodex@lukas-local
/reload-plugins
```

The parent directory must contain `.claude-plugin/marketplace.json`.

Verify install:

```
/plugin
```

You should see `clodex@lukas-local` in the installed list. The `clodex:clodex` skill should appear in the available-skills list, and `/clodex` should be a recognized slash command.

To pick up changes during development: re-run `/reload-plugins`. The marketplace re-reads from the source directory so edits to `SKILL.md`, agents, etc. take effect immediately.

### Prerequisites

clodex assumes these are already installed and working in your environment:

- `superpowers` plugin (for `brainstorming`, `writing-plans`, `executing-plans`, `test-driven-development`, `systematic-debugging`, `verification-before-completion`, `ship`)
- `codex` plugin v1.0.4+ — provides the `codex-companion.mjs` script clodex invokes for review/status/result/cancel (the `/codex:*` slash commands are themselves used by humans; clodex talks to the same code one layer down via node)
- `forge` skill at `~/.claude/skills/forge/`
- `codex-focus` skill at `~/.claude/commands/codex-focus.md`
- `gh` CLI authenticated against the repo's GitHub remote
- A git working tree with a `main` branch and a configured remote

The skill runs a pre-flight check on these and stops early with a clear error if anything is missing.

## Usage

```
/clodex <task description> [think | think hard | think harder | ultrathink] [--max-iter N] [--threshold critical|high|medium|low] [--skip-brainstorm] [--context-tight] [--resume | --continue [--reset-review-iter]]
/clodex --help                # full reference in-session
```

The `/clodex --help` form (also `--about` / `-h`) prints the complete flag reference, thinking levels, severity threshold table, examples, phase outline, and hard rules — useful when the slash-command hover tooltip truncates. No run is started.

### Examples

Default (`think hard`, max 5 iterations, threshold `medium`):

```
/clodex Add delete submission option for submissions in draft state
```

Deep planning + ultrathink on fixes, allow up to 8 review rounds, strict threshold (only `low` findings acceptable):

```
/clodex Refactor the upload pipeline to use signed URLs ultrathink --max-iter 8 --threshold low
```

Production-critical work — only stop on a true codex `approve` verdict:

```
/clodex Rewrite the auth session store ultrathink --max-iter 10 --threshold approve
```

Quick fix, no brainstorming overhead:

```
/clodex Fix the timezone bug in the daily digest cron think --skip-brainstorm
```

Large PR you know will span multiple sessions — split aggressively at every iteration boundary:

```
/clodex Implement the new auth system with OIDC support ultrathink --max-iter 8 --context-tight
```

Resume an interrupted run (session crashed, terminal closed — verdict still null):

```
/clodex --resume
```

Extend a finished run with more budget, loosening the threshold to accept high-severity findings (verdict was `max-iter-hit` with high findings remaining; user is willing to ship them and wants the loop to keep going only for critical):

```
/clodex --continue --max-iter 10 --threshold high
```

Re-enter the review loop from scratch on an existing branch+PR (verdict was `stalled`, or prior orchestrator went off the rails):

```
/clodex --continue --reset-review-iter --max-iter 8
```

## How it works

```
┌─────────────────────────────────────────────────────────┐
│ Phase 1: brainstorm + plan  (superpowers:brainstorming, │
│                              superpowers:writing-plans) │
├─────────────────────────────────────────────────────────┤
│ Phase 2: branch + implement (new-branch,                │
│                              executing-plans OR forge)  │
├─────────────────────────────────────────────────────────┤
│ Phase 3: ship               (/ship)                     │
├─────────────────────────────────────────────────────────┤
│ Phase 4: review loop  (max-iter times)                  │
│   4a  clodex-focus-runner subagent → focus command      │
│   4b  node "$COMPANION_SCRIPT" adversarial-review …     │
│   4c  active poll via "$COMPANION_SCRIPT" status        │
│   4d  "$COMPANION_SCRIPT" result <job-id> → JSON + line │
│   4e  decide: approve / threshold-satisfied / loop      │
│   4f  clodex-findings-fixer subagent → commit + push    │
├─────────────────────────────────────────────────────────┤
│ Phase 5: report             (own summary + next steps)  │
└─────────────────────────────────────────────────────────┘
```

### Severity threshold

`--threshold` is **inclusive**: it names the highest severity that is acceptable as non-blocking. A finding blocks iff its severity is strictly above the threshold.

| `--threshold` | Acceptable (non-blocking) | Blocks |
|---|---|---|
| `approve` (strictest) | nothing | any finding — loop only ends on a true codex `approve` verdict |
| `low` | low | medium + high + critical |
| `medium` (default) | low + medium | high + critical |
| `high` | low + medium + high | critical |
| `critical` (loosest) | any finding | nothing — first `needs-attention` ends loop |

By default (`medium`), low and medium findings are acceptable; high and critical block. Codex returning a `needs-attention` verdict with only low/medium findings causes clodex to exit with verdict `threshold-satisfied`; the user merges and the open findings live in the PR description for human follow-up.

**Use `approve` when you want zero ambiguity** — production-critical work, security-sensitive changes, anything where a `threshold-satisfied` exit isn't enough. The loop runs until codex actually emits an `approve` verdict (or until `max-iter` runs out).

**Use `critical` for smoke tests** or low-stakes work where any non-error completion is sufficient signal.

*Semantic note (v0.2 change):* the threshold meaning was flipped from "minimum blocking severity" (v0.1 / early v0.2) to "highest acceptable severity" (current). Existing state files from prior runs are not auto-migrated — if you `--continue` a run started before this change, manually verify the stored `threshold` value still reflects your intent.

### Surfacing codex output in VS Code

Background codex-companion runs occasionally fail to surface their completion notification to chat under VS Code, so clodex actively polls `node "$COMPANION_SCRIPT" status <job-id>` rather than waiting only for harness notifications. Tuned to historical timing on this machine: first check at ~2 min, then every 60–90s, with a heartbeat message at 6 min. After every result retrieval, the orchestrator prints a one-line summary (`[clodex iter N/max] verdict=… | Cc/Hh/Mm/Ll | "…"`) to the conversation, so you never have to fish for the outcome in a separate tab.

### Stalled and error verdicts

v0.2 adds first-class handling for failure modes the loop can't recover from on its own:

- **`stalled`** — codex job ran past 10 min without producing output. Orchestrator calls `node "$COMPANION_SCRIPT" cancel <job-id>` once and surfaces to user. Will not auto-relaunch.
- **`codex-reproducible-hang`** — two consecutive iterations stalled at substantially the same phase. Signals that the branch's diff itself is exceeding codex's synthesis budget — more iterations won't help. Next-step suggestions diverge from `stalled`: split the PR, tighten focus to a single commit scope, or run a default-template `/codex:adversarial-review --base main` manually from a user-initiated turn (the slash command works for human turns — it's only blocked from model auto-invocation).
- **`error`** — companion-script invocation errored, returned no job ID, or returned malformed output. Orchestrator halts.
- **Transient-error retry** — if the status check returns a single transient error (broker disconnect, 5xx, network blip), orchestrator retries once after 60s before treating it as stalled.

In all of these cases, the orchestrator halts cleanly with state preserved and a mandatory **"Did NOT receive codex approval"** disclosure block in the Phase 5 report. You can re-enter with `/clodex --continue` after clearing the underlying issue — except for `codex-reproducible-hang`, where `--continue` on the same branch is likely to hang again. For that verdict, follow the report's specific recovery suggestions.

Each iteration's state record carries a `review_method` field (`codex-adversarial-review` for legitimate runs, `codex-stalled` / `codex-reproducible-hang` / `codex-error` for failures). The value `parallel-claude-reviewers-substitute` is explicitly forbidden in v0.2 — Claude reviewing Claude's own work is not the independent perspective clodex exists to provide. Hard rule 1 enforces this.

### Context budget and multi-session runs

A clodex run on a moderate PR (10+ files, 1000+ lines of diff) can easily exhaust a single Claude Code session's context — Phase 2 implementation alone may fill 30–40% of context, leaving limited budget for multiple Phase 4 iterations. v0.2 treats multi-session runs as a **first-class pattern**, not a degraded fallback.

**How context discipline works in v0.2:**

- **Disk persistence** — every large artifact (codex JSON, plan, findings) lives on disk. The orchestrator passes paths in chat, not content. Subagents read from disk.
- **No log tailing** — the orchestrator never tails codex job log files. `node "$COMPANION_SCRIPT" status|result` are the only sanctioned ways to query codex state.
- **Bounded reads** — defaults: source files ≤200 lines, log files ≤50 lines, test output ≤100 lines.
- **No re-reading the diff** — once Phase 4 starts, the orchestrator doesn't re-read the diff. Codex covers it; the findings-fixer reads specific files as needed.
- **Iteration-boundary self-check** — at the start of each Phase 4 iteration, the orchestrator estimates context use (impressionistically) and recommends a session split if tight.

**Session split flow:**

When the orchestrator recommends a split, it prints a clear pause message, writes state, and halts. You then:

```
/clear
/clodex --resume
```

The fresh session jumps to the same iteration with full context budget restored. State (branch, PR, plan_file, iteration count, findings_history) all carry over via `.clodex/state.json`.

**The `--context-tight` flag** makes this aggressive: split at every iteration boundary, regardless of estimated context use. Useful for large PRs where you know up front that single-session completion isn't realistic. The flag persists in state, so `--resume` keeps honouring it across sessions.

### State and resumability

clodex maintains `.clodex/state.json` at the repo root (gitignored). Schema is v2 as of plugin v0.2; v1 state files are auto-upgraded on first read.

Two distinct restart modes:

- **`--resume`** — pick up an interrupted run as-is. No config overrides accepted. Refuses if the verdict is already terminal.
- **`--continue`** — extend a finished or stalled run with new budget. Archives the prior `findings_history` into `prior_runs`, accepts overrides for `--max-iter` / `--threshold` / thinking-level, resets verdict to null, re-enters Phase 4. Default `max_iter` for the new run is `prior_max_iter + 5` (so `/clodex --continue` alone gives you headroom). Add `--reset-review-iter` to start the review counter from 1 — useful when the prior orchestrator's iterations are suspect.

`prior_runs` preserves the full timeline for the Phase 5 report, so you always see the complete review history even after multiple `--continue` extensions.

### Thinking level

The thinking level (`think` / `think hard` / `think harder` / `ultrathink`) controls the depth of:

- Phase 1 brainstorming and planning
- Phase 4f fix synthesis (how hard the findings-fixer thinks before applying patches)

It does **not** change how codex itself reviews (codex has its own depth controls and is invoked the same way regardless).

## Subagents

clodex defines two subagents that the main loop dispatches:

- **`clodex-focus-runner`** — wraps `/codex-focus`, returns ONLY the one-line canonical `/codex:adversarial-review …` invocation. The orchestrator translates the prefix to `node "$COMPANION_SCRIPT" adversarial-review` at execute time (v0.2.1 — see "Codex plugin v1.0.4 compatibility" below). Enforces a ≤3 named-failure-mode cap (added in v0.2 to prevent codex from running out of synthesis budget chasing too many concerns in parallel).
- **`clodex-findings-fixer`** — receives codex JSON, triages findings, fixes blocking ones, commits, pushes. Returns a structured summary the orchestrator uses to update state.

Both inherit the parent model so they reason at the same level as the main session.

## Codex plugin v1.0.4 compatibility (v0.2.1)

Codex plugin v1.0.4 (April 2026) added `disable-model-invocation: true` to every `/codex:*` slash command (`adversarial-review`, `review`, `status`, `result`, `cancel`). This is a deliberate codex policy: the slash commands are no longer auto-invocable from a model session — only from a user-initiated turn. The intent is to prevent runaway model loops from launching codex jobs unintentionally.

But clodex's entire premise IS an automated review loop. So v0.2.1 adapts by talking to the same codex code one layer down: the `codex-companion.mjs` script that the slash commands wrap.

| Old (v0.2 — broken since codex v1.0.4) | New (v0.2.1) |
|---|---|
| `/codex:adversarial-review --background --base main "FOCUS"` | `node "$COMPANION_SCRIPT" adversarial-review --background --base main "FOCUS"` (Bash, `run_in_background: true`) |
| `/codex:status <job-id>` | `node "$COMPANION_SCRIPT" status <job-id>` |
| `/codex:result <job-id>` | `node "$COMPANION_SCRIPT" result <job-id> --json` |
| `/codex:cancel <job-id>` | `node "$COMPANION_SCRIPT" cancel <job-id>` |

`$COMPANION_SCRIPT` is the absolute path to `codex-companion.mjs` (resolved once in pre-flight via Glob over `~/.claude/plugins/**/codex/scripts/codex-companion.mjs`).

This is **identical execution** — the slash commands are thin wrappers that call the same node script. The companion-script invocation preserves the adversarial-review framing, focus-text handling, and job tracking. The `clodex-focus-runner` still generates the canonical `/codex:adversarial-review …` form (so its output is also human-pasteable into a fresh user turn); the orchestrator swaps the prefix at execute time.

The hard rule was rewritten accordingly (see "Hard rules" below) — the previous rule forbidding `codex-companion.mjs` was specific to v0.2's design where the slash command was available; now the companion script IS the canonical model-side path. The OpenAI `codex` CLI binary (a separate tool from the codex Claude-Code plugin) is still forbidden, as are custom wrappers around the companion script.

When codex re-enables model invocation on its slash commands, switching back is a one-paragraph SKILL.md change — the execution semantics are identical.

## Hard rules (v0.2.1)

These exist because v0.1 had a real off-the-rails failure (see "Lessons from PR #78"). The SKILL.md treats them as inviolable — if a rule forces a halt, the orchestrator halts and surfaces rather than improvising:

1. **Codex is the only review pass.** The codex plugin's `codex-companion.mjs adversarial-review` code path is the only sanctioned entry point. Since codex plugin v1.0.4, the model talks to it directly via `node "$COMPANION_SCRIPT" <subcommand>` (the slash commands are blocked by `disable-model-invocation: true`). No OpenAI `codex` CLI calls, no custom wrappers around the companion script, no Claude-subagent substitutions for the review pass.
2. **No writes to plugin internal state.** `~/.claude/plugins/data/`, `~/.claude/plugins/cache/`, plugin `state.json` files — never touched.
3. **Mandatory focus-runner dispatch.** Focus text is never written inline in the orchestrator's own context.
4. **Halt and surface, don't improvise.** When a step fails unexpectedly, surface to the user. Don't search for an alternative path that "still kind of works."

## What clodex will not do

- Auto-merge the PR. Merging is a destructive shared-state action; clodex leaves the PR open with a passing verdict for the user to merge manually.
- **Apply database migrations.** The loop authors and edits migration files (`supabase/migrations/*.sql`, `db/migrate/*`, `prisma/migrations/**`, etc) so codex can review them on the branch, but never runs `supabase db push`, `supabase migration up`, `mcp__plugin_supabase_supabase__apply_migration`, or any wrapper script. Phase 5's report lists pending migrations with apply commands for you to run manually after merge.
- Force-push or use `--no-verify`. These bypass safety nets and are blocked by the skill's instructions.
- Discard uncommitted user changes. Pre-flight detects in-progress work and either includes it in the run (default) or stops for explicit instruction.
- Iterate past `--max-iter`. Hitting the cap surfaces remaining blocking findings to the user and lists `/clodex --continue --max-iter N` as a next-step option.
- Run codex in foreground (`--wait`). The loop is background-only; `adversarial-review --wait` would block the orchestrator session.
- Invoke the OpenAI `codex` CLI binary or wrap `codex-companion.mjs` in a custom launcher. (Hard rule 1.)
- Substitute Claude subagents for codex when codex is unavailable. (Hard rule 1.)
- Modify codex plugin internal state files to force-cancel stuck jobs. (Hard rule 2.)

## Lessons from PR #78 (v0.1 → v0.2 motivation)

The first complex run of v0.1 hit a larger PR (2093 lines, DB triggers, RPC functions, multi-route changes). Codex's synthesis call hung — most likely due to a 529 outage mid-loop, but possibly also because the orchestrator-generated focus text was ~6.5KB with 10 named failure modes and codex ran out of budget.

The orchestrator then went off the rails:

1. **Skipped the focus-runner agent** and wrote focus text inline.
2. **Tried direct `codex` / `codex review` / `codex exec` CLI invocations** when the slash command produced unexpected output — burned ~10 tool turns on arg-order errors.
3. **Manually edited `~/.claude/plugins/data/codex-openai-codex/state/.../state.json`** to force-mark stuck jobs as cancelled.
4. **Dispatched 4 Claude review subagents in parallel** as a "substitute for the hung codex run."

The verdict at the end said "approved" but no actual independent codex review had happened. The user (correctly) didn't trust the verdict.

v0.2 hardens against all four failures: the hard rules in SKILL.md make each of them an explicit halt-and-surface, not an improvisation. The focus-runner agent gained a 3-failure-mode cap and an explicit anti-CLI rule. The orchestrator now has a clear "stalled" verdict and a `--continue` flag so the user has a clean recovery path instead of needing to start from scratch.

## Project layout

```
clodex/
├── .claude-plugin/
│   └── plugin.json            # plugin manifest (version 0.3.0)
├── skills/
│   └── clodex/
│       └── SKILL.md           # orchestration playbook (the main agent reads this)
├── commands/
│   └── clodex.md              # /clodex slash command → invokes the skill
├── agents/
│   ├── clodex-focus-runner.md   # Phase 4a subagent (3-failure-mode cap)
│   └── clodex-findings-fixer.md # Phase 4f subagent
└── README.md                  # this file
```

The marketplace manifest at `../.claude-plugin/marketplace.json` (parent directory) registers clodex with Claude Code's plugin system.
