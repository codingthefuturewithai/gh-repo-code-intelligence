# AI Assistant Setup Prompt

Copy and paste the following prompt to your AI coding assistant (Claude Code, Cursor, etc.) to get guided setup help.

## What to Expect

When you use this prompt, your AI assistant will:
- ✅ Check all your prerequisites automatically
- ✅ Execute commands for you where possible  
- ✅ Guide you through manual steps with clear instructions
- ✅ Help troubleshoot any errors that occur
- ✅ Verify everything is working before declaring success

The entire setup typically takes 5-10 minutes with AI assistance vs 30-45 minutes manually.

---

## Setup Assistant Prompt

I need help setting up the GitHub Repository Analysis Tool from https://github.com/codingthefuturewithai/gh-repo-code-intelligence. Please read the README.md file in this repository and guide me through the complete setup process.

Here's what I need you to do:

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
   - For the Code Understanding tool:
     - Check if I need a GitHub Personal Access Token (for private repos)
     - Help me create one if needed at https://github.com/settings/tokens
     - Add the tool with the correct token
   - For the Mermaid Image Generator:
     - Add it using the simple command
   - For Conduit (the tricky one):
     - First check if I have pipx installed
     - Install Conduit: `pipx install conduit-connect`
     - Run initialization: `conduit --init`
     - Guide me to edit the config.yaml file (you can't edit it directly, but show me exactly what to add)
     - Help me get an Atlassian API token if I want Confluence integration
     - Add the MCP server after it's configured

5. **Configure config.json**:
   - Copy the template to create config.json
   - Help me understand each configuration option
   - Guide me in adding my repositories to analyze
   - If I want Confluence integration, help me configure those settings

6. **Verify Setup**:
   - Run some test commands to ensure everything is working
   - Check that all MCP tools are properly registered
   - Provide troubleshooting if anything fails

7. **First Run**:
   - Show me how to run the tool for the first time
   - Explain what to expect during analysis
   - Help interpret any error messages

Please execute commands where you can, and for steps you cannot execute (like editing config files I need to manually change), provide clear, specific instructions with examples.

Start by checking my current environment and installed tools, then guide me step by step. If you encounter any errors, help me troubleshoot them before moving to the next step.

---

## Additional Context for Complex Setups

If you're helping me with Confluence integration, here's additional context:
- I'll need an Atlassian account with access to a Confluence space
- The Conduit config.yaml needs my Atlassian credentials
- The site_alias in config.json must match what I configure in Conduit

Please be patient and thorough - I want to make sure everything is set up correctly the first time!