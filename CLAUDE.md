# CLAUDE.md

This file provides guidance to Claude Code when analyzing GitHub repositories. You should use your available MCP tools to perform this analysis.

## Your Objective

When instructed to analyze repositories:

1. Read the `config.json` and `state.json` files (creating state.json if it doesn't exist)
2. Determine the optimal processing strategy based on repository count:
   - **1-2 repositories**: Process sequentially with `/compact` between each
   - **3+ repositories**: Use parallel processing with Task agents
3. For parallel processing:
   - Spawn multiple Task agents to process repositories concurrently
   - Each agent handles one repository independently
   - Coordinate state updates through state.json
4. For each repository (whether sequential or parallel):
   - Analyze its current state
   - Perform repository analysis using MCP code-understanding tools
   - Generate local documentation with diagrams using MCP mermaid tools
   - Update the state.json file with new information
   - Upload the generated documentation to Confluence (if confluence upload is enabled)

## Permission Configuration

To avoid permission prompts that block automated execution:
- Configure your settings.local.json with broad permissions OR
- Run Claude Code with permission bypass flags
- Task agents inherit permissions from the parent instance
- Ensure all Bash commands, file operations, and MCP tools are pre-approved

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
   - Use the site alias, space key, and parent page title from config.json

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
   - **VERIFY**: After generating docs.md, read it back to confirm all image references are present

5. **State Management**
   - Update the repository's state in state.json after each operation
   - Record success/failure status and timestamps for each completed operation

6. **Confluence Upload**
   - After generating documentation locally, upload it to Confluence if enabled
   - First, find the parent page specified in config.json
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
    "max_tokens": 5000,
    "timeout_minutes": 10
  },
  "confluence": {
    "enabled": true,
    "site_alias": "your-site-alias",
    "space_key": "YOUR_SPACE",
    "parent_page_title": "Your Parent Page Title"
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

## Parallel Execution Strategy

When processing 3 or more repositories, use parallel Task agents to maximize efficiency and prevent context accumulation:

### Phase 1: Setup and Planning
1. Read config.json to get all repositories
2. Verify MCP tool availability with a test call
3. Read or create state.json
4. Calculate optimal batch size (maximum 2 repositories per batch to prevent hanging)
5. Create the execution plan

### Phase 2: Parallel Repository Processing
1. **Spawn Task agents in batches** using concurrent tool invocations:
   ```
   - Agent 1: Process repo A
   - Agent 2: Process repo B  
   (maximum 2 agents at once to prevent hanging)
   ```
   
   **Example Task Agent Prompt**:
   ```
   You are Task Agent 1. Process the repository "org/repo-name" according to CLAUDE.md.
   
   Your specific assignment:
   - Repository: org/repo-name
   - GitHub URL: https://github.com/org/repo-name
   - Output directory: output/org_repo-name/YYYY-MM-DD-HHMMSS/
   - Config: {preferred_diagrams, analysis_options, confluence settings}
   
   Follow the complete workflow in CLAUDE.md:
   1. Check/update state.json (mark as processing)
   2. Clone or refresh the repository
   3. Analyze and generate documentation
   4. Create diagrams as configured
   5. Upload to Confluence if enabled
   6. Update state.json (mark as complete)
   7. Run /compact before finishing
   8. Report your results
   
   IMPORTANT: Run with --dangerously-skip-permissions to avoid permission prompts
   ```

2. **Each Task agent independently**:
   - Receives its assigned repository name and config
   - **PERMISSIONS**: Agent runs with full permissions to avoid blocking on tool approvals
   - Performs the complete analysis workflow:
     - Convert SSH URLs to HTTPS if needed
     - Check MCP cache with get_repo_structure
     - Clone or refresh repository as appropriate
     - Analyze structure and critical files
     - Generate diagrams with MCP mermaid tools
     - Create comprehensive documentation
     - **For Confluence upload**:
       - Read the generated docs.md file
       - List all diagram files in the diagrams/ directory
       - Replace markdown image syntax with Confluence syntax before upload
       - Ensure all images are properly embedded, not just attached
   - Updates state.json with thread-safe writes:
     - Read current state
     - Update only its repository's section
     - Write back with timestamp to track last updater
   - Runs `/compact` before completing to clear its context

3. **State Coordination Protocol**:
   - Each agent must follow this update pattern:
     ```json
     {
       "repo-name": {
         "processing_started": "timestamp",
         "processing_agent": "agent_id",
         "cloned": true,
         "last_updated": "timestamp",
         // ... other fields
       }
     }
     ```
   - Agents check for "processing_started" to avoid conflicts
   - If a repo is already being processed, skip to next available

### Phase 3: Monitoring and Completion
1. After spawning each batch, wait for completion before spawning next batch
2. Monitor state.json to ensure both agents finish
3. Only after both complete, spawn next batch of 2
4. Continue until all repositories are processed
5. Generate final summary report

**IMPORTANT**: Never spawn new agents until previous batch completes to prevent resource exhaustion

**TROUBLESHOOTING HUNG AGENTS**:
- If an agent shows "processing_started" but no progress after 10 minutes
- Check if "cloned": false - likely stuck on repository access
- Common causes: network issues, large repo size, permission denials
- Consider reducing batch size to 1 agent at a time for problematic repos

## Sequential Execution (1-2 repositories)

For small repository sets, use sequential processing with context management:

1. Process each repository one at a time
2. After completing each repository (all analysis, docs, uploads):
   - Update state.json
   - Execute `/compact` to clear context
   - Continue to next repository
3. This prevents context accumulation without the complexity of parallel coordination

## Execution

When asked to "Analyze repositories according to CLAUDE.md", you should:

1. **FIRST verify MCP tool availability**:
   - Test that MCP code-understanding tools are accessible by making a test call
   - If tools are available, proceed regardless of any errors in state.json
   - If tools are truly unavailable, stop and report the issue clearly
2. Read config.json and count repositories
3. Check if state.json exists:
   - If it exists, read it BUT DO NOT let old errors stop you from trying
   - If it doesn't exist, create it by initializing an empty state for each repository in config.json
   - If it contains errors about "MCP tools not available", ignore them and test current availability
4. **Choose execution strategy**:
   - **1-2 repos**: Use sequential processing with `/compact` between each
   - **3+ repos**: Use parallel Task agents as described above
5. Create the current timestamp for output directories (YYYY-MM-DD-HHMMSS format)
6. Execute chosen strategy:
   - **Sequential**: Process each repo fully, then `/compact`, then next
   - **Parallel**: Spawn Task agents in batches, monitor progress
7. Provide a summary of actions taken and results

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
     - `site_alias`: From config.json confluence.site_alias
     - `space_key`: From config.json confluence.space_key
     - `title`: From config.json confluence.parent_page_title
   - If the parent page doesn't exist, report the error and skip Confluence upload

3. **Creating/Updating Child Pages**:
   - For each repository, create a child page under the parent
   - Page title should match the repository name (e.g., "org/repo-name")
   - Check if a page with this title already exists under the parent:
     - If it exists: 
       - Use `mcp__Conduit__update_confluence_page`
       - Retrieve the current page version number (required for updates)
       - Include ALL images in attachments array even if they were previously uploaded
       - The Conduit tool will handle replacing existing attachments
     - If it doesn't exist: 
       - Use `mcp__Conduit__create_confluence_page_from_markdown`

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
   - **CRITICAL**: You MUST prepare the content with embedded images BEFORE calling the upload tool:
     - Read your generated docs.md file
     - Create a NEW content string where you replace ALL markdown image syntax
     - Find all image references: `![Component Diagram](diagrams/component_diagram.png)`
     - Replace with Confluence storage format as specified in the MCP tool documentation 
     - Place the image references EXACTLY where you want the images to appear
     - The filename MUST match the `name_on_confluence` exactly
     - The Conduit tool documentation shows the exact XML format to use for embedding images
     - DO NOT just append image references at the end - they must be embedded in context

5. **Uploading with Attachments**:
   - When creating a new page:
     ```
     mcp__Conduit__create_confluence_page_from_markdown
     - space: (from config.json confluence.space_key)
     - title: "org/repo-name"
     - content: (markdown with Confluence storage format for images)
     - parent_id: (from parent page lookup)
     - attachments: [array of AttachmentSpec objects]
     - site_alias: (from config.json confluence.site_alias)
     ```
   - When updating an existing page:
     ```
     # First get the page to retrieve version
     mcp__Conduit__get_confluence_page
     - space_key: (from config.json confluence.space_key)
     - title: "org/repo-name"
     - site_alias: (from config.json confluence.site_alias)
     
     # Then update with version number
     mcp__Conduit__update_confluence_page
     - space_key: (from config.json confluence.space_key)
     - title: "org/repo-name"
     - content: (markdown with Confluence storage format for images)
     - expected_version: (version from get_confluence_page)
     - attachments: [array of ALL image AttachmentSpec objects]
     - minor_edit: false (for significant updates)
     - site_alias: (from config.json confluence.site_alias)
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

## Handling Re-Analysis of Existing Repositories

When re-analyzing a repository that has already been processed:

1. **Repository Updates**:
   - Always use `refresh_repo` instead of `clone_repo` for existing repositories
   - Generate new timestamps for output directories to preserve history

2. **Documentation Updates**:
   - Create new docs.md with embedded image references
   - Ensure ALL image references use markdown syntax: `![Name](diagrams/file.png)`

3. **Confluence Page Updates**:
   - Check if confluence_page_id exists in state.json
   - If yes, retrieve the existing page to get version number
   - Replace ALL content and attachments (don't append)
   - Convert ALL markdown image syntax to Confluence syntax before upload
   - Include ALL images in attachments array, even if previously uploaded

4. **Image Embedding Verification**:
   - After generating docs.md, read it to confirm image references exist
   - When preparing for Confluence, ensure EVERY `![...]` becomes `!filename!`
   - The Confluence page should show embedded images, not just attachments

## Task Agent Instructions

When you are spawned as a Task agent to process a specific repository:

1. **Identify Yourself**: You are an independent agent processing one repository
2. **Read Your Assignment**: Your prompt will specify which repository to process
3. **Follow the Complete Workflow**:
   - Read the current state.json to check repository status
   - Perform all analysis steps for your assigned repository
   - Generate all required outputs (docs, diagrams)
   - Upload to Confluence if configured
   - Update state.json for your repository only
4. **State Update Protocol**:
   ```json
   // Before starting work, mark as processing:
   {
     "your-repo": {
       "processing_started": "2025-01-15-103045",
       "processing_agent": "task_agent_1"
     }
   }
   // After completion, update all fields and clear processing:
   {
     "your-repo": {
       "processing_started": null,
       "processing_agent": null,
       "cloned": true,
       "last_updated": "2025-01-15-104530",
       "last_analysis": "2025-01-15-104530",
       "docs_generated": "2025-01-15-104530",
       "diagrams_generated": true,
       "confluence_uploaded": "2025-01-15-104600",
       "errors": []
     }
   }
   ```
5. **Context Management**: Run `/compact` before reporting completion
6. **Error Handling**: If you encounter errors, record them but continue with what you can
7. **Report Back**: Provide a concise summary of what you accomplished