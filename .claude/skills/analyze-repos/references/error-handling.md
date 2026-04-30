# Error handling — recovery rules, common failures, timeout protocol

This file is the canonical reference for how to react to errors at every stage of the pipeline. Read it at the start of every run and whenever an error occurs.

## The governing principle: past failures are NOT predictive

Every error in `state.json` represents a moment in time that has passed. Network conditions change, services recover, permissions get fixed, rate limits reset, MCP servers get enabled. NEVER let any historical error prevent you from attempting an operation now. What failed yesterday might work perfectly today.

## Startup verification (Phase 1, every run)

Before processing any repository:

1. **Check your current tool registry for the required MCP tool prefixes:**
   - `mcp__code-understanding__*` (always required)
   - `mcp__mermaid_image_generator__*` (always required)
   - `mcp__Conduit__*` (only if `confluence.enabled: true`)
2. **If ANY required MCP tool is missing from your registry, halt.** Write the failure to `state.json` with `operation: "mcp_preflight"`, set `success: false`, and exit. Do NOT attempt analysis using `git clone`, file reads, or any other non-MCP path. See Critical Rule 2 in the main skill.
3. If all required MCP tools are present, proceed regardless of what errors are in `state.json` from prior runs.
4. Only record NEW errors that occur during the current session.

## The MCP enable trap (the most common cause of "MCP tools unavailable")

If MCP tools you expected aren't in your registry, the most common reason is:

- The MCP servers are **registered** in `~/.claude.json` (`claude mcp list` shows ✓ Connected)
- But they are **disabled** in the per-user enable flag (visible in `/mcp` as `○ disabled`)
- Sub-agents spawned via `claude -p` inherit the disabled flag and cannot use those tools

`claude mcp list` checks reachability only. The enable flag is separate. New servers added via `claude mcp add-json` default to disabled until the user opens `/mcp` and enables them.

When this is the diagnosis, the error message in `state.json` should be specific:

```json
{
  "timestamp": "YYYY-MM-DD-HHMMSS",
  "operation": "mcp_preflight",
  "message": "mcp__code-understanding tools not in registry. Likely cause: MCP server is registered but disabled in ~/.claude.json. Fix: open /mcp inside a Claude Code session, enable the server, restart the session, then re-run."
}
```

## Specific error types and how to handle them

- **MCP tools not in registry**: halt per startup verification. Recoverable across sessions, not the current one.
- **Repository access errors** (network, permissions, deleted): record with timestamp; skip the repo and continue.
- **Repository clone/refresh fails via the MCP tool**: record the error; move on to the next repo.
- **Structure analysis fails (MCP tool returns an error)**: record the error; skip — do NOT substitute with filesystem reads.
- **Diagram generation fails (MCP tool returns an error)**: record the error; skip diagrams — do NOT generate ASCII pseudo-diagrams; do NOT embed text diagrams inline. Better no diagram than a fake one.
- **Confluence MCP tool unavailable**: record `"mcp__Conduit__* tools unavailable in this session"` verbatim. Do NOT write "parent page not found" or any other inferred result. See Critical Rule 2.
- **Sub-agent timeout**: see "Timeout protocol" below.
- **Anything else**: record as an error with a clear message; continue if possible, halt if not.

**Across all of these: never fall back to non-MCP methods.** See Critical Rule 2 in the main skill.

## Timeout protocol (sub-agent spawning)

| Attempt | Timeout | Behavior on timeout |
|---|---|---|
| 1 (initial) | 5 minutes (300000ms) | Retry once with 10-minute timeout |
| 2 (retry) | 10 minutes (600000ms) | Mark as failed, record `sub_agent_timeout` error, move on |

The retry uses the same `--model` and `--fallback-model` flags. Use Bash's timeout parameter for proper control.

Common causes: large repository size, network latency, complex analysis requirements, MCP processing delays.

## State.json error array management

1. Keep only the most recent error per operation type. Don't accumulate stale errors.
2. Always include the timestamp in `YYYY-MM-DD-HHMMSS` form.
3. **Clear the entire errors array when the corresponding operation succeeds.**
4. Errors recorded in `state.json` do NOT predict future failures — they're a record, not a forecast. The current session's success/failure is what matters.

## When error recovery means "stop the run" vs "skip this repo"

- **Stop the run entirely:** required MCP tools missing at startup; main agent itself crashes. Anything that means no repo can be processed correctly.
- **Skip this repo, continue with the next:** clone failed, analysis failed, diagram generation failed, Confluence upload failed (this repo only). Other repos may still succeed.

The summary at the end of the run must distinguish these — partial success with one failed repo is very different from total run failure.

## What to tell the user when something fails

Surface the exact `operation` and `message` from `state.json`. Don't paraphrase or summarize away the detail — they need it to fix the right thing. Example summary line:

```
acme/payments-service: FAILED — mcp_preflight: mcp__Conduit__* tools not in registry.
  Likely cause: Conduit MCP server is registered but disabled. Fix in /mcp.
```
