---
name: clodex
description: |
  Autonomous dev loop: plan → branch → implement → ship PR → /codex:adversarial-review → fix → repeat. Halts on approve, threshold-satisfied, or max-iter hit. Never auto-merges, never applies migrations.

  FLAGS
    --max-iter N           iteration cap                                   (default: 5)
    --threshold X          highest acceptable severity: approve|low|medium|high|critical  (default: medium)
    --skip-brainstorm      skip Phase 1 plan + brainstorm; use forge inline
    --context-tight        recommend session split at every Phase 4 iteration boundary
    --from-pr [<num>]      bootstrap state from existing PR and jump to Phase 4 (skips Phases 1-3)
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

    # Already shipped a PR outside clodex — just want the review-fix loop on it
    /clodex --from-pr 86 --max-iter 5 --threshold medium

    # Same, with explicit task description (overrides PR title)
    /clodex --from-pr 86 "Read-only viewer admin role" --max-iter 5 --threshold medium

    # Continue a finished run, loosening threshold (accept high+medium+low)
    /clodex --continue --max-iter 10 --threshold high

    # Re-enter review loop on existing branch+PR after off-rails iter
    /clodex --continue --reset-review-iter --max-iter 5

    # Resume an interrupted run as-is
    /clodex --resume

  STATE  .clodex/state.json (gitignored, v2 schema). Resumable + extensible across sessions.
  HELP   /clodex --help   (full reference in-session — use when this tooltip truncates)
  TRIGGERS  /clodex <task>, "ralph this", "run the codex loop on X".
argument-hint: "<task description> [think|think hard|think harder|ultrathink] [--max-iter N] [--threshold approve|low|medium|high|critical] [--skip-brainstorm] [--context-tight] [--from-pr [<num>] [<task description>]] [--resume | --continue [--reset-review-iter]] | --help | --version"
allowed-tools: Skill, TaskCreate, TaskUpdate, TaskList, TaskGet, Read, Write, Edit, Glob, Grep, Bash, Agent
---

# /clodex entry point (v0.3.2 — `--from-pr` flag for mid-development PR handoff)

Process `$ARGUMENTS` in this order, taking the FIRST applicable branch. Branches 1 and 2 are handled directly at this slash-command level (never invoke the clodex skill for them) so they always reflect the currently-installed plugin version, even in long-running Claude Code sessions whose SKILL.md cache predates the relevant flag. Slash command bodies are re-read from disk per invocation; skill content is cached at session start.

## 1. Version short-circuit (handled here, never delegated to the skill)

If `$ARGUMENTS` contains any of `--version`, `-v`, or `-V` as a whole token (anywhere in the args, any order), do EXACTLY this and stop:

```
Bash(command: 'printf "clodex@lukas-local v0.3.2\n"', description: "Print clodex version")
```

The Bash tool's stdout is the user-visible response. Emit it as-is, with no preamble, no commentary, no surrounding markdown. Do NOT invoke the `clodex` skill. Do NOT add explanatory text. Stop after the Bash output appears.

If multiple meta-flags are present (e.g. `--help --version`), version wins — it's the cheapest to emit and the user almost certainly wants the version.

**Why this lives here, not in SKILL.md:** Claude Code re-reads commands/*.md per slash command invocation, but SKILL.md is loaded once at session start and cached. Putting --version handling in the skill means a session that started before --version was added (pre-v0.2.2) would not recognize the flag. Putting it here makes the version flag bullet-proof across upgrades, even for long-running VS Code Claude Code panels.

## 2. Help short-circuit (handled here, never delegated to the skill)

If `$ARGUMENTS` contains any of `--help`, `--about`, or `-h` as a whole token (and no version flag — that case is handled by branch 1), do EXACTLY this and stop:

1. Resolve the HELP.md path. If `$CLAUDE_PLUGIN_ROOT` is set, the path is `${CLAUDE_PLUGIN_ROOT}/skills/clodex/HELP.md`. Otherwise resolve via `Glob ~/.claude/plugins/**/clodex/skills/clodex/HELP.md` and pick the first match (there should be exactly one).
2. Run `Bash(command: 'cat "<resolved-path>"', description: "Print clodex help")`.
3. The Bash tool's stdout is the user-visible response. Do NOT transform, summarize, paraphrase, re-wrap, or rewrite. The HELP.md file is the canonical reference.

Do NOT invoke the `clodex` skill. Stop after the cat output appears.

**Same rationale as branch 1:** this lives here so the help block always reflects the currently-installed HELP.md, even when SKILL.md's loaded content is stale.

## 3. Default: invoke the clodex skill with $ARGUMENTS

For any other arguments (or no arguments — the skill will prompt for a task), invoke the `clodex` skill via the `Skill` tool, passing `$ARGUMENTS` verbatim.

The skill defines four hard rules (codex-only review pass, no plugin-state writes with v0.2.6 broker-kill exception, mandatory focus-runner dispatch, halt-and-surface over improvise). The skill reads them fresh from HARD_RULES.md at the start of every run.

Do not auto-merge the PR. End with a Phase 5 report that surfaces continuation commands the user can copy-paste if they want more iterations.

Maintain `.clodex/state.json` throughout so the run is resumable via `/clodex --resume` (interrupted run) or extensible via `/clodex --continue` (finished or stalled run). `/clodex --from-pr [<num>]` is the entry path for PRs already shipped outside clodex — the skill's Phase 0.7 bootstraps state from `gh pr view` and jumps directly to Phase 4.
