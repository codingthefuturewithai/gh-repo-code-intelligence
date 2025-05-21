# GitHub Repository Analysis Tool

This tool uses Claude Code's MCP capabilities to automatically analyze GitHub repositories, generate documentation, and create visual diagrams of code architecture.

## Features

- **Repository Analysis**: Clone and analyze GitHub repositories
- **Code Structure Mapping**: Generate maps of code organization and relationships
- **Critical File Identification**: Highlight the most important files in the repository
- **Diagram Generation**: Create component, sequence, and class diagrams
- **Documentation Creation**: Generate comprehensive markdown documentation

## Roadmap

- **Coming Soon**: Integration with Atlassian Confluence using MCP tools to automatically publish documentation to your team's Confluence spaces.

## Setup

1. Clone this repository:
   ```bash
   git clone https://github.com/your-username/gh-repo-code-intelligence.git
   cd gh-repo-code-intelligence
   ```

2. Create a `config.json` file based on the template:
   ```bash
   cp config.json.template config.json
   ```

3. Edit `config.json` to include the repositories you want to analyze:
   ```json
   {
     "repos": [
       {
         "name": "organization/repository-name",
         "github_url": "https://github.com/organization/repository-name"
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

## Usage

Run the tool using Claude Code:

```bash
cd /path/to/gh-repo-code-intelligence
claude -p "Analyze repositories according to CLAUDE.md"
```

This will:

1. Read repositories from your `config.json`
2. Create or update `state.json` to track progress
3. Clone or refresh each repository 
4. Analyze the repository structure
5. Generate diagrams based on the code analysis
6. Create comprehensive documentation with embedded diagrams
7. Store all outputs in the `output/` directory with timestamped folders

## Output Structure

```
output/
└── {repo_name}/
    └── {timestamp}/
        ├── docs.md        <- Generated documentation
        └── diagrams/      <- Generated diagrams
            ├── component_diagram.png
            ├── sequence_diagram.png
            └── class_diagram.png
```

## Requirements

- Claude Code CLI
- MCP tools access (code-understanding and mermaid_image_generator)

## Configuration Options

See `config.json.template` for the full configuration options. Key settings include:

- `repos`: List of repositories to analyze
- `preferred_diagrams`: Types of diagrams to generate
- `analysis_options`: Settings for code analysis depth and detail

## Future Enhancements

We're actively working on enhancing this tool with more features:

- **Confluence Integration**: Automatically publish documentation to Atlassian Confluence spaces
- **Documentation Templates**: Customizable templates for different types of documentation
- **Enhanced Diagram Types**: Additional diagram formats for more comprehensive visualization
- **Automated Scheduling**: Integrated scheduling for regular repository analysis updates

## License

MIT