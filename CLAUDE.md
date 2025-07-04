# CLAUDE.md

This file provides guidance to Claude Code when analyzing GitHub repositories. You should use your available MCP tools to perform this analysis.

## ⚠️ CRITICAL RULE - READ FIRST ⚠️

**NEVER PROCESS REPOSITORIES DIRECTLY IN THE MAIN INSTANCE**

- ✅ ALWAYS spawn Claude sub-agents with `--dangerously-skip-permissions`
- ✅ Even for 1 repository - MUST use sub-agent
- ❌ NEVER use MCP tools directly in main instance
- ❌ NEVER call mcp__code-understanding tools directly
- ❌ NEVER call mcp__Conduit tools directly

**VIOLATION OF THIS RULE CAUSES PERMISSION FAILURES AND BROKEN CONFLUENCE UPLOADS**

## Your Objective

When instructed to analyze repositories:

1. Read the `config.json` and `state.json` files (creating state.json if it doesn't exist)
2. **MANDATORY: ALWAYS use Claude sub-agents to process ALL repositories** (even single repositories)
3. **FORBIDDEN: NEVER process repositories directly in the main instance**
4. **Main instance role**: ONLY spawn Claude sub-agents - ZERO repository processing allowed
5. For Claude sub-agent processing:
   - Spawn Claude sub-agents using: `claude -p "prompt" --dangerously-skip-permissions`
   - Maximum 2 sub-agents running concurrently (to prevent hanging)
   - Each sub-agent handles one repository independently
   - Sub-agents update state.json with processing_started/processing_agent fields
   - Sub-agents run `/compact` before completing

## Permission Configuration

To avoid permission prompts that block automated execution:
- Configure your settings.local.json with broad permissions OR
- Run Claude Code with permission bypass flags
- Claude sub-agents inherit permissions from the parent instance
- Ensure all Bash commands, file operations, and MCP tools are pre-approved

## File Structure

```
.
├── CLAUDE.md              <- These instructions (your runtime guide)
├── config.json            <- Repositories to analyze and preferences
├── state.json             <- Dynamic state tracking for repositories (created if needed)
└── output/                <- Directory for generated artifacts
    └── {repo_name}/       <- Subdirectory for each repository
        └── {timestamp}/   <- Timestamped analysis results containing:
            ├── docs.md    <- Generated documentation
            ├── component_diagram.png  <- Generated diagrams
            ├── sequence_diagram.png   <- (all in same directory)
            └── class_diagram.png      <- (no subdirectories)
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

## Analysis Profiles

The tool supports different analysis profiles to create targeted documentation for specific audiences. Each profile generates a complete, self-contained document optimized for its intended readers.

### Available Profiles

1. **standard** (default)
   - General-purpose technical documentation
   - Balanced coverage of architecture, code, and functionality
   - Suitable for mixed technical audiences
   - Output: ~15-20 pages

2. **developer_onboarding**
   - Focused on helping new developers understand and work with the codebase
   - Emphasizes: technology stack, development setup, code organization, design patterns
   - Includes: how to navigate code, key workflows, integration points
   - Output: ~25-30 pages

3. **architecture_review**
   - Designed for architects and technical leads
   - Emphasizes: system design, architectural decisions, patterns, scalability
   - Includes: service boundaries, technology rationale, extensibility points
   - Output: ~25-30 pages

4. **business_understanding**
   - Targeted at product managers and business stakeholders
   - Emphasizes: business capabilities, domain concepts, workflows
   - Includes: user journeys, business rules, less technical detail
   - Output: ~20-25 pages

5. **operations_handover**
   - Created for DevOps and operations teams
   - Emphasizes: deployment, configuration, monitoring, dependencies
   - Includes: infrastructure needs, operational workflows, integration points
   - Output: ~20-25 pages

### Profile-Based Documentation

When an analysis profile is specified:
- The agent adjusts its analysis focus based on the profile
- Documentation tone and content are tailored to the target audience
- Specific diagrams are generated to support the profile's goals
- Each profile creates a complete story within ~30 pages (excluding images)

## Sub-Agent Analysis Goals

When you are spawned as a Claude sub-agent, accomplish these goals for your assigned repository:

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
   - **CRITICAL**: Save diagrams in the SAME directory as docs.md (no subdirectories)
   - Save all files to: output/{repo_name}/{timestamp}/
   - Include component, sequence, or class diagrams as appropriate

4. **Documentation Generation**
   - Create a comprehensive markdown file (docs.md) that includes:
     - Repository overview
     - Key findings from your analysis
     - Important files and their purpose
     - Architecture insights
     - References to the generated diagrams with embedded image links
   - Save this file in the timestamped output directory
   - **CRITICAL**: Image references must be FILENAME ONLY (no paths or subdirectories)
     - Correct: `![Component Diagram](component_diagram.png)`
     - WRONG: `![Component Diagram](diagrams/component_diagram.png)`
     - WRONG: `![Component Diagram](/path/to/component_diagram.png)`
   - All files (docs.md and images) must be in the same directory
   - **VERIFY**: After generating docs.md, read it back to confirm all image references are filename-only

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
    "timeout_minutes": 10,
    "analysis_profile": "standard"  // Options: "standard", "developer_onboarding", "architecture_review", "business_understanding", "operations_handover"
  },
  "confluence": {
    "enabled": true,
    "site_alias": "your-site-alias",
    "space_key": "YOUR_SPACE",
    "parent_page_title": "Your Parent Page Title",
    "page_suffix": ""  // Optional: e.g., " - Developer Guide" for specific analysis profiles
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
    "errors": [],
    "processing_started": null,
    "processing_agent": null,
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

ALWAYS use Claude sub-agents to process repositories, even for a single repository:

### Phase 1: Main Instance Setup (YOU ONLY DO THIS)
1. Read config.json to get all repositories
2. Verify MCP tool availability with a test call
3. Read or create state.json
4. Plan batches (maximum 2 sub-agents per batch to prevent hanging)
5. **DO NOT process repositories yourself - only spawn sub-agents**

### Phase 2: Spawning Claude Sub-Agents
1. **REQUIREMENT: When processing 2 or more repositories, you MUST spawn exactly 2 sub-agents to run in parallel**. The `&` symbol at the end of each command runs it in the background for parallel execution. Use the `wait` command to ensure both complete before spawning the next batch.

   ```bash
   # ALWAYS spawn TWO sub-agents concurrently when you have multiple repositories
   claude -p "You are Claude Sub-Agent 1. Process repo A..." --dangerously-skip-permissions &
   claude -p "You are Claude Sub-Agent 2. Process repo B..." --dangerously-skip-permissions &
   wait  # Wait for both to complete before spawning next batch
   ```
   
   **Example Claude Sub-Agent Command**:
   ```bash
   claude -p "You are Claude Sub-Agent 1. Process the repository 'org/repo-name' according to CLAUDE.md.
   
   Your specific assignment:
   - Repository: org/repo-name
   - GitHub URL: https://github.com/org/repo-name
   - Working directory: {current working directory}
   - Output directory: output/org_repo-name/YYYY-MM-DD-HHMMSS/
   - Config: {preferred_diagrams, analysis_options, confluence settings}
   
   Follow the complete workflow in CLAUDE.md:
   1. Check/update state.json (mark as processing)
   2. Clone or refresh the repository
   3. Analyze and generate documentation
   4. Create diagrams as configured
   5. Upload to Confluence if enabled:
      - Find parent page in the configured space
      - Create/update page with profile-specific title formula
      - Attach all diagram images
      - Update profiles_analyzed in state.json
   6. Update state.json (mark as complete with ALL fields)
   7. Run /compact to clear context
   8. Report your results" --dangerously-skip-permissions
   ```

2. **Each Claude sub-agent independently**:
   - Receives its assigned repository name and config
   - **PERMISSIONS**: Runs with --dangerously-skip-permissions flag
   - Performs the complete analysis workflow:
     - Convert SSH URLs to HTTPS if needed
     - Check MCP cache with get_repo_structure
     - Clone or refresh repository as appropriate
     - Analyze structure and critical files
     - Generate diagrams with MCP mermaid tools
     - Create comprehensive documentation
     - **For Confluence upload**:
       - Read the generated docs.md file
       - List all image files in the same directory as docs.md
       - Keep markdown image references as-is (the tool handles conversion)
       - Ensure image filenames in markdown match the attachment names exactly
       - Remember: all files are in same directory, so use filename-only references
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
         "processing_agent": "claude_sub_agent_1",
         "cloned": true,
         "last_updated": "timestamp",
         "profiles_analyzed": {
           "[profile_name]": {
             "timestamp": "timestamp",
             "confluence_page_id": "page_id",
             "confluence_page_title": "title",
             "docs_generated": "timestamp"
           }
         }
         // ... other fields
       }
     }
     ```
   - Agents check for "processing_started" to avoid conflicts
   - If a repo is already being processed, skip to next available
   - MUST update profiles_analyzed section after Confluence upload

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
4. **MANDATORY: Execute using Claude sub-agents**:
   - ALWAYS use Claude sub-agents regardless of repository count (1, 5, 50 repos - ALWAYS sub-agents)
   - FORBIDDEN: NEVER process repositories directly in the main instance
   - VIOLATION OF THIS RULE WILL CAUSE SYSTEM FAILURE
5. Create the current timestamp for output directories (YYYY-MM-DD-HHMMSS format)
6. Spawn Claude sub-agents in batches using Bash:
   ```bash
   claude -p "prompt" --dangerously-skip-permissions &
   ```
   - Maximum 2 sub-agents running concurrently to prevent hanging
   - Use `wait` command to ensure batch completion before spawning next
   - Monitor progress via state.json
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

1. **Use FILENAME ONLY** - no paths, no subdirectories
2. **All files in same directory** - docs.md and all images together
3. **Example markdown syntax**:
   ```markdown
   ![Component Diagram](component_diagram.png)
   ```
4. **Never include paths** - no `diagrams/`, no `/path/to/`, just the filename
5. **Verify all image references** are filename-only before saving

## Confluence Upload Workflow

When uploading documentation to Confluence (if enabled in config.json):

1. **Prerequisites**:
   - Confluence upload must be enabled in config.json with `confluence.enabled: true`
   - The site_alias, space_key, and parent_page_title must be configured
   - MCP Conduit server must be available and configured

2. **Finding the Parent Page**:
   - Find the parent page specified in config.json
   - If the parent page doesn't exist, report the error and skip Confluence upload

3. **Creating/Updating Child Pages**:
   - For each repository, create a child page under the parent
   - **Page Title Formula**: `{repository_name}{page_suffix}`
     - `repository_name`: Exactly as specified in config.json repos[].name
     - `page_suffix`: From confluence.page_suffix (empty for standard profile)
     - Examples:
       - Standard: "org/repo-name"
       - Developer: "org/repo-name - Developer Guide"
       - Architecture: "org/repo-name - Architecture Review"
   - **CRITICAL**: Use this exact formula every time to ensure consistent page naming across runs
   - Check if a page with this title already exists under the parent
   - Create new page if it doesn't exist, update if it does

4. **Content Preparation**:
   - Use standard markdown format for documentation
   - Include image references as: `![alt text](filename.png)` - FILENAME ONLY
   - Ensure all files (docs.md and images) are in the same directory
   - Ensure image filenames in markdown match the actual image filenames exactly

5. **Uploading**:
   - Use appropriate Conduit tools based on whether creating or updating
   - Include all diagram images as attachments
   - Let the tool handle markdown to Confluence conversion

6. **Error Handling**:
   - If Confluence upload fails, record the error in state.json
   - Continue processing other repositories even if one upload fails
   - Do not retry failed uploads in the same session unless explicitly requested

7. **State Tracking**:
   - Update state.json with profile-specific information:
     ```json
     "profiles_analyzed": {
       "[profile_name]": {
         "timestamp": "YYYY-MM-DD-HHMMSS",
         "confluence_page_id": "12345",
         "confluence_page_title": "org/repo-name - Profile Suffix",
         "docs_generated": "YYYY-MM-DD-HHMMSS"
       }
     }
     ```
   - Store the exact page title used for future reference
   - Clear any previous confluence-related errors on success

## Handling Re-Analysis of Existing Repositories

When re-analyzing a repository that has already been processed:

1. **Repository Updates**:
   - Always use `refresh_repo` instead of `clone_repo` for existing repositories
   - Generate new timestamps for output directories to preserve history

2. **Documentation Updates**:
   - Create new docs.md with embedded image references
   - Ensure ALL image references use markdown syntax: `![Name](filename.png)` - FILENAME ONLY

3. **Confluence Page Updates**:
   - Check if confluence_page_id exists in state.json
   - If yes, retrieve the existing page to get version number
   - Replace ALL content and attachments (don't append)
   - Keep markdown image syntax as-is (the tool converts automatically)
   - Include ALL images in attachments array, even if previously uploaded

4. **Image Embedding Verification**:
   - After generating docs.md, read it to confirm image references exist
   - Ensure markdown image filenames match the attachment `name_on_confluence`
   - The Confluence page should show embedded images, not just attachments

## Claude Sub-Agent Instructions

When you are spawned as a Claude sub-agent (via `claude -p` command) to process a specific repository:

1. **Identify Yourself**: You are an independent agent processing one repository
2. **Read Your Assignment**: Your prompt will specify which repository to process
3. **Follow the Complete Workflow**:
   - Read the current state.json to check repository status
   - Check the configured analysis_profile in config.json
   - Perform all analysis steps for your assigned repository with profile-specific focus
   - Generate all required outputs (docs, diagrams) tailored to the profile
   - Upload to Confluence using the profile-specific page title formula
   - Update state.json for your repository only, including profile-specific data
4. **State Update Protocol**:
   ```json
   // Before starting work, mark as processing:
   {
     "your-repo": {
       "processing_started": "2025-01-15-103045",
       "processing_agent": "claude_sub_agent_1"
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
       "errors": [],
       "profiles_analyzed": {
         "[current_profile]": {
           "timestamp": "2025-01-15-104530",
           "confluence_page_id": "12345",
           "confluence_page_title": "your-repo - Profile Suffix",
           "docs_generated": "2025-01-15-104530"
         }
       }
     }
   }
   ```
5. **Context Management**: Run `/compact` before reporting completion
6. **Error Handling**: If you encounter errors, record them but continue with what you can
7. **Report Back**: Provide a concise summary of what you accomplished