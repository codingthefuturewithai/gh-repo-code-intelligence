# CLAUDE.md

This file provides guidance to Claude Code when analyzing GitHub repositories. You should use your available MCP tools to perform this analysis.

## Your Objective

When instructed to analyze repositories:

1. Read the `config.json` and `state.json` files (creating state.json if it doesn't exist)
2. For each repository in the config:
   - Analyze its current state
   - Perform repository analysis using MCP code-understanding tools
   - Generate local documentation with diagrams using MCP mermaid tools
   - Update the state.json file with new information
   - Upload the generated documentation to Confluence (if confluence upload is enabled)

## File Structure

```
.
├── CLAUDE.md              <- These instructions (your runtime guide)
├── config.json            <- Repositories to analyze and preferences
├── state.json             <- Dynamic state tracking for repositories (created if needed)
└── output/                <- Directory for generated artifacts
    └── {repo_name}/       <- Subdirectory for each repository
        ├── {timestamp}/   <- Timestamped analysis results
        │   ├── docs.md    <- Generated documentation
        │   └── diagrams/  <- Generated diagrams
        └── latest/        <- Symlink to most recent analysis
```

## Tool Selection Requirements

You MUST adhere to these tool selection guidelines:

1. **Repository Operations**: 
   - Use ONLY the MCP code-understanding tools for repository cloning, refreshing, and analysis
   - Do NOT fall back to web fetching or other non-MCP methods if these tools fail
   - If a repository clone fails, mark it as failed in state.json and move on to the next repository

2. **Diagram Generation**:
   - Use ONLY the MCP mermaid_image_generator tools for creating diagrams
   - Generate diagrams based on your analysis of the repository's structure

3. **File Operations**:
   - Use file read/write tools to manage local documentation and state

4. **Confluence Operations**:
   - Use MCP Conduit tools for uploading documentation to Confluence
   - Use the 'pbs' site alias for the PBS Atlassian instance
   - Upload to the CFDP Confluence space under the parent page "CFDP Polaris/SDVI Repo Docs"

## URL Format Requirements

- Always verify all repository URLs in config.json use the HTTPS format (`https://github.com/...`) 
- If you encounter SSH URLs (`git@github.com:...`), convert them to HTTPS format before using
- SSH URLs will prompt for credentials and break automation
- Example conversion:
  - From: `git@github.com:organization/repo-name.git`
  - To: `https://github.com/organization/repo-name.git`

## Analysis Goals

For each repository in config.json, accomplish these goals:

1. **Repository Access**
   - First, verify and fix repository URLs to ensure they use HTTPS format
   - Then attempt to use the MCP code-understanding get_repo_structure tool
     - If this tool returns an error indicating the repository is not in the cache, attempt to clone the repository regardless of what state.json indicates
     - The repository may have been purged from the MCP server's cache even if state.json shows it as cloned
   - If the repository has never been cloned (according to state.json AND the get_repo_structure check), use MCP code-understanding clone tool
   - If it's already cloned and verified to be in the cache, use MCP code-understanding refresh tool
   - If either operation fails, record the error in state.json and skip to the next repository

2. **Repository Analysis**
   - Use MCP code-understanding tools to:
     - Analyze the repository's structure
     - Map the code's organization and relationships
     - Identify the most important files
     - Extract existing documentation
   - If any analysis step fails, note the failure but continue with available information

3. **Diagram Generation**
   - Use MCP mermaid_image_generator to create diagrams based on your analysis
   - Focus on diagram types specified in the configuration
   - Save diagrams to the output directory with the current timestamp
   - Include component, sequence, or class diagrams as appropriate

4. **Documentation Generation**
   - Create a comprehensive markdown file (docs.md) that includes:
     - Repository overview
     - Key findings from your analysis
     - Important files and their purpose
     - Architecture insights
     - References to the generated diagrams with embedded image links
   - Save this file in the timestamped output directory
   - **IMPORTANT**: Always use relative paths for image references in the markdown documentation
     - Example: Use `![Component Diagram](diagrams/component_diagram.png)` instead of absolute paths
     - All image paths should be relative to the docs.md file location

5. **State Management**
   - Update the repository's state in state.json after each operation
   - Record success/failure status and timestamps for each completed operation

6. **Confluence Upload**
   - After generating documentation locally, upload it to Confluence
   - First, find the parent page "CFDP Polaris/SDVI Repo Docs" in the CFDP space
   - For each repository, create or update a child page under the parent
   - Page title should be the repository name (e.g., "org/repo-name")
   - Convert the markdown documentation to Confluence format
   - Upload any generated diagrams as attachments (if supported)

## MCP Cache Handling

The MCP code-understanding tool has a limit on the number of repositories it can keep in its cache:

- When this limit is reached, the MCP server automatically purges older repositories
- Your state.json may indicate a repository is cloned, but the MCP server might have removed it
- Always verify repository existence with get_repo_structure before deciding to refresh vs. clone
- If get_repo_structure returns an error indicating the repository doesn't exist in cache, you must clone it regardless of what state.json says
- Only after successful verification with get_repo_structure should you use refresh_repo instead of clone_repo

## Error Handling

**CRITICAL**: Past failures are NOT predictive. Every error in state.json represents a moment in time that has passed. Network conditions change, services recover, permissions get fixed, rate limits reset. NEVER let ANY historical error prevent you from attempting operations. What failed yesterday might work perfectly today.

### Startup Verification
Before processing any repository, you MUST:
1. Check your current environment to see which tools are actually available
2. Test MCP tool availability by attempting a simple operation (e.g., `mcp__code-understanding__get_repo_structure` on any URL)
3. If MCP tools are available NOW, proceed with analysis regardless of what errors are in state.json
4. Only record NEW errors that occur during the current session

### Error Recovery Process
When you encounter ANY errors in state.json from previous runs:
1. **TREAT ALL PAST ERRORS AS POTENTIALLY TEMPORARY** - Network issues, service outages, permission problems, rate limits, and tool availability can all change between runs
2. **NEVER SKIP OPERATIONS** based on past failures - Always attempt every operation fresh
3. **TEST CURRENT CONDITIONS** - What failed before might work now
4. **CLEAR OLD ERRORS** when operations succeed - Past failures are historical, not permanent
5. **ASSUME EVERYTHING IS RECOVERABLE** unless proven otherwise in the current session

### Handling Specific Error Types
- **"MCP tools not available" errors**: These are often temporary. Always retry regardless of past failures
- **Repository access errors**: May be due to network issues, permissions, or deleted repos
- **If repository cloning/refreshing fails**: Record the NEW error with current timestamp and move to the next repository
- **If structure analysis fails**: Note the failure but continue with available information
- **If diagram generation fails**: Note the failure but still produce documentation with text-only analysis
- **Do NOT attempt to use alternative non-MCP methods if the recommended tools fail**

### State.json Error Management
When updating state.json:
1. Keep only the most recent error for each operation type
2. Include timestamps with all errors
3. Clear errors when operations succeed
4. Structure errors as: `{"timestamp": "YYYY-MM-DD-HHMMSS", "operation": "clone|refresh|analyze", "message": "error details"}`

## Configuration Format

The `config.json` file follows this structure:
```json
{
  "repos": [
    {
      "name": "org/repo-name",
      "github_url": "https://github.com/org/repo-name"
    }
  ],
  "preferred_diagrams": ["component", "sequence", "class"],
  "doc_format": "markdown",
  "analysis_options": {
    "critical_files_limit": 50,
    "include_metrics": true,
    "max_tokens": 5000
  },
  "confluence": {
    "enabled": true,
    "site_alias": "pbs",
    "space_key": "CFDP",
    "parent_page_title": "CFDP Polaris/SDVI Repo Docs"
  }
}
```

## State Management

The `state.json` file tracks each repository's status. If this file doesn't exist, you must create it:

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
    "errors": []
  }
}
```

### Important State Management Rules

1. **state.json is a HISTORICAL LOG, not a predictor of future failures**
   - It records what happened at specific points in time
   - Past failures MUST NOT prevent future attempts
   - Every run is a fresh start - assume all previous issues have been resolved
   - Errors in state.json represent moments in time, NOT permanent conditions

2. **Error Array Management**
   - Errors should be structured objects with timestamps
   - Clear the errors array when operations succeed
   - Example error format:
   ```json
   {
     "timestamp": "2025-06-20-143022",
     "operation": "clone",
     "message": "Repository not found"
   }
   ```

3. **Recovery Behavior**
   - **IGNORE ALL PAST ERRORS** regardless of type - they're history, not current reality
   - **ATTEMPT EVERY OPERATION** as if it's the first time
   - Common temporary failures that MUST be retried:
     - "MCP tools not available" 
     - "Network timeout"
     - "Rate limit exceeded"
     - "Permission denied"
     - "Repository not found"
     - "Clone failed"
     - ANY other error from a previous run
   - Success should clear ALL relevant errors and update status fields
   - **The only failures that matter are ones happening RIGHT NOW**

During processing, record any NEW errors in the "errors" array, and update other fields as appropriate.

## Execution

When asked to "Analyze repositories according to CLAUDE.md", you should:

1. **FIRST verify MCP tool availability**:
   - Test that MCP code-understanding tools are accessible by making a test call
   - If tools are available, proceed regardless of any errors in state.json
   - If tools are truly unavailable, stop and report the issue clearly
2. Read config.json
3. Check if state.json exists:
   - If it exists, read it BUT DO NOT let old errors stop you from trying
   - If it doesn't exist, create it by initializing an empty state for each repository in config.json
   - If it contains errors about "MCP tools not available", ignore them and test current availability
4. Create the current timestamp for output directories (YYYY-MM-DD-HHMMSS format)
5. For each repository:
   - Convert any SSH URLs to HTTPS format
   - First verify if the repository exists in the MCP cache by attempting get_repo_structure
   - If get_repo_structure fails with a "not in cache" error, clone the repository regardless of state.json status
   - If get_repo_structure succeeds but state.json shows it needs updating, refresh the repository
   - If repository access fails, record the error and move to the next repository
   - Create a timestamped directory under output/{repo_name}/
   - Use MCP mermaid_image_generator to create diagrams in the diagrams/ subdirectory
   - Create a comprehensive docs.md file with embedded references to the diagrams using relative paths
   - Update state.json with your progress
   - If confluence upload is enabled in config.json:
     - Find the parent page using get_confluence_page with space_key and parent_page_title
     - Check if a child page for this repository already exists
     - Prepare attachments array by scanning the diagrams/ directory for generated images
     - Convert markdown image references to Confluence format (!filename!)
     - Create or update the Confluence page with:
       - The converted documentation content
       - All diagram images as attachments
     - Record the confluence upload status and page ID in state.json
6. Provide a summary of actions taken and results

## URL Format Conversion

When converting SSH URLs to HTTPS:

1. SSH URL pattern: `git@github.com:organization/repo-name.git`
2. HTTPS URL pattern: `https://github.com/organization/repo-name.git`

Steps to convert:
- Replace `git@github.com:` with `https://github.com/`
- Keep the organization/repo-name part unchanged
- Keep or add `.git` at the end if desired (optional)

## Image Path Guidelines

When including images in the documentation:

1. **Always use relative paths** - this ensures documentation can be moved between systems
2. **Reference images relative to the docs.md location** - typically `diagrams/image_name.png`
3. **Example markdown syntax**:
   ```markdown
   ![Component Diagram](diagrams/component_diagram.png)
   ```
4. **Never use absolute file paths** starting with / or drive letters
5. **Verify all image paths** are correctly formatted before saving the documentation

## Confluence Upload Workflow

When uploading documentation to Confluence (if enabled in config.json):

1. **Prerequisites**:
   - Confluence upload must be enabled in config.json with `confluence.enabled: true`
   - The site_alias, space_key, and parent_page_title must be configured
   - MCP Conduit server must be available and configured

2. **Finding the Parent Page**:
   - Use `mcp__Conduit__get_confluence_page` with:
     - `site_alias`: The configured site (default: "pbs")
     - `space_key`: The configured space (default: "CFDP")
     - `title`: The configured parent page title (default: "CFDP Polaris/SDVI Repo Docs")
   - If the parent page doesn't exist, report the error and skip Confluence upload

3. **Creating/Updating Child Pages**:
   - For each repository, create a child page under the parent
   - Page title should match the repository name (e.g., "org/repo-name")
   - Check if a page with this title already exists under the parent:
     - If it exists: Use `mcp__Conduit__update_confluence_page`
     - If it doesn't exist: Use `mcp__Conduit__create_confluence_page_from_markdown`

4. **Content Preparation**:
   - The Conduit tool automatically converts markdown to Confluence format
   - Prepare image attachments for upload:
     - Create an attachments array with `AttachmentSpec` objects
     - Each attachment needs:
       - `local_path`: Full path to the image file on local system
       - `name_on_confluence`: Name for the attachment in Confluence
     - Example:
       ```json
       {
         "local_path": "/path/to/output/repo-name/timestamp/diagrams/component_diagram.png",
         "name_on_confluence": "component_diagram.png"
       }
       ```
   - Update markdown content to reference attached images using Confluence syntax:
     - Replace relative paths like `![Diagram](diagrams/image.png)`
     - With Confluence attachment syntax: `!image.png!`

5. **Uploading with Attachments**:
   - When creating a new page:
     ```
     mcp__Conduit__create_confluence_page_from_markdown
     - space: "CFDP"
     - title: "org/repo-name"
     - content: (markdown with Confluence image syntax)
     - parent_id: (from parent page lookup)
     - attachments: [array of AttachmentSpec objects]
     - site_alias: "pbs"
     ```
   - When updating an existing page:
     ```
     mcp__Conduit__update_confluence_page
     - space_key: "CFDP"
     - title: "org/repo-name"
     - content: (markdown with Confluence image syntax)
     - expected_version: (current version number)
     - attachments: [array of AttachmentSpec objects]
     - minor_edit: false (for significant updates)
     - site_alias: "pbs"
     ```

6. **Image Reference Format**:
   - In local markdown: `![Component Diagram](diagrams/component_diagram.png)`
   - In Confluence content: `!component_diagram.png!`
   - The tool will handle the conversion when preparing content for upload

7. **Error Handling**:
   - If Confluence upload fails, record the error in state.json
   - Continue processing other repositories even if one upload fails
   - Do not retry failed uploads in the same session unless explicitly requested

6. **State Tracking**:
   - Update `confluence_uploaded` timestamp upon successful upload
   - Store the `confluence_page_id` for future reference
   - Clear any previous confluence-related errors on success