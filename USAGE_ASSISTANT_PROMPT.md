# Usage Assistant Instructions

This file contains instructions for AI coding assistants to help users configure and use the GitHub Repository Analysis Tool after setup is complete.

---

## To the AI Assistant

You are helping a user who has already set up the GitHub Repository Analysis Tool. Your role is to guide them through configuration, usage, troubleshooting, and optimization.

First, review these key files to understand the current state:
1. Read `config.json` to see their current configuration
2. Read `state.json` (if it exists) to understand any previous runs
3. Review `CLAUDE.md` to understand how the tool works internally
4. Check the `README.md` for usage documentation

## Tasks You Can Help With

### 1. Configuration Assistance

**Reviewing Current Configuration**:
- Read their `config.json` and explain each setting
- Suggest improvements based on their needs
- Help add or remove repositories

**Adding Repositories**:
```json
{
  "repos": [
    {
      "name": "organization/repository-name",
      "github_url": "https://github.com/organization/repository-name"
    }
  ]
}
```
- Ensure URLs are in HTTPS format (not SSH)
- Validate repository accessibility

**Analysis Options**:
- `critical_files_limit`: How many important files to analyze (default: 50)
- `include_metrics`: Whether to include code metrics (default: true)
- `max_tokens`: Token limit for analysis depth (default: 5000)
- `timeout_minutes`: How long to wait for analysis (default: 10)
- `analysis_profile`: Which profile to use (see section 3)

### 2. Running the Tool

**Basic Execution**:
```bash
claude -p "Analyze repositories according to CLAUDE.md" --dangerously-skip-permissions
```

**With Output Capture** (recommended for debugging):
```bash
# See real-time output and save to file
claude -p "Analyze repositories according to CLAUDE.md" --dangerously-skip-permissions --verbose --output-format stream-json | tee output.json

# Unbuffered output (prevents hanging appearance)
claude -p "Analyze repositories according to CLAUDE.md" --dangerously-skip-permissions --verbose --output-format stream-json | stdbuf -o0 tee output.json
```

**What Happens During Execution**:
1. Main instance reads config.json and state.json
2. Spawns sub-agents (max 2 at a time) to process repositories
3. Each sub-agent clones/refreshes, analyzes, generates docs and diagrams
4. Results saved to `output/{repo_name}/{timestamp}/`
5. If Confluence enabled, uploads documentation

**Monitoring Progress**:
- Check `state.json` for real-time status updates
- Look for `processing_started` and `processing_agent` fields
- `success: true/false` indicates completion status
- Check `errors` array for any issues

### 3. Choosing Analysis Profiles

Help the user select the right profile by asking about their needs:

**Questions to Ask**:
1. "Who will read this documentation?"
2. "What's the primary purpose?"
3. "What level of technical detail is needed?"

**Profile Selection Guide**:

| Profile | Best For | Emphasizes | Output Size |
|---------|----------|------------|-------------|
| `standard` | General technical audience | Balanced technical overview | ~15-20 pages |
| `developer_onboarding` | New developers joining team | Setup, code organization, how to contribute | ~25-30 pages |
| `architecture_review` | Architects, tech leads | Design decisions, patterns, scalability | ~25-30 pages |
| `business_understanding` | Product managers, stakeholders | Business logic, workflows, capabilities | ~20-25 pages |
| `operations_handover` | DevOps teams | Deployment, config, monitoring | ~20-25 pages |

**To Change Profile**:
```json
{
  "analysis_options": {
    "analysis_profile": "developer_onboarding"
  },
  "confluence": {
    "page_suffix": " - Developer Guide"  // Optional, for clarity
  }
}
```

### 4. Output Management

**Finding Generated Documentation**:
```
output/
└── {repository_name}/
    └── {YYYY-MM-DD-HHMMSS}/
        ├── docs.md              # Main documentation
        ├── component_diagram.png
        ├── sequence_diagram.png
        └── class_diagram.png
```

**Viewing Results**:
- Open `docs.md` in any markdown viewer
- Images are in the same directory
- If Confluence enabled, check the configured space

### 5. Confluence Integration

**Checking Confluence Configuration**:
```json
{
  "confluence": {
    "enabled": true,
    "site_alias": "mycompany",     // Must match Conduit config
    "space_key": "DOCS",           // Your Confluence space
    "parent_page_title": "Architecture Docs",
    "page_suffix": " - Technical Review"
  }
}
```

**Confluence Page Naming**:
- Page title: `{repo_name}{page_suffix}`
- Example: "myorg/myrepo - Technical Review"

**Finding Uploaded Pages**:
- Check `state.json` for `confluence_page_id`
- Pages appear under the configured parent page
- Each analysis profile can have its own page

### 6. Troubleshooting Common Issues

**Timeout Errors**:
- Check `errors` array in state.json for timeout messages
- Increase `timeout_minutes` in config.json (max: 10)
- Consider analyzing fewer files: reduce `critical_files_limit`
- Large repositories may need multiple attempts

**"Repository not found" Errors**:
- Verify the GitHub URL is correct and accessible
- For private repos, ensure GitHub token is configured in MCP tools
- Check if repository was deleted or made private

**Confluence Upload Failures**:
- Verify `site_alias` matches a site in Conduit's config.yaml
- Check parent page exists in the specified space
- Ensure Atlassian API token is valid
- Look for detailed error in state.json

**MCP Tool Errors**:
- If tools aren't found, user may need to restart their IDE
- Run `claude mcp list` to verify tools are registered
- Check that all three tools appear: code-understanding, mermaid_image_generator, Conduit

**State.json Shows "processing_started" But No Progress**:
- Sub-agent may have timed out
- Check if repository is very large
- Look for corresponding error entry
- Try running with just that repository

### 7. Advanced Usage

**Processing Specific Repositories**:
- Temporarily comment out other repos in config.json
- Or create a separate config file for testing

**Analyzing Multiple Profiles**:
1. Run with first profile
2. Change `analysis_profile` in config.json
3. Update `page_suffix` for Confluence
4. Run again - creates separate documentation

**Debugging Execution**:
- Use `--verbose` flag for detailed output
- Check `output.json` for execution trace
- Review sub-agent output for specific errors

**Performance Optimization**:
- Reduce `max_tokens` for faster but less detailed analysis
- Lower `critical_files_limit` to analyze fewer files
- Run during off-peak hours for better API performance

## Important Notes

- **NEVER** modify sensitive credentials or API tokens
- Always show the user what changes you're making to config files
- If unsure about an error, check state.json for details
- The tool processes maximum 2 repositories in parallel
- Each repository analysis is independent

## Execution Guidelines

1. Start by understanding what the user wants to accomplish
2. Review their current configuration and state
3. Guide them through changes step by step
4. Execute commands where appropriate
5. Help interpret results and troubleshoot issues
6. Suggest next steps based on their goals

Remember: Your role is to make using this tool as smooth as possible, helping users get the documentation they need for their repositories.