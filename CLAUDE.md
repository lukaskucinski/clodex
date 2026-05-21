# CLAUDE.md — clodex project guidance

Read this before touching any clodex code. Concise; if you need depth see `skills/clodex/HARD_RULES.md` (authoritative) and `README.md` (full history).

## What clodex is

A Claude Code plugin that drives an autonomous plan → ship → review → fix loop on a feature branch. The "review" is performed by OpenAI's codex CLI (via the codex Claude-Code plugin's `codex-companion.mjs` script), NOT by another Claude instance. Clodex orchestrates; codex reviews; clodex fixes; repeat until codex approves, only sub-threshold findings remain, or the iteration cap is hit.

## Architecture: the four-layer stack

```
Layer 1 (orchestration)   /clodex slash command  → skills/clodex/SKILL.md (the loop controller)
Layer 2 (subagents)       clodex-focus-runner    → agents/clodex-focus-runner.md  (Phase 4a)
                          clodex-findings-fixer  → agents/clodex-findings-fixer.md (Phase 4f)
Layer 3 (review plumbing) node codex-companion.mjs adversarial-review (codex plugin)
Layer 4 (model call)      codex.exe app-server   → OpenAI API (codex CLI binary)
```

Clodex owns Layers 1 + 2. Layer 3 is the codex plugin (don't reimplement; just invoke it correctly). Layer 4 is the codex CLI binary on disk; clodex never touches it.

## The four hard rules (mirror of HARD_RULES.md — that file wins on conflict)

1. **Codex is the only review pass.** Underlying call MUST be `node "$COMPANION_SCRIPT" <subcommand>` via **Bash**. PowerShell is forbidden for codex-companion subcommands.
2. **Plugin-state operations (positive contract — Bash-mandatory).** Allowed: read plugin state, invoke companion script via Bash, kill broker PID via PowerShell `Stop-Process` (the ONLY PowerShell carve-out, max 2 per run), write to `.clodex/` and implementation files. Everything else forbidden.
3. **Always dispatch `clodex-focus-runner` for Phase 4a.** Never write focus text inline.
4. **Halt and surface, do not improvise.** If a prescribed step fails unexpectedly, halt — do not invent workarounds.

The full text lives in `skills/clodex/HARD_RULES.md` and is re-read fresh on every clodex run via `Bash(cat ...)`. **If you edit hard rules, edit HARD_RULES.md and bump its version line** — the inlined mirror in SKILL.md is for SKILL readers; HARD_RULES.md is what the orchestrator actually loads at runtime.

## The v0.3.0 → v0.3.1 lesson (critical — read this before "improving" anything)

**Don't do what v0.3.0 did.** v0.3.0 reframed Hard Rule 1 to allow "mechanism flexibility" — specifically, it made PowerShell the preferred shell on Windows based on a hypothesis that Bash-spawned-detached-node correlates with a Windows IPC pipe wedge. That hypothesis was **wrong**. The May 14 PR #77 production run, multiple May 18 manual `/codex:adversarial-review` Bash calls, and (now) v0.3.1's first PR #86 run all succeeded via Bash against the same broker process and codex CLI binary that PowerShell-invoked v0.3.0 calls were failing on with `400 invalid_request_error: gpt-5.2-codex not supported`.

Whatever the shell-vs-shell difference is (most likely env-var inheritance, possibly PowerShell argument quoting), it is real and **Bash is the working pattern**. PowerShell breaks it.

Concrete forbidden patterns (these were v0.3.0; v0.3.1 reverted them):

- `& "C:\Program Files\nodejs\node.exe" "$COMPANION_SCRIPT" adversarial-review ...` (PowerShell call op) — FORBIDDEN
- "Up to 3 retry attempts with exponential backoff per iteration" — FORBIDDEN (v0.2.8 single-retry is the canonical pattern)
- "PowerShell-invoked cancel bypasses the MSYS /PID bug" — FORBIDDEN as a silent auto-fallback (if Bash cancel hits MSYS, surface per Hard Rule 4; don't auto-route to PowerShell)
- "≥4 broker kills per run" — FORBIDDEN (cap is 2: 1 pre-flight + 1 mid-iteration)

PowerShell is sanctioned for EXACTLY one operation: `powershell.exe -NoProfile -Command "Stop-Process -Id <pid> -Force"` for broker-kill recovery. That's it. Nothing else.

## Why the user kept being right

Across multiple wrong diagnoses (codex CLI version, focus-size auto-routing, codex CLI version again with downgrade hypothesis), the user persistently pointed at "this used to work, what changed in clodex." The user was directionally correct every time. The variable that changed between working (May 14) and broken (today) was **inside clodex** — specifically the v0.3.0 PowerShell pivot — not in any downstream layer (codex plugin, codex CLI, OpenAI account).

**Process lesson**: when the user reports a regression on something that demonstrably worked recently, prioritize finding the minimal change between working and broken state over downstream-blame hypotheses. The simplest way: `git checkout <last-known-working-commit>` and re-test. We could have shortcut 6+ hours of wrong-hypothesis chasing by doing that on day one.

## Source of truth

- `skills/clodex/HARD_RULES.md` — authoritative hard rules; re-read fresh per run
- `skills/clodex/SKILL.md` — the orchestrator's playbook (Phase 0.5/0.6/0.7, 1–5)
- `skills/clodex/HELP.md` — user-facing /clodex --help reference
- `commands/clodex.md` — slash-command entry point (re-read fresh per invocation; safe place for --version/--help short-circuits)
- `agents/clodex-focus-runner.md` — Phase 4a subagent (focus text generation)
- `agents/clodex-findings-fixer.md` — Phase 4f subagent (triage + commit fixes)
- `.claude-plugin/plugin.json` — manifest; bump `version` on every release
- `../.claude-plugin/marketplace.json` — parent dir local-install manifest; bump in lockstep

## Entry paths

There are four ways to enter the loop. Pick the right one:

- `/clodex <task>` — fresh start. Phases 1 (plan) → 2 (implement) → 3 (ship) → 4 (review-fix) → 5 (report).
- `/clodex --resume` — interrupted run (verdict still null). Reads `.clodex/state.json`, jumps to the right phase based on what's already populated.
- `/clodex --continue` — finished or stalled run. Archives prior `findings_history` into `prior_runs`, accepts override flags (`--max-iter`, `--threshold`, thinking-level), re-enters Phase 4.
- `/clodex --from-pr [<num>]` (v0.3.2+) — mid-development PR handoff. Plan/implement/ship were done outside clodex (manual or other agent); bootstrap state from `gh pr view` and jump straight to Phase 4. PR number optional (auto-detects via current branch). Mutually exclusive with `--resume`/`--continue`/`--reset-review-iter`/`--skip-brainstorm`. Implemented in SKILL.md Phase 0.7.

## Testing changes before shipping

Don't push clodex changes without at least:

1. **Smoke test on a trivial repo first** if changing orchestration: spawn a 1-2-file scratch branch and run `/clodex` end-to-end. Verify pre-flight banner reports the new version, Bash launch fires, codex completes, findings-fixer dispatches (or doesn't, correctly) on the result.
2. **Re-read HARD_RULES.md after edits** — the inlined mirror in SKILL.md must match the canonical file. If they drift, the pre-flight `cat HARD_RULES.md` will surface the canonical version to the orchestrator at runtime, but reviewers reading SKILL.md will be confused.
3. **Bump every version reference** when shipping: `plugin.json`, `marketplace.json`, `README.md` (Status line + install snippet + project-layout reference), `SKILL.md` version banner, `HELP.md` header, `commands/clodex.md` version printf, `HARD_RULES.md` version line. Use `grep -rn "0\.X\.0" .` after the bump to catch stragglers.
4. **For Windows-touching changes**: Bash works. Whatever you're about to add that involves PowerShell — don't. Read the v0.3.0 → v0.3.1 lesson above and reconsider.

## When in doubt

The simplest pattern that works is:

```bash
node "$COMPANION_SCRIPT" adversarial-review --background --base main "FOCUS" < /dev/null
```

via the Bash tool with `run_in_background: true`. That's it. The poll loop, the result fetch, the cancel — all Bash. Don't make it more complex than that unless you have direct empirical evidence (not a hypothesis) that the complexity is needed.
