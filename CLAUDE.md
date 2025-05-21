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

## MCP Cache Handling

The MCP code-understanding tool has a limit on the number of repositories it can keep in its cache:

- When this limit is reached, the MCP server automatically purges older repositories
- Your state.json may indicate a repository is cloned, but the MCP server might have removed it
- Always verify repository existence with get_repo_structure before deciding to refresh vs. clone
- If get_repo_structure returns an error indicating the repository doesn't exist in cache, you must clone it regardless of what state.json says
- Only after successful verification with get_repo_structure should you use refresh_repo instead of clone_repo

## Error Handling

- If repository cloning/refreshing fails: Record the error in state.json and move to the next repository
- If structure analysis fails: Note the failure and attempt to continue with any available information
- If diagram generation fails: Note the failure but still produce documentation with text-only analysis
- Do NOT attempt to use alternative non-MCP methods if the recommended tools fail

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
    "errors": []
  }
}
```

During processing, record any errors in the "errors" array, and update other fields as appropriate.

## Execution

When asked to "Analyze repositories according to CLAUDE.md", you should:

1. Read config.json
2. Check if state.json exists:
   - If it exists, read it
   - If it doesn't exist, create it by initializing an empty state for each repository in config.json
3. Create the current timestamp for output directories (YYYY-MM-DD-HHMMSS format)
4. For each repository:
   - Convert any SSH URLs to HTTPS format
   - First verify if the repository exists in the MCP cache by attempting get_repo_structure
   - If get_repo_structure fails with a "not in cache" error, clone the repository regardless of state.json status
   - If get_repo_structure succeeds but state.json shows it needs updating, refresh the repository
   - If repository access fails, record the error and move to the next repository
   - Create a timestamped directory under output/{repo_name}/
   - Use MCP mermaid_image_generator to create diagrams in the diagrams/ subdirectory
   - Create a comprehensive docs.md file with embedded references to the diagrams using relative paths
   - Update state.json with your progress
5. Provide a summary of actions taken and results

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