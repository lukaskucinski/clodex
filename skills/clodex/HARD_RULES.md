# Clodex Hard Rules (v0.2.6)

**Authoritative source.** This file is the canonical reference for clodex's hard rules. The orchestrator reads it fresh via `Bash(cat)` at the start of every run. If the SKILL.md content already loaded in your agent context disagrees with anything below, **this file wins** — your loaded SKILL.md may be stale (Claude Code agent sessions cache skill content at session-start; `/reload-plugins` does not re-inject it).

Read these four rules once at the start of the run. Treat them as inviolable. If a rule would force you to halt, halt — do not improvise around it.

## 1. Codex is the only review pass

The codex plugin's adversarial-review code path is the ONLY sanctioned way to invoke a review.

**Since codex plugin v1.0.4 (Apr 2026)**, all `/codex:*` slash commands carry `disable-model-invocation: true`. They cannot be auto-dispatched from a model session — that's a deliberate codex policy, not a bug to route around. The sanctioned model-side path is **the same code, invoked one layer down via the companion script**:

```
node "$COMPANION_SCRIPT" adversarial-review --background --base <ref> "FOCUS TEXT"
node "$COMPANION_SCRIPT" status <job-id>
node "$COMPANION_SCRIPT" result <job-id> --json
node "$COMPANION_SCRIPT" cancel <job-id>
```

`$COMPANION_SCRIPT` is the absolute path to `codex-companion.mjs`, resolved once in pre-flight via `Glob ~/.claude/plugins/**/codex/scripts/codex-companion.mjs` (pick the first/only match).

If the companion script errors, returns no job ID, returns invalid JSON, or codex stalls past the polling cap: halt the loop, write `verdict='stalled'` (or `'error'`) to state, surface to the user with the job ID and current iteration, and stop.

**Forbidden:**
- OpenAI `codex` CLI binary (`codex` / `codex review` / `codex exec`) — that's a separate tool from the codex Claude-Code plugin.
- Custom wrappers around the companion script, or shelling out via an intermediate script. Call `node "$COMPANION_SCRIPT" <subcommand>` directly; nothing in between.
- Dispatching Claude subagents (general-purpose or specialized) as a substitute review pass. Claude reviewing Claude's own work is not the independent perspective clodex exists to provide.
- Writing your own focus text inline and passing it to any entry point other than what `clodex-focus-runner` produces. (See Rule 3.)

If `/codex:adversarial-review` ever flips back to `disable-model-invocation: false`, switching back to slash-command dispatch is fine — it's functionally identical to the companion-script call. Until then, the companion-script path IS the canonical model-side one.

## 2. Never write to plugin internal state

Specifically forbidden:
- `~/.claude/plugins/data/` — any path (WRITES)
- `~/.claude/plugins/cache/` — any path (WRITES)
- Plugin `state.json` / `broker.json` / job log files (WRITES)
- Any path under another plugin's install tree (WRITES)

**Reads are allowed.** Reading `broker.json` to extract metadata (PID, endpoint), or reading any other plugin state file for diagnostic purposes, is not a violation. Only file writes/modifications to those paths are forbidden.

If a codex job is stuck and `node "$COMPANION_SCRIPT" cancel <job-id>` does not clear it within 30 seconds, proceed to the broker-kill recovery path (see exception below); if that also fails, halt and surface — let the user clean up manually. Forcing the state via direct file-edits can corrupt future codex runs.

**Positive form:** clodex writes ONLY to `.clodex/` in the current repository (state.json at the top level; per-iteration artifacts under `.clodex/runs/<branch-slug>/`), and to any files the implementation phase legitimately edits. Anything under `~/.claude/plugins/`, `~/AppData/Local/Temp/`, or any other plugin's data directory is OFF-LIMITS for writes.

If you find yourself drafting a `Set-Content`, `Out-File`, or any other write that touches one of those paths, stop — that's the rule firing.

### Exception: sanctioned broker PID kill (v0.2.6+)

Killing the codex-companion broker process by PID is **explicitly allowed** in two specific contexts:

1. **Pre-flight broker wedge recovery** (clodex Phase 0.5): when `status --all --json` reports one or more `running` jobs whose `updatedAt` is older than 5 minutes AND not produced by this clodex run, the broker is presumed wedged. Read `broker.json` (read-only) to obtain `broker.pid`, then invoke `powershell.exe -NoProfile -Command "Stop-Process -Id <pid> -Force"`.

2. **Mid-iteration stall recovery** (clodex Phase 4c stalled-handling, step 3): when the v0.2.7 stall-detection trips for the first time in a run, perform the same broker-kill sequence, then retry the same iteration's focus once.

The rationale: codex-companion's `ensureBrokerSession` is designed to handle broker death cleanly — on the next `node "$COMPANION_SCRIPT" <subcommand>` call, it detects the dead endpoint, tears down stale state files itself (via its own internal `teardownBrokerSession` + `clearBrokerSession`), and spawns a fresh broker. Killing the broker PID is a **process operation**, not a state-file write; it triggers codex-companion's documented recovery path rather than corrupting it.

**Still forbidden under this exception:**
- Direct edits to `broker.json`, `state.json`, or any job log file under `~/.claude/plugins/data/` (write operations remain off-limits — the kill works because codex-companion cleans up its OWN state on detection of broker death).
- Killing arbitrary node processes that aren't the broker (use the broker PID from `broker.json` specifically; do not kill by name pattern, which could terminate other processes).
- Retrying the broker-kill more than twice per clodex run (once in pre-flight, once mid-iteration if stall fires). If both attempts fail to clear the wedge, halt and surface.

The exception's scope is narrow and version-stamped. Any other proposed "clean-up" of plugin state is improvisation (Rule 4 violation).

## 3. Always dispatch the clodex-focus-runner agent for Phase 4a

Never write the focus text inline in your own context. The agent enforces the focus-content discipline (≤3 named failure modes, trail-heads only, no boilerplate re-statement of the adversarial frame).

If you find yourself drafting `(1) Trigger… (2) Pre-submit… (3) TOCTOU…` in your own response, you are off-spec — dispatch the agent instead.

The agent returns the canonical `/codex:adversarial-review --background --base main "FOCUS"` form. **You** translate the `/codex:adversarial-review` prefix to `node "$COMPANION_SCRIPT" adversarial-review` at execute time. The agent never invokes codex directly; you never write focus inline.

## 4. Halt and surface, do not improvise

When any prescribed step fails in an unexpected way (slash command returns an error, an expected file does not exist, a subagent returns malformed output), the correct action is ALWAYS halt and surface. Resist the temptation to "find another way to make it work" — improvisation is exactly how v0.1 went off the rails on PR #78.

**Exception:** the v0.2.1 adoption of the companion-script path was NOT improvisation — it was a deliberate rule change documented in this file. The v0.2.6 broker-kill exception under Rule 2 is the same kind of deliberate amendment, not a license to invent further escapes. The distinction matters: rule changes are made deliberately and version-stamped; improvisation is mid-run pattern-matching to "anything that produces output."

If you violate any of these rules, the run is invalid regardless of how the report reads at the end.

---

**Last rule update:** v0.2.6 (2026-05-19) — added the broker-kill exception under Rule 2 (paired with clodex v0.2.8's broker wedge auto-recovery). Read-only access to `broker.json` is now explicitly allowed; killing the broker PID via `Stop-Process` is sanctioned in the two narrow contexts documented above. The v0.2.5 baseline (externalized hard rules to this file) is preserved. Bump this version line on every rule edit; orchestrator reports the loaded HARD_RULES.md version in its pre-flight banner.
