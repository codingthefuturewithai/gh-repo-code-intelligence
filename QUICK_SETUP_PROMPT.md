# Quick Setup Instructions for Experienced Users

---

## To the AI Assistant

Quick setup for gh-repo-code-intelligence. The user is experienced with command line tools.

Steps:

1. Check prerequisites: `python --version`, `pipx --version`, `claude --version`

2. Setup permissions:
   ```bash
   cp .claude/settings.local.json.template .claude/settings.local.json
   ```

3. Install MCP tools:
   ```bash
   # Mermaid - execute this
   claude mcp add-json -s user mermaid_image_generator '{"type":"stdio","command":"uvx","args":["mcp_mermaid_image_gen"]}'
   
   # Code Understanding - check if user needs private repo access
   # If yes: SECURITY BOUNDARY - prepare command with YOUR_GITHUB_TOKEN placeholder
   # If no: execute without token
   
   # Conduit - execute these
   pipx install conduit-connect
   conduit --init
   ```

4. For Conduit config.yaml - show structure but DO NOT handle credentials:
   ```yaml
   confluence:
     sites:
       SITE_ALIAS:
         url: "https://YOUR_DOMAIN.atlassian.net"
         email: "YOUR_EMAIL"
         api_token: "YOUR_API_TOKEN"  # User must add this manually
   ```

5. After user confirms Conduit is configured:
   ```bash
   claude mcp add-json -s user Conduit '{"type":"stdio","command":"mcp-server-conduit","args":[]}'
   ```

6. Setup config.json from template - guide through options

7. Note: MCP tools need IDE restart to be detected. Provide appropriate restart instructions.