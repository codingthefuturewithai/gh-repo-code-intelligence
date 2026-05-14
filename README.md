# GitHub Repository Analysis Tool

`gh-repo-code-intelligence` analyzes one or more GitHub repositories and produces audience-tailored documentation with architectural diagrams. Output lands on disk and, optionally, as Confluence pages.

There is no application code to install. The "engine" is a set of instructions that an AI coding assistant (Claude Code, Cursor, Codex, etc.) follows — driven by three skills shipped with this repo.

## Getting started — just talk to your coding assistant

After cloning this repo, open it in your AI coding assistant of choice and run one of the three skills below. The skills do all the work: prerequisite checks, MCP server installs, permission files, building `config.json`, and running the pipeline.

```bash
git clone https://github.com/codingthefuturewithai/gh-repo-code-intelligence.git
cd gh-repo-code-intelligence
# then launch Claude Code / Cursor / Codex / your assistant in this directory
```

| When you want to… | Run this skill | Or just say… |
| --- | --- | --- |
| Set up the tool for the first time | `/onboard-new-user` | "help me get started", "I just cloned this, what do I do" |
| Configure a new analysis run (repos, profile, Confluence target) | `/configure-analysis` | "configure a new run", "I want to do an architecture review of foo/bar" |
| Actually run the pipeline against `config.json` | `/analyze-repos` | "analyze the repos", "run the analysis", "publish to Confluence" |

You don't need to type the slash-command form — the natural-language phrases trigger the same skills. The skills know which step you're at and what's missing.

### Which assistant?

Any AI coding assistant that supports slash-skills and MCP servers will work — Claude Code is the reference environment and the one the skills were authored against. Cursor, Codex, and similar tools will follow the same skill definitions; you may need to start the relevant skill manually if the host doesn't auto-trigger from natural language.

## How the pipeline actually works

If you want a tour of what the tool does end-to-end — how each phase fits together, what the sub-agents do, where outputs land — see [`docs/WALKTHROUGH.md`](docs/WALKTHROUGH.md). It's a short read and a good orientation before your first run.

## What you'll get

For each repository in `config.json`, the tool produces:

- A markdown document tailored to a chosen audience (developer onboarding, architecture review, business overview, ops handover, or standard).
- Component, sequence, and class diagrams rendered as PNGs.
- Optionally, a Confluence child page under a parent you pick.

Local output:

```
output/
└── {repo_name}/
    └── {timestamp}/
        ├── docs.md
        └── diagrams/
            ├── component_diagram.png
            ├── sequence_diagram.png
            └── class_diagram.png
```

Sample outputs are in [`examples/`](examples/).

## Analysis profiles

The same repository can be analyzed multiple times with different profiles to produce different documents:

| Profile | Audience |
| --- | --- |
| `standard` (default) | General technical |
| `developer_onboarding` | New developers joining the project |
| `architecture_review` | Architects, tech leads |
| `business_understanding` | Product managers, business stakeholders |
| `operations_handover` | DevOps and operations |

`/configure-analysis` walks you through choosing one. If you publish to Confluence, set `confluence.page_suffix` (e.g. `" - Developer Guide"`) so multiple profiles for the same repo don't overwrite each other.

## Requirements

- An AI coding assistant with skill + MCP support (Claude Code recommended).
- Python 3.10+ and `pipx` / `uvx` (the MCP servers are Python packages).
- Three MCP servers — `code-understanding`, `mermaid_image_generator`, and (for Confluence) `Conduit`. The `/onboard-new-user` skill installs and registers these for you.
- A GitHub Personal Access Token if you'll analyze private repositories.
- For Confluence publishing: an Atlassian API token and a Confluence space you can write to.

## Manual / advanced usage

The three skills cover essentially every workflow. If you need to drive the pipeline outside an interactive assistant session — e.g. from a cron job — you can invoke it headlessly:

```bash
claude -p "Analyze repositories according to CLAUDE.md" \
  --dangerously-skip-permissions \
  --verbose --output-format stream-json > output.json
```

`--dangerously-skip-permissions` is required because the pipeline spawns sub-agents that need to run MCP tools without per-call prompts. `config.json` must already exist; use `/configure-analysis` (or copy `config.json.template`) to build it.

## License

MIT — see [LICENSE](LICENSE).
