# Clodex Hard Rules (v0.3.1)

**Authoritative source.** This file is the canonical reference for clodex's hard rules. The orchestrator reads it fresh via `Bash(cat)` at the start of every run. If the SKILL.md content already loaded in your agent context disagrees with anything below, **this file wins** — your loaded SKILL.md may be stale (Claude Code agent sessions cache skill content at session-start; `/reload-plugins` does not re-inject it).

Read these four rules once at the start of the run. Treat them as inviolable. If a rule would force you to halt, halt — do not improvise around it.

**What changed in v0.3.1 (paired with clodex v0.3.1) — REVERT of v0.3.0.** The v0.3.0 "mechanism flexibility" reframe (PowerShell preferred on Windows; 3-retry aggressive recovery contract) is reverted. Evidence: a successful May 14 PR #77 production run, a successful May 18 manual `/codex:adversarial-review` run via Bash, and the same broker process today serving both successful Bash-invoked manual calls and failing PowerShell-invoked clodex calls — same script, same broker, same codex CLI binary in memory — all converge on shell choice (Bash vs PowerShell) being the load-bearing variable, not "Windows IPC pipe wedge." v0.3.1 restores the v0.2.6 Bash-mandatory invocation contract while preserving v0.3.0 Phase 0.6 orphaned-state cleanup (now Bash-invoked) and the cleaner positive-contract structure of Rule 2 (with Bash-only re-imposed as the allowed shell).

## 1. Codex is the only review pass

The codex plugin's adversarial-review code path is the ONLY sanctioned way to invoke a review.

**Since codex plugin v1.0.4 (Apr 2026)**, all `/codex:*` slash commands carry `disable-model-invocation: true`. They cannot be auto-dispatched from a model session — that's a deliberate codex policy, not a bug to route around. The sanctioned model-side path is **the same code, invoked one layer down via the companion script, via Bash**:

```
node "$COMPANION_SCRIPT" adversarial-review --background --base <ref> "FOCUS TEXT"
node "$COMPANION_SCRIPT" status <job-id>
node "$COMPANION_SCRIPT" status --all --json
node "$COMPANION_SCRIPT" result <job-id> --json
node "$COMPANION_SCRIPT" cancel <job-id>
```

`$COMPANION_SCRIPT` is the absolute path to `codex-companion.mjs`, resolved once in pre-flight via `Glob ~/.claude/plugins/**/codex/scripts/codex-companion.mjs` (pick the first/only match).

**Bash is the canonical shell** for every codex-companion invocation listed above. This was the working pattern in clodex v0.2.x through v0.2.9, validated end-to-end in the May 14 PR #77 production run. The single exception is `Stop-Process` for broker-kill recovery, which is sanctioned PowerShell per Rule 2 (the Bash `taskkill /PID` fallback hits a known MSYS path-mangling bug in Git Bash).

The launch invocation must close inherited stdin (`< /dev/null`) as a defensive measure against detached-node stdio inheritance patterns on Windows. This is the one durable lesson from the v0.3.0 experiment.

If the companion script errors, returns no job ID, returns invalid JSON, or codex stalls past the polling cap AND the v0.2.8 single-retry broker-recovery cycle has been attempted: halt the loop, write `verdict='stalled'`, `'codex-reproducible-hang'`, or `'error'` to state, surface to the user with the job ID and current iteration, and stop.

**Forbidden:**
- OpenAI `codex` CLI binary (`codex` / `codex review` / `codex exec`) — that's a separate tool from the codex Claude-Code plugin. Don't use it.
- PowerShell for invoking `codex-companion.mjs` subcommands (`adversarial-review`, `status`, `result`, `cancel`). v0.3.0 tried this; it broke the working pattern. PowerShell is allowed ONLY for the narrow `Stop-Process` carve-out in Rule 2.
- Custom wrappers around the companion script, or shelling out via an intermediate script. Call `node "$COMPANION_SCRIPT" <subcommand>` directly under Bash; nothing in between.
- Dispatching Claude subagents (general-purpose or specialized) as a substitute review pass. Claude reviewing Claude's own work is not the independent perspective clodex exists to provide.
- Writing your own focus text inline and passing it to any entry point other than what `clodex-focus-runner` produces. (See Rule 3.)

If `/codex:adversarial-review` ever flips back to `disable-model-invocation: false`, switching back to slash-command dispatch is fine — it's functionally identical to the companion-script call. Until then, the companion-script path via Bash IS the canonical model-side one.

## 2. Plugin-state operations (positive contract — Bash-mandatory)

Instead of "no writes + exception list" (the v0.2.x formulation), Rule 2 enumerates allowed plugin-state operations as a positive contract. Anything in the contract is allowed; anything not in the contract is forbidden. **All shell invocations in the contract use Bash, with one narrow PowerShell carve-out for `Stop-Process`.**

### Allowed operations

1. **Read any plugin state file** under `~/.claude/plugins/data/` or `~/.claude/plugins/cache/`, including `broker.json`, `state.json`, and job log files. Read-only access for diagnostic purposes is always permitted.

2. **Invoke `codex-companion.mjs <command>` via Bash.** The `<command>` must be one of the documented commands listed in Rule 1. Bash is the only allowed shell for these invocations.

3. **Kill the broker process PID via `powershell.exe -NoProfile -Command "Stop-Process -Id <pid> -Force"`** in two specific contexts (this is the narrow PowerShell carve-out):
   - **Pre-flight broker wedge recovery** (SKILL.md Phase 0.5): when `status --all --json` reports running jobs whose `updatedAt` is older than 5 minutes and not produced by this clodex run. The broker is presumed wedged.
   - **Mid-iteration stall recovery** (SKILL.md Phase 4c stalled-handling): when the v0.2.7 stall-detection trips for the first time in a run, kill the broker once as part of the single-retry recovery cycle.

   The broker PID is read from `broker.json` (read-only access, allowed under point 1). The kill is a process operation, not a state-file write — codex-companion's `ensureBrokerSession` is designed to detect dead-broker endpoints on next call and respawn cleanly via its own internal `teardownBrokerSession` + `clearBrokerSession`. By killing the broker via Stop-Process (which the Bash `taskkill /PID` fallback in codex-companion cannot do reliably on Windows due to MSYS `/PID` mangling), we trigger codex-companion's documented recovery path.

   **Cap: at most twice per clodex run** (once pre-flight + once mid-iteration if the stall fires). If both attempts fail to clear the wedge, halt and surface — at that point the issue is upstream and not addressable by clodex.

4. **Write to `.clodex/` in the current repository** (`state.json` at the top level; per-iteration artifacts under `.clodex/runs/<branch-slug>/`). This is clodex's OWN state directory; the constraint above applies to other plugins' state directories.

5. **Write to files the implementation phase legitimately edits** (Phase 2 source code edits, Phase 4f findings-fixer commits). These are the project's working files, not plugin state.

### Forbidden operations

- **Direct edits** to `~/.claude/plugins/data/` or `~/.claude/plugins/cache/` paths — including `state.json`, `broker.json`, job log files, or any other file under those trees. The codex-companion broker manages this state; clodex never writes there directly.
- **PowerShell for any codex-companion subcommand invocation** (`adversarial-review`, `status`, `result`, `cancel`). PowerShell's sole sanctioned use is `Stop-Process` for the broker-kill carve-out above.
- **Killing non-broker processes** by name pattern (e.g., `Get-Process node.exe | Stop-Process`). The Rule 2 kill exception authorizes killing the broker PID specifically (read from `broker.json`). Pattern-based kills risk terminating unrelated processes (other plugins' node servers, other codex sessions, etc.).
- **Repeat the broker-kill more than twice per clodex run** (1 pre-flight + 1 mid-iteration). v0.3.0's "up to 4 kills per run with 3 retries" expansion is reverted — back to v0.2.6's twice-per-run cap.
- **Editing files under `~/AppData/Local/Temp/`, `~/AppData/Roaming/`, or any path outside `.clodex/` in the current workspace** that isn't part of the implementation phase's legitimate work.

### Specific anti-patterns (still forbidden)

- `Stop-Process -Id <codex_pid> -Force; ...; Set-Content "...\codex-openai-codex\state\...\state.json"` — DO NOT do this. The sanctioned broker-kill is killing the broker PID ONLY; it does NOT pair with state-file edits. codex-companion cleans up its own state when the broker dies.
- Manually editing job-record JSON files under `~/.claude/plugins/data/codex-openai-codex/state/<workspace>/jobs/` — DO NOT do this. If a job record is orphaned (state says running but PID dead), invoke `cancel <job-id>` via Bash (sanctioned per point 2) to clear it through codex-companion's own API. If the Bash cancel hits the MSYS bug, surface the failure per Rule 4 — do not silently switch to PowerShell invocation of cancel.
- Pattern-matching on node processes to "find and kill codex" — DO NOT do this. Read the broker PID from `broker.json` and kill that specific PID.

## 3. Always dispatch the clodex-focus-runner agent for Phase 4a

(Unchanged from v0.2.x.)

Never write the focus text inline in your own context. The agent enforces the focus-content discipline (≤3 named failure modes, trail-heads only, no boilerplate re-statement of the adversarial frame).

If you find yourself drafting `(1) Trigger… (2) Pre-submit… (3) TOCTOU…` in your own response, you are off-spec — dispatch the agent instead.

The agent returns the canonical `/codex:adversarial-review --background --base main "FOCUS"` form. **You** translate the `/codex:adversarial-review` prefix to `node "$COMPANION_SCRIPT" adversarial-review` at execute time. The agent never invokes codex directly; you never write focus inline.

## 4. Halt and surface, do not improvise

When any prescribed step fails in an unexpected way (slash command returns an error, an expected file does not exist, a subagent returns malformed output), the correct action is ALWAYS halt and surface. Resist the temptation to "find another way to make it work" — improvisation is exactly how v0.1 went off the rails on PR #78.

**What "unexpected" means.** Operations explicitly listed in the Rule 2 positive contract are NOT unexpected, even when they involve recovery from failures. Example: PowerShell `Stop-Process` for the broker-kill carve-out after a Bash cancel hit the MSYS bug is a configured fallback, not improvisation. The v0.2.8 single-retry pattern (kill broker once + retry once before escalating to `codex-reproducible-hang`) is configured behavior. Beyond that, halt and surface.

**Deliberate rule changes are NOT improvisation.** The v0.2.1 adoption of the companion-script path, the v0.2.6 broker-kill exception, the v0.3.0 experiment with PowerShell-preferred shell, and v0.3.1's revert of that experiment are all deliberate version-stamped amendments documented in this file. The distinction matters: rule changes are made deliberately and version-stamped; improvisation is mid-run pattern-matching to "anything that produces output."

If you violate any of these rules, the run is invalid regardless of how the report reads at the end.

---

**Last rule update:** v0.3.1 (2026-05-19) — REVERT of v0.3 mechanism-flexibility reframe. Rule 1 restored to Bash-mandatory invocation (with `< /dev/null` stdin closure as the one preserved defense-in-depth measure from v0.3.0). Rule 2 keeps v0.3's positive-contract structure but re-imposes Bash-only for codex-companion invocations; PowerShell `Stop-Process` carve-out narrowed back to v0.2.6's twice-per-run cap. Rules 3 and 4 unchanged in substance. The v0.3.0 retry-3 / 4-kill expansion is reverted. v0.2.5 externalization, v0.2.6 broker-kill provision, and v0.3.0 Phase 0.6 orphaned-state cleanup (now Bash-invoked) are all preserved. Bump this version line on every rule edit; orchestrator reports the loaded HARD_RULES.md version in its pre-flight banner.
