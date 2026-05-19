# Clodex Hard Rules (v0.3)

**Authoritative source.** This file is the canonical reference for clodex's hard rules. The orchestrator reads it fresh via `Bash(cat)` at the start of every run. If the SKILL.md content already loaded in your agent context disagrees with anything below, **this file wins** — your loaded SKILL.md may be stale (Claude Code agent sessions cache skill content at session-start; `/reload-plugins` does not re-inject it).

Read these four rules once at the start of the run. Treat them as inviolable. If a rule would force you to halt, halt — do not improvise around it.

**What changed in v0.3 (paired with clodex v0.3.0):** Rules 1 and 2 are reframed to separate WHAT review tool runs (immutable) from HOW the invocation is wired (flexible per documented recovery contract). The previous v0.2.6 "no writes + narrow broker-PID-kill exception" Rule 2 formulation has been replaced with a positive contract that explicitly enumerates allowed recovery operations. Rules 3 and 4 are unchanged. This is a deliberate, version-stamped amendment — exactly the kind of rule change Rule 4 describes (rule changes are made deliberately, not improvised mid-run).

## 1. Codex is the only review pass

The codex plugin's adversarial-review code path is the ONLY sanctioned way to invoke a review.

### What this means (unchanged from v0.2.x)

The underlying call MUST be one of codex-companion.mjs's documented commands:

- `adversarial-review --background --base <ref> "FOCUS TEXT"` — launch a review
- `status <job-id>` — poll a specific job
- `status --all --json` — query broker state for all jobs
- `result <job-id> --json` — fetch a completed review's JSON output
- `cancel <job-id>` — clean up a job

The `$COMPANION_SCRIPT` variable is the absolute path to `codex-companion.mjs`, resolved once in pre-flight via `Glob ~/.claude/plugins/**/codex/scripts/codex-companion.mjs` (pick the first/only match).

If the companion script errors, returns no job ID, returns invalid JSON, or codex stalls past the polling cap AND the v0.3 retry contract (3 attempts with exponential backoff per the SKILL.md stalled-handling section) has been exhausted: halt the loop, write `verdict='stalled'`, `'codex-reproducible-hang'`, or `'error'` to state, surface to the user with the job ID and current iteration, and stop.

### What's flexible (NEW in v0.3): invocation mechanics

The SHELL used to invoke the companion script, the stdio handling, the retry policy, and any process-level recovery (e.g., killing the broker PID per the Rule 2 positive contract) are flexible. The orchestrator chooses the most reliable mechanism for the current platform. Specifically:

- On Windows, **PowerShell is the preferred shell** for codex-companion invocations (`adversarial-review`, `status`, `result`, `cancel`). PowerShell avoids the documented MSYS `/PID` path-mangling bug in `codex-companion`'s `taskkill` fallback (upstream openai/codex-plugin-cc#331) and may avoid the IPC pipe wedge pattern (#330) that Bash-spawned-detached-node correlates with.
- On Unix, Bash is the default.
- The Bash invocation path is still allowed as a fallback (with stdin closed via `< /dev/null` to defend against inherited-handle issues on Windows-detached-node specifically).
- Thin invocation helpers (inline Bash + Node one-liners; or PowerShell one-liners) that pick the appropriate shell and handle known error patterns are ALLOWED. They are NOT "wrappers" in the forbidden sense because they only normalize invocation mechanics — they do NOT change what review tool runs.

### Forbidden (unchanged)

- OpenAI `codex` CLI binary (`codex` / `codex review` / `codex exec`) — that's a separate tool from the codex Claude-Code plugin. Don't use it.
- Dispatching Claude subagents (general-purpose or specialized) as a substitute review pass. Claude reviewing Claude's own work is not the independent perspective clodex exists to provide.
- Writing your own focus text inline (Rule 3 covers this).
- Building a thick wrapper script that introduces NEW behavior beyond shell selection + stdio handling + retry — e.g., a wrapper that tries to "fix" codex output, restructure findings, or substitute its own review logic. Allowed wrappers: just choose the shell and call the same `node companion-script.mjs <command>` underneath.

### What "no wrappers" used to mean and why it's relaxed

The pre-v0.3 rule text was: *"Custom wrappers around the companion script, or shelling out via an intermediate script. Call `node "${COMPANION_SCRIPT}" <subcommand>` directly; nothing in between."*

That text conflated "no substituting a different tool" (correct) with "no helper code at all" (overconstraining). In practice, Windows requires shell-choice logic (PowerShell vs Bash) to handle MSYS bugs and IPC reliability. v0.3 keeps the first concern as Rule 1's spirit, and explicitly permits the second.

## 2. Positive contract for plugin-state operations

Instead of "no writes + exception list" (the v0.2.x formulation), Rule 2 in v0.3 enumerates the allowed plugin-state operations as a positive contract. Anything in the contract is allowed; anything not in the contract is forbidden.

### Allowed operations (positive contract)

1. **Read any plugin state file** under `~/.claude/plugins/data/` or `~/.claude/plugins/cache/`, including `broker.json`, `state.json`, and job log files. Read-only access for diagnostic purposes is always permitted.

2. **Invoke `codex-companion.mjs <command>` via any shell** (Bash, PowerShell, etc.). The `<command>` must be one of the documented commands listed in Rule 1. Shell choice is per the mechanism flexibility allowed in Rule 1.

3. **Kill the broker process PID via `powershell.exe Stop-Process -Id <pid> -Force`** in two specific contexts:
   - **Pre-flight broker wedge recovery** (SKILL.md Phase 0.5): when `status --all --json` reports running jobs whose `updatedAt` is older than 5 minutes and not produced by this clodex run. The broker is presumed wedged.
   - **Mid-iteration stall recovery** (SKILL.md Phase 4c stalled-handling): when the v0.2.7 stall-detection trips, kill the broker as part of the retry recovery cycle (up to 3 retries per iteration before escalating).

   The broker PID is read from `broker.json` (read-only access, allowed under point 1). The kill is a process operation, not a state-file write — codex-companion's `ensureBrokerSession` is designed to detect dead-broker endpoints on next call and respawn cleanly via its own internal `teardownBrokerSession` + `clearBrokerSession`. By killing the broker via Stop-Process, we trigger codex-companion's documented recovery path.

4. **Write to `.clodex/` in the current repository** (`state.json` at the top level; per-iteration artifacts under `.clodex/runs/<branch-slug>/`). This is clodex's OWN state directory; the constraint above applies to other plugins' state directories.

5. **Write to files the implementation phase legitimately edits** (Phase 2 source code edits, Phase 4f findings-fixer commits). These are the project's working files, not plugin state.

### Forbidden operations (anything not above)

- **Direct edits** to `~/.claude/plugins/data/` or `~/.claude/plugins/cache/` paths — including `state.json`, `broker.json`, job log files, or any other file under those trees. The codex-companion broker manages this state; clodex never writes there directly.
- **Killing non-broker processes** by name pattern (e.g., `Get-Process node.exe | Stop-Process`). The Rule 2 kill exception authorizes killing the broker PID specifically (read from `broker.json`). Pattern-based kills risk terminating unrelated processes (other plugins' node servers, other codex sessions, etc.).
- **Repeat the broker-kill more than 4 times per clodex run** (1 in pre-flight + up to 3 in mid-iteration retry). If all 4 attempts fail to clear the wedge, halt and surface — at that point the issue is upstream and not addressable by clodex.
- **Editing files under `~/AppData/Local/Temp/`, `~/AppData/Roaming/`, or any path outside `.clodex/` in the current workspace** that isn't part of the implementation phase's legitimate work.

### Specific anti-patterns observed in v0.1 incidents (still forbidden)

- `Stop-Process -Id <codex_pid> -Force; ...; Set-Content "...\codex-openai-codex\state\...\state.json"` — DO NOT do this. The sanctioned broker-kill is killing the broker PID ONLY; it does NOT pair with state-file edits. codex-companion cleans up its own state when the broker dies.
- Manually editing job-record JSON files under `~/.claude/plugins/data/codex-openai-codex/state/<workspace>/jobs/` — DO NOT do this. If a job record is orphaned (state says running but PID dead), invoke `cancel <job-id>` via PowerShell (sanctioned per point 2) to clear it through codex-companion's own API.
- Pattern-matching on node processes to "find and kill codex" — DO NOT do this. Read the broker PID from `broker.json` and kill that specific PID.

## 3. Always dispatch the clodex-focus-runner agent for Phase 4a

(Unchanged from v0.2.x.)

Never write the focus text inline in your own context. The agent enforces the focus-content discipline (≤3 named failure modes, trail-heads only, no boilerplate re-statement of the adversarial frame).

If you find yourself drafting `(1) Trigger… (2) Pre-submit… (3) TOCTOU…` in your own response, you are off-spec — dispatch the agent instead.

The agent returns the canonical `/codex:adversarial-review --background --base main "FOCUS"` form. **You** translate the `/codex:adversarial-review` prefix to `node "$COMPANION_SCRIPT" adversarial-review` at execute time (via the platform-appropriate shell per Rule 1). The agent never invokes codex directly; you never write focus inline.

## 4. Halt and surface, do not improvise

(Unchanged from v0.2.x, with v0.3 clarification on what counts as "unexpected.")

When any prescribed step fails in an unexpected way (slash command returns an error, an expected file does not exist, a subagent returns malformed output), the correct action is ALWAYS halt and surface. Resist the temptation to "find another way to make it work" — improvisation is exactly how v0.1 went off the rails on PR #78.

**What "unexpected" means in v0.3.** Operations explicitly listed in the Rule 2 positive contract are NOT unexpected, even when they involve recovery from failures. Example: cancel via PowerShell after a Bash cancel hit the MSYS bug is a configured fallback per Rule 1's mechanism flexibility, not improvisation. Aggressive retry on stalls (up to 3 attempts per iteration) is configured behavior, not improvisation. The orchestrator following the documented recovery contract IS the prescribed action; halt-and-surface only fires when the contract itself is exhausted (broker kill attempted 4 times, retry attempted 3 times per iteration, etc.).

**Deliberate rule changes are NOT improvisation.** The v0.2.1 adoption of the companion-script path, the v0.2.6 broker-kill exception, and v0.3's full reframe of Rules 1 and 2 are deliberate version-stamped amendments documented in this file. The distinction matters: rule changes are made deliberately and version-stamped; improvisation is mid-run pattern-matching to "anything that produces output."

If you violate any of these rules, the run is invalid regardless of how the report reads at the end.

---

**Last rule update:** v0.3 (2026-05-19) — significant reframe of Rules 1 and 2. Rule 1 now separates review-tool identity (immutable) from invocation mechanics (flexible: PowerShell preferred on Windows; thin invocation helpers allowed; shell choice + stdio handling + retry policy are NOT "wrappers" in the forbidden sense). Rule 2 replaces "no writes + narrow exception list" with a positive contract that explicitly enumerates allowed plugin-state operations. Rules 3 and 4 unchanged in substance; Rule 4 gains a clarification that operations in the Rule 2 positive contract are not "unexpected" failures. The v0.2.5 externalization (read fresh per run via Bash cat) and the v0.2.6 broker-kill provision are preserved within the new structure. Bump this version line on every rule edit; orchestrator reports the loaded HARD_RULES.md version in its pre-flight banner.
