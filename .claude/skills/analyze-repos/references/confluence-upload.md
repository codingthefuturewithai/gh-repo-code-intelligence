# Confluence upload workflow

This file applies only when `confluence.enabled: true` in `config.json`. It tells the sub-agent exactly how to publish a generated `docs.md` plus its diagram PNGs as a Confluence child page under the user's configured parent.

## Prerequisites

- `confluence.enabled: true`
- `confluence.site_alias`, `space_key`, `parent_page_title` are set
- The Conduit MCP server is available (`mcp__Conduit__*` tools in registry) and the alias matches the user's Conduit config
- The parent page already exists in the configured space — **the pipeline does not create the parent**

## Page title formula — apply EVERY TIME, no exceptions

```
{repository_name}{page_suffix}
```

- `repository_name`: exactly the value of `repos[].name` in config.json (e.g., `acme/payments-service`).
- `page_suffix`: the value from `confluence.page_suffix`. Empty for the `standard` profile, otherwise typically tied to the profile (e.g., ` - Architecture Review`).

Examples:

| profile | page_suffix | final title |
|---|---|---|
| standard | (empty) | `acme/payments-service` |
| developer_onboarding | ` - Developer Onboarding` | `acme/payments-service - Developer Onboarding` |
| architecture_review | ` - Architecture Review` | `acme/payments-service - Architecture Review` |
| business_understanding | ` - Business Overview` | `acme/payments-service - Business Overview` |
| operations_handover | ` - Operations Handover` | `acme/payments-service - Operations Handover` |

**This formula must be applied identically every run** so re-runs update the same page rather than creating duplicates.

## Workflow steps

### 1. Find the parent page

Use the Conduit MCP to look up the page with title `confluence.parent_page_title` in `confluence.space_key`. If it does not exist:

- Record the error in `state.json`:
  ```json
  {
    "timestamp": "...",
    "operation": "confluence_upload",
    "message": "Parent page '{title}' not found in space {space_key}"
  }
  ```
- Set `success: false`.
- Skip Confluence upload but **leave local docs+diagrams in place** — they're still useful.

### 2. Compute the target page title

`{repository_name}{page_suffix}` — see formula above.

### 3. Check if a page with that title already exists under the parent

- If yes: capture its `confluence_page_id` and version number. You will UPDATE it (replace all content + attachments).
- If no: you will CREATE a new child page under the parent.

### 4. Prepare the markdown content

- Use the generated `docs.md` as-is.
- Image references in the markdown MUST be filename-only: `![Component Diagram](component_diagram.png)` — NOT `diagrams/component_diagram.png`, NOT `/absolute/path/component_diagram.png`. The Conduit tool converts these references to Confluence attachment lookups, and the lookup matches by filename only.
- Verify by reading docs.md back after generation: every image reference should be a filename with no directory.

### 5. Upload via the Conduit MCP

- Pass the markdown content directly. Conduit handles markdown → Confluence storage format conversion.
- Attach all generated diagram PNGs from the same output directory. The attachment names must match the filenames referenced in the markdown exactly (case-sensitive).
- For an UPDATE, replace ALL content and ALL attachments — don't append. This ensures stale images don't linger.

### 6. Record the result in state.json

On success, populate `profiles_analyzed[{current_profile}]`:

```json
{
  "timestamp": "YYYY-MM-DD-HHMMSS",
  "confluence_page_id": "12345",
  "confluence_page_title": "acme/payments-service - Architecture Review",
  "docs_generated": "YYYY-MM-DD-HHMMSS"
}
```

Also set `confluence_uploaded` and `confluence_page_id` at the top level of the repo's entry. Clear any prior `confluence_upload` errors from the `errors` array.

## Error handling specifics

If the Conduit upload fails for any reason, record an `operation: "confluence_upload"` error with the verbatim failure message. Continue processing other repos — one repo's Confluence failure must not stop the rest.

**Do not retry failed Confluence uploads in the same session unless the user explicitly requests it.** Surface the failure in the final summary.

If the Conduit MCP tool is **unavailable** (not just returning an error — actually missing from the tool registry), that's a different failure mode. Record:

```json
{
  "timestamp": "...",
  "operation": "confluence_upload",
  "message": "mcp__Conduit__* tools unavailable in this session — Confluence upload skipped"
}
```

Do **not** invent "page not found" or other plausible-sounding results when the tool was simply unreachable. That misleads the user about the root cause. See Critical Rule 3 / Rule 2 in the main skill.

## Re-analysis (when this repo + profile has been analyzed before)

Re-running the pipeline against a repo that already has a Confluence page for this profile:

1. Always use `mcp__code-understanding__refresh_repo` (not clone) for the analysis side. New timestamp for the local output directory — preserve history.
2. For Confluence: the page title formula gives the same title, so `find_page_by_title` returns the existing page. Update it (don't create a duplicate).
3. Replace ALL content and attachments — don't append. Pass every diagram in the attachments array, even if it was previously uploaded.
4. After uploading, read docs.md once more to confirm all image references are filename-only and that those filenames match the freshly uploaded attachments.
