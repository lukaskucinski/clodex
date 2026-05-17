---
name: clodex
description: |
  Autonomous dev loop: plan → branch → implement → ship PR → /codex:adversarial-review → fix → repeat. Halts on approve, threshold-satisfied, or max-iter hit. Never auto-merges, never applies migrations.

  FLAGS
    --max-iter N           iteration cap                                   (default: 5)
    --threshold X          highest acceptable severity: approve|low|medium|high|critical  (default: medium)
    --skip-brainstorm      skip Phase 1 plan + brainstorm; use forge inline
    --context-tight        recommend session split at every Phase 4 iteration boundary
    --resume               continue an interrupted run (no config overrides)
    --continue             extend a finished/stalled run (accepts override flags)
    --reset-review-iter    with --continue, reset iteration counter to 0
    --help | --about | -h  print full reference in-session (no run started)
    --version | -v | -V    print plugin version line and exit

  THINKING LEVELS  (passed as bare token, controls Phase 1 plan + Phase 4f fix depth)
    think | think hard | think harder | ultrathink                         (default: think hard)

  EXAMPLES
    # New task, defaults (think hard, max 5, threshold medium)
    /clodex Add delete option for submissions in draft state

    # New task, deeper thinking + stricter threshold (only low findings acceptable) + 8 iters
    /clodex Refactor upload pipeline to use signed URLs ultrathink --max-iter 8 --threshold low

    # Production-critical — only stop on a true codex approve verdict
    /clodex Rewrite auth session store ultrathink --max-iter 10 --threshold approve

    # Quick fix, no brainstorm
    /clodex Fix the timezone bug in the daily digest cron think --skip-brainstorm

    # Large PR, plan for multi-session
    /clodex Implement OIDC auth ultrathink --max-iter 8 --context-tight

    # Continue a finished run, loosening threshold (accept high+medium+low)
    /clodex --continue --max-iter 10 --threshold high

    # Re-enter review loop on existing branch+PR after off-rails iter
    /clodex --continue --reset-review-iter --max-iter 5

    # Resume an interrupted run as-is
    /clodex --resume

  STATE  .clodex/state.json (gitignored, v2 schema). Resumable + extensible across sessions.
  HELP   /clodex --help   (full reference in-session — use when this tooltip truncates)
  TRIGGERS  /clodex <task>, "ralph this", "run the codex loop on X".
argument-hint: "<task description> [think|think hard|think harder|ultrathink] [--max-iter N] [--threshold approve|low|medium|high|critical] [--skip-brainstorm] [--context-tight] [--resume | --continue [--reset-review-iter]] | --help | --version"
allowed-tools: Skill, TaskCreate, TaskUpdate, TaskList, TaskGet, Read, Write, Edit, Glob, Grep, Bash, Agent
---

Invoke the `clodex` skill with the following arguments: `$ARGUMENTS`

If `$ARGUMENTS` contains `--help`, `--about`, or `-h`, the skill emits the verbatim help/about block and stops — no pre-flight, no state read, no run. Otherwise, follow the full playbook phase-by-phase, dispatching the `clodex-focus-runner` and `clodex-findings-fixer` subagents at Phase 4a and 4f. Maintain `.clodex/state.json` throughout so the run is resumable via `/clodex --resume` (interrupted run) or extensible via `/clodex --continue` (finished or stalled run).

The skill defines four hard rules (codex-only review pass, no plugin-state writes, mandatory focus-runner dispatch, halt-and-surface over improvise). Read them at the start of the run and treat as inviolable.

Do not auto-merge the PR. End with a Phase 5 report that surfaces continuation commands the user can copy-paste if they want more iterations.
