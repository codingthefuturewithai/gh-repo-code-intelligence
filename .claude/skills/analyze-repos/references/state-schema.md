# state.json schema and management rules

`state.json` lives at the repo root and tracks the pipeline's per-repository state. Create it in Phase 1 if it doesn't exist; otherwise read it.

## The most important rule: state.json is a HISTORICAL LOG, not a predictor of failures

- Errors recorded in `state.json` represent moments in time, NOT permanent conditions.
- **Past failures must NOT prevent future attempts.** Network conditions change, services recover, permissions get fixed, rate limits reset.
- Every run is a fresh start — assume all previous issues have been resolved unless proven otherwise *in the current session*.

## Per-repository entry shape

```json
{
  "org/repo-name": {
    "cloned": false,
    "last_updated": null,
    "last_analysis": null,
    "docs_generated": null,
    "diagrams_generated": false,
    "confluence_uploaded": null,
    "confluence_page_id": null,
    "errors": [],
    "processing_started": null,
    "processing_agent": null,
    "success": false,
    "profiles_analyzed": {
      "standard": {
        "timestamp": null,
        "confluence_page_id": null,
        "confluence_page_title": null,
        "docs_generated": null
      }
    }
  }
}
```

## Field semantics

- `cloned` (boolean): whether the MCP cache currently has this repo. Trust but verify — cache may have purged it (see `references/mcp-cache.md`).
- `last_updated`, `last_analysis`, `docs_generated`, `confluence_uploaded`: timestamps in `YYYY-MM-DD-HHMMSS` format.
- `diagrams_generated` (boolean): true after all preferred diagrams successfully generated.
- `confluence_page_id` (string|null): set by sub-agent after successful upload.
- `errors` (array of structured-error objects): see "Error format" below.
- `processing_started` (string|null): timestamp when a sub-agent claimed this repo. Used as a soft lock — agents check this to avoid races.
- `processing_agent` (string|null): unique id of the sub-agent currently processing (e.g., `claude_sub_agent_1`).
- `success` (boolean): the bottom-line outcome flag. `true` only if every required step (clone/refresh, analyze, diagrams, docs, optional Confluence) succeeded this run.
- `profiles_analyzed` (object): keyed by profile name. Each profile gets its own entry recording when it was last produced and where its Confluence page lives. This lets the same repo produce multiple profile-specific pages without one overwriting another's tracking.

## Structured error format

Every entry in `errors[]` MUST be an object, not a bare string:

```json
{
  "timestamp": "YYYY-MM-DD-HHMMSS",
  "operation": "clone | refresh | analyze | mcp_preflight | confluence_upload | sub_agent_timeout | ...",
  "message": "Human-readable description of what failed and why"
}
```

Examples:

```json
{
  "timestamp": "2025-07-04-143022",
  "operation": "sub_agent_timeout",
  "message": "Sub-agent timed out after 10 minutes (600000ms) - repository analysis incomplete"
}
```

```json
{
  "timestamp": "2025-07-04-143022",
  "operation": "mcp_preflight",
  "message": "mcp__code-understanding tools not available in this session — sub-agent cannot run analysis"
}
```

```json
{
  "timestamp": "2025-07-04-143022",
  "operation": "confluence_upload",
  "message": "Failed to upload to Confluence: parent page 'X' not found in space DEMO"
}
```

## Error array management rules

1. Keep only the most recent error per operation type. Don't accumulate stale errors.
2. Always include the timestamp.
3. **Clear the entire errors array when the operation succeeds.** Don't leave old failures in place after a successful run — they confuse future readers and the rule above (past errors are historical) means they shouldn't influence future attempts anyway.
4. On failure, record the new error and leave any pre-existing fields (`cloned`, `last_updated`, etc.) as they were before this run.

## Recovery behavior — read this if state.json has errors when the run starts

- **Ignore all past errors regardless of type** — they're history, not current reality.
- **Attempt every operation** as if it's the first time.
- Common temporary failures that MUST be retried this run:
  - "MCP tools not available"
  - "Network timeout"
  - "Rate limit exceeded"
  - "Permission denied"
  - "Repository not found"
  - "Clone failed"
  - Any other error from a previous run
- Success in the current session should clear all relevant errors and update status fields accordingly.
- The only failures that matter are the ones happening **right now**.

## profiles_analyzed coordination

After a successful Confluence upload, the sub-agent updates `profiles_analyzed[current_profile]` with:

```json
{
  "timestamp": "YYYY-MM-DD-HHMMSS",
  "confluence_page_id": "12345",
  "confluence_page_title": "org/repo-name - Architecture Review",
  "docs_generated": "YYYY-MM-DD-HHMMSS"
}
```

This block lets the same repo carry multiple profile-specific page records simultaneously — useful when the user runs the pipeline against the same repo with different profiles (e.g., `developer_onboarding` AND `architecture_review`).

## Initial creation (Phase 1)

If `state.json` doesn't exist when Phase 1 starts, create it with one entry per repo from `config.json`, all fields set to their default-empty values shown in the schema above. Write it before spawning any sub-agents.
