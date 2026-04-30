# CLAUDE.md

This repository is a tool: `gh-repo-code-intelligence`. Its job is to analyze one or more GitHub repositories (listed in `config.json`) and produce audience-tailored documentation, optionally publishing each repository's docs as a child page in Confluence.

## How the work happens — via skills, not this file

The end-to-end workflows live in skills. This file deliberately does **not** contain the procedures themselves — those don't belong in always-loaded project memory.

- **`onboard-new-user`** drives first-time setup: prerequisite checks, MCP server installs and registration, permission file, `config.json` construction. Triggers on phrasings like "help me get started", "I just cloned this", "first time setting up", "how do I run this."
- **`analyze-repos`** runs the multi-repository analysis pipeline against `config.json`. Triggers on "analyze the repos", "run the analysis", "process the repositories", "let's run it", "do the docs", "publish to Confluence now", and similar.

If you find yourself wanting to write instructions about *how* to analyze a repo or *how* to onboard a user, those instructions belong inside the relevant skill — not in this file.

## Repo-level facts worth knowing in any session

- The pipeline produces local artifacts in `output/{repo}/{timestamp}/` (gitignored) and tracks per-repo state in `state.json` (gitignored).
- `config.json` is user-specific and is intentionally not committed; `config.json.template` shows the shape.
- Sub-agents spawned by the analysis pipeline are pinned to **Sonnet 4.6** with **Haiku 4.5** as the overload fallback. Rationale is captured in the commit history (search `git log` for "Pin sub-agents to Sonnet 4.6").
- The required MCP servers (`code-understanding`, `mermaid_image_generator`, optionally `Conduit` for Confluence) must be both **registered** AND **enabled** in `~/.claude.json` before sub-agents can use their tools. `claude mcp list` shows reachability only — `/mcp` shows the actual enable flag. The onboarding skill walks users through this; the analysis skill verifies it as a Phase 1 pre-flight.
