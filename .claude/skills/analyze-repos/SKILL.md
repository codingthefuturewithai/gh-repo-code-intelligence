---
name: analyze-repos
description: Run the multi-repository code analysis pipeline defined by config.json — for each listed GitHub repository, clone or refresh it via the code-understanding MCP, generate component/sequence/class diagrams via the mermaid_image_generator MCP, produce audience-tailored documentation (one of five analysis profiles), and (when configured) publish each repository's docs as a child page in Confluence under a parent the user specified. Trigger this skill whenever the user, working inside the gh-repo-code-intelligence repo, says any of "analyze the repos", "run the analysis", "analyze repositories according to CLAUDE.md", "process the repositories", "kick off the analysis", "generate the docs", "publish to Confluence now", "regenerate the docs", "do a fresh analysis run", or any phrasing that clearly means "execute the pipeline described in config.json against all repositories listed there." Trigger this skill even when the user does not explicitly say "analyze" — short phrases like "let's run it", "go", "do the docs", or "ship the docs to Confluence" should also fire this skill if config.json is present and the user is clearly initiating the analysis pipeline. Do NOT trigger for first-time setup (use onboard-new-user instead), for editing config.json without running, or for diagnosing a previous failed run — those are different tasks.
---

# Analyze repositories per config.json

This skill orchestrates a multi-repository code analysis pipeline. The user has a `config.json` listing one or more GitHub repositories and an analysis profile; your job is to spawn Claude sub-agents that analyze each repository via MCP tools, generate diagrams, write a `docs.md`, and (if enabled) publish to Confluence under a configured parent page.

The user's only inputs are `config.json` and the run command. Everything else happens via sub-agents you spawn.

## ⚠️ TWO CRITICAL RULES — read every time, never skip

### Rule 1: NEVER process repositories directly in the main instance

- ✅ ALWAYS spawn Claude sub-agents with `--dangerously-skip-permissions`
- ✅ Even for 1 repository — MUST use a sub-agent
- ❌ NEVER call MCP tools (`mcp__code-understanding__*`, `mcp__mermaid_image_generator__*`, `mcp__Conduit__*`) directly in the main instance
- ❌ Main instance role: ONLY spawn sub-agents, monitor `state.json`, and report. ZERO repository processing in the main instance.

**Why:** sub-agents have isolated context, parallelize cleanly, and run with permission flags the main session cannot use. Processing in the main instance causes permission failures and broken Confluence uploads.

### Rule 2: NEVER fall back to non-MCP tools if MCP tools are unavailable

If a sub-agent (or you) cannot find `mcp__code-understanding__*`, `mcp__mermaid_image_generator__*`, or `mcp__Conduit__*` (when Confluence is enabled) in the tool registry:

- ❌ DO NOT use `git clone`, `curl`, `wget`, file reads, `find`, `grep`, or any non-MCP method as a substitute
- ❌ DO NOT generate ASCII pseudo-diagrams in place of real PNGs
- ❌ DO NOT invent results from tools you never called (especially "parent page not found" when the Conduit tool was simply unavailable)
- ✅ Write the failure to `state.json` naming exactly which MCP tool was missing, set `success: false`, and exit
- ✅ When recording the error, distinguish "tool unavailable" from "tool returned not-found" — those are different problems with different fixes

**Why:** fallback output looks like partial success but bypasses the entire pipeline. The user is then misled into reviewing docs that the tool was never supposed to produce. Fail loudly so they can fix the MCP enable state (registered ≠ enabled — see `references/error-handling.md`) and re-run.

## High-level workflow — three phases

### Phase 1 — Setup (main instance only)

1. Read `config.json`. Count repositories. Note `analysis_options.analysis_profile`. Read `references/profiles.md` to understand what the chosen profile means for the analysis.
2. **Verify MCP tool availability in this session.** Confirm `mcp__code-understanding__*` and `mcp__mermaid_image_generator__*` are present in your tool registry. If `confluence.enabled: true`, also confirm `mcp__Conduit__*`. If any are missing, halt — explain to the user that the MCP servers must be **enabled** in `/mcp` (registered ≠ enabled). Do not proceed under any circumstances.
3. Read `state.json`; create it if missing per the schema in `references/state-schema.md`. **Treat all errors in state.json as historical** — past failures must NOT prevent a fresh attempt this run.
4. Convert any SSH GitHub URLs in `config.json` to HTTPS — see `references/url-conversion.md`. SSH URLs prompt for credentials and break unattended automation.
5. Generate a single timestamp (`YYYY-MM-DD-HHMMSS`) for this run's output directories.
6. Plan batches: maximum **2 sub-agents concurrent** to prevent hanging. With N repos, you'll have ⌈N/2⌉ batches.

### Phase 2 — Spawn sub-agents (main instance only)

Read `references/sub-agent-spawn.md` before constructing the spawn command. It contains:
- The exact `claude -p` flags (model, fallback model, dangerously-skip-permissions) — these are required, not optional
- The complete sub-agent prompt template covering analysis goals 0–6
- Timeout protocol (5-min initial → 10-min retry → fail)
- State coordination protocol so concurrent sub-agents don't conflict

Spawn each batch via Bash with `&` for parallel + `wait` to block until both finish. Capture exit codes; on timeout retry once with the longer timeout per `references/error-handling.md`.

### Phase 3 — Monitor and report (main instance only)

After each batch:
1. Re-read `state.json`. Each repo's entry should have `success: true` (and a `confluence_page_id` if Confluence is enabled) OR `success: false` with errors recorded.
2. Continue spawning batches until all configured repos are processed.
3. Generate a final summary listing per-repo success/failure, Confluence page titles for successes, error details for failures, total processing time, and any retry attempts.

Do **not** declare the workflow complete if any repo has `success: false`. Name the failing repos and their recorded errors in the summary so the user knows what to fix.

## Reference files — read as needed

These hold the verbatim directives. Read the relevant file at the corresponding step.

- **`references/sub-agent-spawn.md`** — exact spawn command (model flags), full sub-agent prompt template covering all 6 analysis goals, sub-agent state-update protocol, `/compact` requirement. **Read before Phase 2.**
- **`references/config-format.md`** — `config.json` schema with full example and field-by-field explanation. **Read in Phase 1.**
- **`references/state-schema.md`** — `state.json` schema, structured-error format with timestamps, the `profiles_analyzed` block, and rules for clearing errors on success vs preserving them on failure. **Read whenever creating, reading, or updating state.json.**
- **`references/profiles.md`** — the five `analysis_profile` values (`standard`, `developer_onboarding`, `architecture_review`, `business_understanding`, `operations_handover`) with audience, emphasis, and expected page count. **Read in Phase 1.**
- **`references/confluence-upload.md`** — Confluence upload workflow, page title formula (`{repo_name}{page_suffix}`), parent-page lookup, attachment handling, image-reference rules (filename-only), and re-analysis behavior. **Read whenever Confluence is enabled.**
- **`references/error-handling.md`** — error recovery rules, the principle that past errors are not predictive, MCP-enable-state diagnosis (registered ≠ enabled — the most common failure mode), specific error types, timeout/retry protocol. **Read at start of every run and whenever an error occurs.**
- **`references/url-conversion.md`** — short. SSH→HTTPS GitHub URL conversion. **Read in Phase 1 if any URL looks like SSH.**
- **`references/mcp-cache.md`** — short. The MCP code-understanding server has a cache limit and may purge older repos. **Read by sub-agents whenever a clone-or-refresh decision is being made.**

## Permissions and pre-conditions

- The main session and sub-agents run with `--dangerously-skip-permissions` — required for unattended MCP tool use.
- Sub-agents inherit permissions from the parent.
- The MCP servers `code-understanding` and `mermaid_image_generator` (and `Conduit` if Confluence is enabled) must be **enabled in `/mcp`**, not just registered. Phase 1 step 2 catches this — do not bypass that check.
- `claude --version` must work; the run command shells out to a fresh `claude -p` process per sub-agent.

## Output structure produced by a run

```
output/
└── {repo_name_with_underscores}/
    └── {YYYY-MM-DD-HHMMSS}/        <- one directory per analysis run
        ├── docs.md                  <- audience-tailored documentation
        ├── component_diagram.png    <- all diagrams in same directory as docs.md
        ├── sequence_diagram.png
        └── class_diagram.png
```

`docs.md` references diagrams by **filename only** — no `diagrams/` subdirectory, no absolute paths. Example: `![Component Diagram](component_diagram.png)`. See `references/confluence-upload.md` for the rationale (the filename has to match the attachment name on Confluence).

## When NOT to use this skill

- The user is doing first-time setup of the gh-repo-code-intelligence repo — use `onboard-new-user` instead.
- The user is editing `config.json` for a future run, not running it now.
- A prior run already failed and the user is debugging — that's a separate task; help them inspect `state.json` and `output.json`.
- The user is asking how the tool works under the hood without wanting to run it.
