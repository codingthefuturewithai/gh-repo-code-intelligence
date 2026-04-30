# Sub-agent spawn — model flags, prompt template, analysis goals

This is the verbatim spec for spawning Claude sub-agents from the main instance. The main instance does not analyze repos itself — every repository is processed by a sub-agent spawned via `claude -p`.

## Required spawn flags (never omit, never substitute)

Every sub-agent spawn command MUST include all three of these:

- `--model claude-sonnet-4-6` — primary model
- `--fallback-model claude-haiku-4-5-20251001` — used only when Sonnet is overloaded
- `--dangerously-skip-permissions` — required for unattended MCP tool use

**Rationale:** the main agent runs on whatever the user has configured (typically Opus). Sub-agents do not need that level of capability for this workflow — Sonnet 4.6 handles multi-step MCP orchestration and 15–30 page audience-tailored documentation reliably while running significantly faster and at materially lower cost than Opus. Haiku 4.5 as the fallback prevents a Sonnet capacity event from killing a batch mid-run.

## Concurrency rule

Maximum **2 sub-agents running concurrently**. Spawning more than two in parallel causes hangs. With N repositories, run ⌈N/2⌉ batches sequentially, using `&` to background each spawn within a batch and `wait` to block until both finish.

```bash
claude -p "<sub-agent prompt for repo A>" \
  --model claude-sonnet-4-6 --fallback-model claude-haiku-4-5-20251001 \
  --dangerously-skip-permissions & PID1=$!
claude -p "<sub-agent prompt for repo B>" \
  --model claude-sonnet-4-6 --fallback-model claude-haiku-4-5-20251001 \
  --dangerously-skip-permissions & PID2=$!

wait $PID1; EXIT1=$?
wait $PID2; EXIT2=$?
```

## Timeout protocol

- **Initial attempt:** 5-minute timeout (300000ms).
- **If a sub-agent times out:** retry **ONCE** with 10-minute timeout (600000ms).
- **If it times out a second time:** mark that repo as failed in `state.json` with a clear error and move on.

Common timeout causes: large repository size, network latency, complex analysis requirements, MCP processing delays. Use the Bash tool's timeout parameter for proper timeout control.

## Sub-agent prompt template

Use this template verbatim, substituting the values for the specific repository being processed.

```
You are Claude Sub-Agent {N}. Process the repository '{org/repo-name}' according to the analyze-repos skill.

Your specific assignment:
- Repository: {org/repo-name}
- GitHub URL: {https://github.com/org/repo-name}
- Working directory: {absolute path to gh-repo-code-intelligence repo}
- Output directory: output/{repo_name_with_underscores}/{YYYY-MM-DD-HHMMSS}/
- Config: {preferred_diagrams, analysis_options, confluence settings from config.json}

Follow this complete workflow:

0. PRE-FLIGHT (mandatory): verify mcp__code-understanding__* and mcp__mermaid_image_generator__* tools are in your tool registry, and mcp__Conduit__* if confluence.enabled. If ANY required MCP tool is missing, write the failure to state.json with operation 'mcp_preflight', set success: false, clear processing_started/processing_agent, and EXIT immediately. Do NOT use git clone, file reads, or any non-MCP fallback. See Critical Rule 2 in the skill.

1. Mark this repo as processing in state.json:
   - Set processing_started to current timestamp
   - Set processing_agent to a unique id (e.g., claude_sub_agent_1)
   - If processing_started is already set on this repo by another agent, skip this repo and exit cleanly.

2. Verify repository access via MCP:
   - First try mcp__code-understanding__get_repo_structure on the URL.
   - If the response indicates the repo is not in the cache, the MCP cache may have purged it — proceed to clone via mcp__code-understanding__clone_repo regardless of what state.json says about cloned: true.
   - If state.json says cloned: false AND get_repo_structure says not cached, clone via mcp__code-understanding__clone_repo.
   - If state.json says cloned: true AND get_repo_structure confirms it, refresh via mcp__code-understanding__refresh_repo.
   - If clone or refresh fails, record the error in state.json with timestamp and operation, set success: false, clear processing_started, and exit.

3. Repository analysis — use ONLY mcp__code-understanding__* tools:
   - Analyze repository structure
   - Map code organization and relationships
   - Identify the most important files (use the configured critical_files_limit)
   - Extract existing documentation
   - Adjust focus per the analysis_profile (see references/profiles.md)

4. Diagram generation — use ONLY mcp__mermaid_image_generator__*:
   - Generate the diagrams listed in preferred_diagrams (typically component, sequence, class)
   - Save diagrams in the SAME directory as docs.md — no subdirectories
   - Filenames: component_diagram.png, sequence_diagram.png, class_diagram.png

5. Documentation generation:
   - Create docs.md in the timestamped output directory
   - Include: repository overview, key findings, important files and their purpose, architecture insights, embedded image references
   - Tailor tone, depth, and content to the analysis_profile (see references/profiles.md)
   - Image references MUST be filename only: ![Component Diagram](component_diagram.png) — NOT diagrams/component_diagram.png and NOT /absolute/path
   - After writing docs.md, read it back to verify all image references are filename-only

6. Confluence upload (only if confluence.enabled: true):
   - See references/confluence-upload.md for the full workflow
   - Use the page title formula: {repository_name}{page_suffix}
   - If a page with this title already exists under the parent, update it (replace all content + attachments). Otherwise create new.
   - Attach all generated diagram PNGs
   - Update profiles_analyzed in state.json with the page id, page title, and timestamp

7. Update state.json on completion:
   - On success: set cloned, last_updated, last_analysis, docs_generated, diagrams_generated, confluence_uploaded (if applicable), success: true, clear errors, populate profiles_analyzed[current_profile]
   - On failure: set success: false, populate errors with the structured error, clear processing_started/processing_agent
   - In both cases, clear processing_started and processing_agent

8. Run /compact to clear context

9. Report a concise summary of what you accomplished
```

## State coordination protocol

Each agent writes to `state.json` thread-safely:

1. Read current state.json.
2. Update only the agent's repo entry.
3. Write back.

Two-agent batches mean two concurrent writers in the worst case. Check for `processing_started` on the target repo before claiming it — if another agent already claimed it (different `processing_agent`), skip and exit cleanly so you don't race.

## State update fields (full set on completion)

```json
{
  "{org/repo-name}": {
    "processing_started": null,
    "processing_agent": null,
    "cloned": true,
    "last_updated": "YYYY-MM-DD-HHMMSS",
    "last_analysis": "YYYY-MM-DD-HHMMSS",
    "docs_generated": "YYYY-MM-DD-HHMMSS",
    "diagrams_generated": true,
    "confluence_uploaded": "YYYY-MM-DD-HHMMSS",
    "confluence_page_id": "12345",
    "success": true,
    "errors": [],
    "profiles_analyzed": {
      "{current_profile}": {
        "timestamp": "YYYY-MM-DD-HHMMSS",
        "confluence_page_id": "12345",
        "confluence_page_title": "{org/repo-name} - {profile suffix}",
        "docs_generated": "YYYY-MM-DD-HHMMSS"
      }
    }
  }
}
```

For state.json schema details and the structured-error format, see `references/state-schema.md`.
