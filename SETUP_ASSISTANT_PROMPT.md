# Setup Assistant Instructions

This file contains instructions for AI coding assistants to help users set up this tool.

---

## To the AI Assistant

You are helping set up the GitHub Repository Analysis Tool. First, review these key documentation resources to understand the tools involved:

1. **Claude Code Documentation**: Review https://docs.anthropic.com/en/docs/claude-code/settings to understand:
   - How Claude Code permissions work
   - MCP (Model Context Protocol) tool configuration
   - Configuration file locations and formats

2. **Conduit Documentation**: Review https://github.com/codingthefuturewithai/conduit to understand:
   - Installation process
   - Configuration file structure
   - How to set up Atlassian authentication

3. **This Repository**: Read the README.md file in this repository for specific setup steps

## Setup Tasks to Perform

1. **Review Prerequisites**: Check if I have all required tools installed:
   - Python 3.10+ (check with `python --version` or `python3 --version`)
   - pip/pipx/uv (check with `pipx --version`)
   - Claude Code CLI (check with `claude --version`)
   - Git (check with `git --version`)

2. **Clone and Setup Repository**:
   - Clone the repository if not already done
   - Create necessary directories
   - Copy template files where needed

3. **Configure Claude Permissions**:
   - Help me copy the `.claude/settings.local.json.template` to `.claude/settings.local.json`
   - Explain what permissions are being granted and why they're needed

4. **Setup MCP Tools** (this is the most complex part):
   - Guide me through adding the three required MCP tools
   
   **For Mermaid Image Generator**:
   - Execute: `claude mcp add-json -s user mermaid_image_generator '{"type":"stdio","command":"uvx","args":["mcp_mermaid_image_gen"]}'`
   
   **For Code Understanding tool**:
   - First ask: "Will you be analyzing any private GitHub repositories?"
   - If NO: Execute the command without a GitHub token
   - If YES: 
     - **SECURITY BOUNDARY - STOP HERE**
     - Instruct user: "You need to create a GitHub Personal Access Token"
     - Direct them to: https://github.com/settings/tokens
     - Show them the exact scopes needed (repo access)
     - Prepare the command but with placeholder: `YOUR_GITHUB_TOKEN`
     - Instruct user to execute the command manually after replacing the token
   
   **For Conduit**:
   - Check if pipx is installed
   - Install Conduit: `pipx install conduit-connect`
   - Run initialization: `conduit --init`
   - Show the user the exact config.yaml structure they need
   - If they want Confluence integration:
     - **SECURITY BOUNDARY - STOP HERE**
     - Direct them to create Atlassian API token at: https://id.atlassian.com/manage-profile/security/api-tokens
     - Show config.yaml structure with placeholder for api_token
     - Have them manually edit the file to add their credentials
   - After user confirms configuration is complete, add the MCP server

5. **Configure config.json**:
   - Copy the template to create config.json
   - Help me understand each configuration option
   - Guide me in adding my repositories to analyze
   - If I want Confluence integration, help me configure those settings

6. **Verify Setup**:
   - **MCP Tool Verification Note**: Newly installed MCP tools typically require restarting your IDE/editor
   - Determine the appropriate restart procedure for your environment:
     - For VS Code with Continue/Codeium: May need to restart VS Code
     - For Claude Code: May need to restart the application
     - For Cursor: May need to restart Cursor
   - Provide the user with verification commands to run AFTER restart:
     ```bash
     # These commands should be run by the user after restarting:
     claude mcp list  # Should show all three MCP tools
     ```
   - If verification fails, provide troubleshooting steps

7. **First Run**:
   - Show the user how to run: `claude -p "Analyze repositories according to CLAUDE.md" --dangerously-skip-permissions`
   - Explain what to expect during analysis
   - Provide guidance on common issues and their solutions

## Important Security Notes

- **NEVER** handle or store API keys, tokens, or passwords
- **ALWAYS** stop at security boundaries and instruct the user to handle sensitive data manually
- **CLEARLY** mark where automated assistance ends and manual steps begin

## Execution Guidelines

1. Execute commands where safe and appropriate
2. For configuration files that need sensitive data, show the structure but have user edit manually
3. Be explicit about what you're doing vs what the user needs to do
4. If unsure about your IDE's MCP tool detection, explain the options to the user

Start by checking the current environment and installed tools, then proceed step by step.