---
name: onboard-new-user
description: Drives end-to-end onboarding for the gh-repo-code-intelligence repository — verifies prerequisites, installs and registers the required MCP servers (code-understanding, mermaid_image_generator, optionally Conduit for Confluence), configures permissions, builds the user's first config.json, and hands off the analysis run command. Trigger this skill whenever the user is working inside the gh-repo-code-intelligence repo and says any of "help me get started", "I just cloned this", "first time setting up", "how do I run this", "I want to analyze a repo", "help me onboard", "how do I use this tool", or describes a setup symptom (missing MCP servers, no config.json, blank state.json, "what do I do first"). Also trigger when the user says they want to wire up Confluence publishing for the first time. Do NOT trigger if the user is clearly past first-run setup and is asking about a specific phase like editing config.json for a new run, debugging a failed analysis, or interpreting output — those are different tasks.
---

# Onboard a new user to gh-repo-code-intelligence

This skill takes a user from a freshly-cloned repo to "ready to produce a stakeholder-tailored documentation page" without making them read a single .md file. Diagnose what is already in place, instruct on what is missing, and hand off the exact run command at the end.

## What this tool does (one-paragraph orientation)

`gh-repo-code-intelligence` analyzes a GitHub repository and produces audience-tailored documentation — developer onboarding, architecture review, ops handover, business overview, or a generic baseline. Output lives locally in `output/{repo}/{timestamp}/` as `docs.md` plus three architectural diagrams (component, sequence, class). When Confluence is enabled, the same content is also published as a Confluence page under a parent the user specifies. The tool is driven entirely by `config.json`; the engine is `CLAUDE.md`, which the user does not need to read. The actual processing happens in Claude sub-agents that the main agent spawns — the user's only inputs are the config and a single CLI command.

If the user asks what the tool does, give them this paragraph and move on. Onboarding is the job; architecture explanation is not.

## Critical rules (read every time)

These four constraints override everything else in this file.

1. **Never ask the user for a secret value.** GitHub Personal Access Tokens, Atlassian API tokens, and any other credentials must be inserted by the user themselves into commands you provide. Show the command with a clearly-marked placeholder (e.g. `YOUR_TOKEN`), explain how to get the secret, and wait for the user to confirm they've run it. Do not paste secrets the user types into chat back into commands or files — discourage them from sharing the value at all.

2. **Always show the literal run command at the end** — even when offering to execute it for the user. The user must be able to copy/paste it and run it themselves at any future moment.

3. **Never auto-execute the analysis command without explicit confirmation in this session.** Running it costs API tokens and clones repositories; offer first, wait for a clear yes.

4. **Always convert SSH GitHub URLs to HTTPS** before writing them to `config.json`. SSH URLs prompt for an SSH passphrase and break the unattended workflow. Convert `git@github.com:org/repo.git` → `https://github.com/org/repo.git` silently and tell the user you did.

## Pattern: external-vendor token guidance

When the user has to create a token at GitHub, Atlassian, or any other vendor's site, **don't pretend to know the current UI**. Vendor UIs change — sometimes monthly. Hardcoded click-through instructions rot. Instead, follow this three-part pattern every time:

1. **Provide the canonical URL.** These tend to be stable for years (e.g. `https://github.com/settings/tokens`, `https://id.atlassian.com/manage-profile/security/api-tokens`). Lead with the URL.
2. **Narrate the last-known click path** — clearly framed as "as of last known, the path was X → Y → Z" so the user understands this could be stale.
3. **Explicitly invite the user to come back if the UI doesn't match.** When they do, *use WebFetch on the vendor's current docs* to figure out the real flow together rather than guessing.

Also surface the gotchas that don't depend on UI: token-shown-only-once, SSO authorization for org-owned private repos, scope-less Atlassian tokens, etc. Those bite even when the user finds the right page.

## The path

Seven phases. Always announce which phase you're entering so the user can pause. **Skip any phase whose work is already done** — diagnose state before instructing.

### Phase 0 — Prerequisites and orientation

Verify before doing anything else:

1. The current working directory is the gh-repo-code-intelligence repo. Tell-tale files: `CLAUDE.md`, `config.json.template`, `.mcp.json.template`, `README.md` mentioning "repository analysis tool". If the user is not in the repo, ask them to `cd` there first.
2. Claude Code CLI is available: `claude --version`.
3. The user has *some* repo in mind they want to analyze, even loosely ("I want dev docs for our internal payments service"). If they have no target yet, ask them to pick one — the rest of onboarding is more concrete with a real target.

If `config.json` already exists with a non-template repo entry and `state.json` shows prior successful runs, the user is past onboarding — say so and ask whether they want to onboard a *new* config or do something else.

### Phase 1 — Permissions

Set up the local Claude Code permissions file:

```bash
cp .claude/settings.local.json.template .claude/settings.local.json
```

If the template is missing or `.claude/settings.local.json` already exists, skip — this file pre-approves common tool calls so unattended runs don't pause for permission prompts. Running with `--dangerously-skip-permissions` (which the documented run command uses) covers the same need, so this phase is a nice-to-have, not blocking.

### Phase 2 — MCP server install and registration

Three MCP servers may be involved. Conduit is only needed if the user wants to publish to Confluence.

| Server | Required when | Provides |
|---|---|---|
| `code-understanding` | always | clone, refresh, structure, critical files |
| `mermaid_image_generator` | always | generate component/sequence/class PNGs |
| `Conduit` | only with Confluence | find/create/update Confluence pages |

#### Detect what is already usable in this session

The reliable source of truth is **your own session's tool availability**, not what's registered. You (the assistant running this skill) already see all MCP tools as `mcp__<server>__<tool>` entries in your tool list. Check whether tools with each of these prefixes are available to you:

- `mcp__code-understanding__*` — required
- `mcp__mermaid_image_generator__*` — required
- `mcp__Conduit__*` — required only if the user wants Confluence publishing

If a prefix is missing from your available tools, that server is not usable in this session, regardless of what's registered.

**Why this matters — `claude mcp list` alone isn't enough:** a server can be registered but disabled (via `/mcp` → Disable). In that state `claude mcp list` happily reports it as Connected while your session cannot actually call its tools. Trusting the registration listing in that case would make the skill falsely declare "you're already set up" when the user can't use anything.

**Secondary cross-reference** (helpful for disambiguation, not authoritative):

```bash
claude mcp list
cat .mcp.json 2>/dev/null
```

Combine the two signals to bucket each required server:

| Tools available to you in session? | Shows in `claude mcp list`? | Diagnosis | What to do |
|---|---|---|---|
| Yes | Yes | Working | Skip install — already usable |
| No | Yes | Registered but disabled | Tell the user: `/mcp` → pick the server → Enable. Then they restart the session. Do NOT re-install. |
| No | No | Not installed | Walk through the install commands below |

Tell the user which of the three required servers fall into which bucket and only run install commands for the entirely-missing ones.

#### Install code-understanding

Public-repo-only setup (no token):

```bash
claude mcp add-json -s user code-understanding '{"type":"stdio","command":"uvx","args":["code-understanding-mcp-server","--max-cached-repos","20"]}'
```

If the user plans to analyze **private** repos, give them the token-injecting variant to run themselves, following the external-vendor token guidance pattern. Do **not** ask for the token value. Use this exact handoff:

> "code-understanding needs a GitHub Personal Access Token to clone private repositories. Public repos work without one — skip this entire step if you only need public.
>
> **Where to create one (canonical URL, stable):** https://github.com/settings/tokens
>
> **Last-known click path** (GitHub may have rearranged this — if the page looks different, tell me and we'll figure out the current flow together; I can fetch GitHub's current docs):
> 1. Sign in to github.com.
> 2. Click your profile picture (top right) → **Settings**.
> 3. Scroll the left sidebar to the bottom → **Developer settings**.
> 4. **Personal access tokens** → **Tokens (classic)**. (Fine-grained tokens also work but are more complex; classic is fine to start with.)
> 5. **Generate new token** → **Generate new token (classic)**.
> 6. Name it (e.g. `gh-repo-code-intelligence`), pick an expiration, and check the **`repo`** scope.
> 7. **Generate token** at the bottom of the page.
> 8. **Copy the token immediately** — GitHub shows it exactly once.
>
> **Org-owned private repos with SSO:** If the repo lives under a GitHub organization that requires SSO, look for an **Authorize** or **Configure SSO** button next to your new token in the token list and click it to authorize the token for that org. Without this step, clones of org repos fail with cryptic 'not found' errors.
>
> When you have the token, **run this command yourself**, replacing `YOUR_TOKEN`:
>
> ```
> claude mcp add-json -s user code-understanding '{"type":"stdio","command":"uvx","args":["code-understanding-mcp-server","--max-cached-repos","20"],"env":{"GITHUB_TOKEN":"YOUR_TOKEN"}}'
> ```
>
> Tell me when it's done — I'll continue. Don't paste the token value here; I don't need to see it. **If anything in the GitHub UI didn't match what I described** (e.g. menu item gone, page restructured), come back and tell me — I'll fetch the current GitHub docs and we'll work it out together rather than guessing."

#### Install mermaid_image_generator

```bash
claude mcp add-json -s user mermaid_image_generator '{"type":"stdio","command":"uvx","args":["mcp_mermaid_image_gen"]}'
```

#### Install Conduit (only if Confluence wanted)

Conduit needs system-level setup *before* it can be registered with Claude Code:

```bash
pipx install conduit-connect
conduit --init
```

Then point the user at the config file and step away from the secret. Use this handoff exactly (it follows the external-vendor token guidance pattern):

> "Conduit needs your Atlassian credentials. Open `~/.config/conduit/config.yaml` (macOS/Linux) or `%APPDATA%\conduit\config.yaml` (Windows) and add your site under a chosen alias. The shape is:
>
> ```yaml
> sites:
>   YOUR_ALIAS:
>     url: https://YOUR_DOMAIN.atlassian.net
>     email: you@example.com
>     api_token: YOUR_ATLASSIAN_API_TOKEN
> ```
>
> What goes in each field:
> - **`YOUR_ALIAS`** — a short name you choose (e.g. `acme`, `work`, `ctf`). You'll use this exact alias as `confluence.site_alias` in `config.json` later, so pick something short and memorable.
> - **`url`** — your full Atlassian Cloud URL (e.g. `https://acme.atlassian.net`).
> - **`email`** — the email address you use to sign in to Atlassian.
> - **`api_token`** — see below.
>
> **Where to create the API token (canonical URL, stable):** https://id.atlassian.com/manage-profile/security/api-tokens
>
> **Last-known click path** (Atlassian rearranges this page periodically — if it doesn't match, tell me and we'll figure out the current flow):
> 1. Sign in at https://id.atlassian.com.
> 2. Find **Security** in the navigation (sidebar or top — depends on the layout currently in use).
> 3. **API tokens** section → **Create API token**.
> 4. Give it a label (e.g. `gh-repo-code-intelligence`) → **Create**.
> 5. **Copy the token immediately** — Atlassian shows it exactly once. If you lose it, you'll have to revoke and create a new one.
>
> Things worth knowing about Atlassian tokens (don't depend on the UI, so they don't go stale):
> - The token has **no scopes** — one token works for Confluence, Jira, and everything else on your account. (This trips up people coming from GitHub.)
> - The token inherits your user account's permissions exactly. If you can't see a Confluence space in the browser, the token can't either.
>
> Tell me what alias you picked (the alias is fine to share; the token is not). Once saved, I'll register Conduit with Claude Code. **If the Atlassian UI didn't match what I described**, come back and we'll fetch the current docs together."

After confirmation:

```bash
claude mcp add-json -s user Conduit '{"type":"stdio","command":"mcp-server-conduit","args":[]}'
```

### Phase 3 — Verify MCP registration

MCP servers only load at Claude Code session start. Tell the user:

> "Quick — exit this Claude Code session (Ctrl+D or `exit`), then reopen with `claude` from this same directory. Once you're back, run `claude mcp list` and paste the output. I need to see all the servers we just registered show up there before we go further."

When they return, confirm the expected servers are listed. If anything is missing, look at the error output or ask them to re-run the install command for that server.

### Phase 4 — Build config.json

Walk the user through the choices. Don't write the file until you have all the inputs.

1. **Target repository.** Ask for the GitHub URL. If they give SSH, silently convert and mention it. The shape:
   ```
   name: org/repo-name
   github_url: https://github.com/org/repo-name.git
   ```

2. **Audience profile.** Ask who is going to read the documentation, then map the answer to a profile. Don't make the user pick from a list of names — pick *for* them based on their answer:

   | Audience the user names | Profile to use | Suggested page_suffix |
   |---|---|---|
   | "new developers", "people joining the team" | `developer_onboarding` | ` - Developer Onboarding` |
   | "architects", "tech leads", "design review" | `architecture_review` | ` - Architecture Review` |
   | "PMs", "business folks", "product team" | `business_understanding` | ` - Business Overview` |
   | "DevOps", "SRE", "operations", "on-call" | `operations_handover` | ` - Operations Handover` |
   | mixed / not sure / general | `standard` | (none) |

3. **Confluence target (skip if Confluence not wanted).** Ask for:
   - `site_alias` (must match what they used in `~/.config/conduit/config.yaml`)
   - `space_key` (e.g. `ENG`, `DEMO`)
   - `parent_page_title` — the existing parent page that new analysis pages will be created under. **Critical:** this page must already exist in Confluence; the tool will not create it. Confirm with the user that it exists.
   - `page_suffix` — recommend the value matching the chosen profile (table above).

Write `config.json` at the repo root using this shape:

```json
{
  "repos": [
    {
      "name": "org/repo-name",
      "github_url": "https://github.com/org/repo-name.git"
    }
  ],
  "preferred_diagrams": ["component", "sequence", "class"],
  "doc_format": "markdown",
  "analysis_options": {
    "critical_files_limit": 100,
    "include_metrics": true,
    "max_tokens": 5000,
    "analysis_profile": "developer_onboarding"
  },
  "confluence": {
    "enabled": true,
    "site_alias": "...",
    "space_key": "...",
    "parent_page_title": "...",
    "page_suffix": " - Developer Onboarding"
  }
}
```

If Confluence is not wanted, set `confluence.enabled: false` and leave the rest of the block as harmless placeholders (or remove it; both work).

### Phase 5 — Pre-flight check

Before showing the run command, run through these in order with the user:

1. Repo URL is HTTPS (not `git@`). You should already have ensured this in Phase 4.
2. Profile and `page_suffix` agree (no mismatched suffix on a different profile).
3. If Confluence enabled: explicitly confirm with the user that `parent_page_title` exists in `space_key`. The most common first-run failure is the parent page not existing — the local docs+diagrams are fine, but Confluence upload silently fails.
4. `state.json` does not have a `processing_started` lock left over from a crashed prior run for this repo entry. If it does, suggest clearing it.

### Phase 6 — The run command and (optional) execution

Always present the literal run command:

```bash
claude -p "Analyze repositories according to CLAUDE.md" \
  --dangerously-skip-permissions \
  --verbose --output-format stream-json > output.json
```

Briefly explain:

- The terminal will look frozen because stdout is being captured into `output.json`. That's normal — the run is happening, you just can't see it.
- Wall-clock for one repo is typically 3–8 minutes depending on profile and repo size.
- **Live progress** can be watched in another terminal:
  - `state.json` — high-level phase tracker (sub-agent updates this as it advances)
  - `output/{repo_with_underscores}/{timestamp}/` — files appear as they're written
  - `tail -f output.json | jq -rc 'select(.type=="assistant") | .message.content[]? | select(.type=="tool_use") | "→ \(.name)"'` — current tool calls

Then offer to execute:

> "Want me to kick this off now, or would you rather inspect `config.json` once more and run it yourself? Either way, the command is yours to keep."

If the user says yes, run via Bash and stream-monitor `state.json` in the background. If no, leave it.

### Phase 7 — Verify output

After the run completes (or after the user runs it themselves and reports back):

1. Read `state.json` and confirm the user's repo entry has `success: true`, a `confluence_page_id` (if applicable), and an entry in `profiles_analyzed`.
2. Check that `output/{repo_with_underscores}/{timestamp}/` contains `docs.md` and three diagram PNGs.
3. If Confluence enabled, offer to open the new page in browser (the URL pattern is `https://YOUR_SITE.atlassian.net/wiki/spaces/{space_key}/pages/{confluence_page_id}/`).
4. If `success: false`, look at `state.json` `errors` array for diagnostic data and `output.json` for sub-agent transcripts.

## Common gotchas

Watch for these during onboarding:

- **MCP servers don't reload mid-session.** After installing or modifying any MCP server, the user must restart Claude Code. This catches people every time. Don't skip Phase 3.
- **Conduit `site_alias` mismatch.** The alias in `~/.config/conduit/config.yaml` must exactly match `confluence.site_alias` in `config.json`. A mismatch produces "site not found" with no obvious fix.
- **Parent Confluence page doesn't exist.** The tool will not create it. The local docs will be fine but Confluence upload step will fail. Confirm in Phase 5.
- **`cloned: false` in state.json during a run is not a failure signal.** Sub-agents batch state.json writes; clone often succeeds and the field updates only at the end. If you need ground-truth progress, look at the cache dir or `output/`.
- **SSH GitHub URLs.** Always convert. SSH prompts for passphrase and breaks unattended automation.

## When NOT to use this skill

- The user is already mid-analysis and is asking about progress, errors, or output interpretation — that's a debugging/observability task.
- The user is asking how the tool works under the hood (sub-agent model, MCP plumbing) — that's research, not onboarding.
- The user has clearly already onboarded and is now configuring a new run for a different repo or profile — help them edit `config.json` directly without re-running install/setup phases.
- The user wants to develop or change the tool itself (modify `CLAUDE.md`, change the sub-agent model, etc.) — that's tool development, not onboarding.
