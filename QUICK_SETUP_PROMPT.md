# Quick Setup Prompt for Experienced Users

For a faster setup if you're comfortable with command line tools:

---

I need to set up gh-repo-code-intelligence. Please:

1. Check prerequisites: Python 3.10+, pipx, Claude Code CLI
2. Run these setup commands:
   ```bash
   # Copy permissions template
   cp .claude/settings.local.json.template .claude/settings.local.json
   
   # Install and setup MCP tools
   claude mcp add-json -s user mermaid_image_generator '{"type":"stdio","command":"uvx","args":["mcp_mermaid_image_gen"]}'
   
   # For Code Understanding (I'll add my GitHub token if needed)
   claude mcp add-json -s user code-understanding '{"type":"stdio","command":"uvx","args":["code-understanding-mcp-server","--max-cached-repos","20"],"env":{"GITHUB_PERSONAL_ACCESS_TOKEN":"YOUR_GITHUB_TOKEN"}}'
   
   # Install Conduit
   pipx install conduit-connect
   conduit --init
   ```

3. Show me the Conduit config.yaml format for adding Atlassian credentials
4. Add Conduit MCP server after I configure it
5. Help me create config.json from the template with my repositories

Execute what you can and guide me through the rest.

---