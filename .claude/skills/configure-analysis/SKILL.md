---
name: configure-analysis
description: Help the user configure or update `config.json` for the next analyze-repos pipeline run — add or remove repositories, choose or change the analysis profile (developer onboarding / architecture review / business overview / ops handover / standard), set or update the Confluence target (site_alias, space_key, parent page, page suffix), and verify the config is sane before they invoke the pipeline. Trigger this skill whenever the user, working in the gh-repo-code-intelligence repo and past initial onboarding, says any of "configure a new run", "set up a new analysis", "configure the analysis", "add a repo to config", "remove this repo from config", "change the profile", "switch the audience to X", "edit config.json", "I want to analyze a different repository", "swap the Confluence parent page", "I want to run a new analysis with different settings", or "let's set up the next run." Trigger even when the user does not say the word "configure" — phrases like "I want to do an architecture review of foo/bar", "let's prep an ops handover for the payments service", or "set me up to analyze acme/billing" should fire this skill because they imply rewriting `config.json`. Do NOT trigger for first-time setup (use `onboard-new-user` — that builds the very first config), do NOT trigger for actually running the pipeline (use `analyze-repos`), and do NOT trigger for diagnosing failed runs.
---

# Configure the next analyze-repos run

This skill produces a `config.json` that's ready for the `analyze-repos` pipeline. It handles three cases gracefully: no `config.json` yet, an existing `config.json` the user wants to edit, or an existing `config.json` the user wants to replace entirely. Each case starts the same way: read what's there, then ask what the user wants.

The user's job is to describe what they want analyzed and who the docs are for. Your job is to translate that into a valid `config.json` and confirm before writing.

## Assumptions

This skill assumes the user has already completed `onboard-new-user`. That means:
- The required MCP servers are installed AND enabled in `/mcp` (`code-understanding`, `mermaid_image_generator`, and `Conduit` if they used Confluence in onboarding).
- Claude Code permissions are configured.
- If Confluence is in play, the user's Conduit credentials are already in `~/.config/conduit/config.yaml` and they remember their `site_alias`.

Do **not** re-verify any of those — that's onboarding's job. If the user has not done onboarding, redirect them to `onboard-new-user` and stop.

This skill assumes **nothing** about whether `config.json` or `state.json` exists. Diagnose first, instruct second.

## Critical rules

1. **Never ask the user for a secret value.** Conduit's API token is already in `~/.config/conduit/config.yaml`; you don't need to know it and you must not ask. Same for any other credential. If a setting reasonably needs a secret, point at where the user manages it instead of capturing it in chat.
2. **Always convert SSH GitHub URLs to HTTPS** before writing them to `config.json`. SSH URLs prompt for credentials and break the unattended pipeline. Convert `git@github.com:org/repo.git` → `https://github.com/org/repo.git` silently, and tell the user once that you did.
3. **Never run the pipeline from this skill.** Configuration ends with the `config.json` written and a hand-off line. The user (or `analyze-repos`) does the run.
4. **Never modify `state.json`** unless the user explicitly asks for a clean slate. If they want a fresh run with no leftover state, offer to delete `state.json` and `output/` separately — don't decide for them.

## The path

Announce the phase you're in so the user can pause. Skip phases whose work is already done.

### Phase 0 — Diagnose current state

Read what's actually on disk so the rest of the conversation is grounded:

1. Does `config.json` exist at the repo root? If yes, read it and note: how many repos, current `analysis_profile`, whether Confluence is enabled, the `site_alias` and `parent_page_title` if so.
2. Does `state.json` exist? If yes, briefly note which repos have prior `success: true` entries — purely informational, doesn't drive any decision.
3. Does `config.json.template` exist? It usually does — useful as a fallback shape if the user wants to start from scratch.

Tell the user what you found in one or two lines. Don't dump the whole config — just enough that they know where they're starting from.

### Phase 1 — Determine intent

Map the user's request to one of these:

- **No config.json or user wants a clean slate** → build a new config from scratch (continue with all phases).
- **Add a repo to existing config** → preserve the existing `analysis_options` and `confluence` blocks; only Phase 2 runs to collect the new repo, then jump to Phase 5.
- **Remove a repo from existing config** → ask which one, drop it from `repos[]`, jump to Phase 5.
- **Change the analysis profile** → re-run Phase 3 only, then offer to update `confluence.page_suffix` to match (Phase 4-lite), then Phase 5.
- **Change the Confluence target** → re-run Phase 4, then Phase 5.
- **Edit something specific** the user named → run only the relevant phase(s), then Phase 5.

If their request is ambiguous (e.g., they just said "configure a new run" but a config already exists), ask: "There's already a config for `org/repo-name` with the `architecture_review` profile. Do you want to add to it, replace it, or modify a specific field?"

### Phase 2 — Repository intake

For each repo the user wants in the run:

1. Ask for the GitHub URL.
2. If it's SSH, convert to HTTPS silently and mention you did.
3. Extract the `org/repo-name` part — that becomes the `name` field.
4. Repeat until they're done. Multiple repos are fine; the pipeline processes them in batches of two.

The shape per repo:

```json
{
  "name": "org/repo-name",
  "github_url": "https://github.com/org/repo-name.git"
}
```

### Phase 3 — Audience profile

Don't make the user pick from a list of profile names. Ask who is going to read the documentation, then map their answer to a profile yourself.

| Audience the user names | Profile to use | Suggested page_suffix |
|---|---|---|
| "new developers", "people joining the team", "onboarding engineers" | `developer_onboarding` | ` - Developer Onboarding` |
| "architects", "tech leads", "design review", "principal engineers" | `architecture_review` | ` - Architecture Review` |
| "PMs", "business folks", "product team", "executives" | `business_understanding` | ` - Business Overview` |
| "DevOps", "SRE", "operations", "on-call", "infra team" | `operations_handover` | ` - Operations Handover` |
| "mixed", "not sure", "general technical audience" | `standard` | (none) |

Brief description per profile (so you can answer "what's the difference?" if asked):
- **`standard`** — general-purpose technical doc, balanced. ~15–20 pages.
- **`developer_onboarding`** — tech stack, dev setup, code organization, design patterns, key workflows. ~25–30 pages.
- **`architecture_review`** — system design, architectural decisions, scalability, technology rationale, extensibility points. ~25–30 pages.
- **`business_understanding`** — business capabilities, domain concepts, user journeys, less technical detail. ~20–25 pages.
- **`operations_handover`** — deployment, configuration, monitoring, dependencies, runbook-style. ~20–25 pages.

### Phase 4 — Confluence target (skip if user does not want Confluence)

Ask if they want Confluence publishing for this run. If no, set `confluence.enabled: false` and move on.

If yes, collect:

1. **`site_alias`** — the alias they configured during onboarding (in `~/.config/conduit/config.yaml`). They should know this; if they're unsure, point them at that file. Don't read it yourself.
2. **`space_key`** — the Confluence space (e.g., `ENG`, `DEMO`, `CTF`).
3. **`parent_page_title`** — the **exact** title of the existing page under which new analysis pages will be created. The pipeline does **not** create the parent. Critical: the user must confirm this page already exists in the configured space, or the upload step will fail (local docs+diagrams will still be produced — only Confluence upload fails).
4. **`page_suffix`** — recommend the value matching the chosen profile (table in Phase 3). Empty string is fine for the `standard` profile.

The full Confluence page title will be `{repository_name}{page_suffix}` — e.g., `acme/payments-service - Architecture Review`. Show the user what title(s) will be created so they can confirm.

### Phase 5 — Compose and confirm

Build the full `config.json` in memory using the shape below. Show it to the user before writing. Ask one final yes/no: "Write this to `config.json`?"

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
    "analysis_profile": "architecture_review"
  },
  "confluence": {
    "enabled": true,
    "site_alias": "...",
    "space_key": "...",
    "parent_page_title": "...",
    "page_suffix": " - Architecture Review"
  }
}
```

If `confluence.enabled` is false, leave the rest of the Confluence block as harmless placeholders or strip it — both work.

### Phase 6 — Write and pre-flight

Write `config.json` to the repo root. Then verify quickly:

1. Every `github_url` is HTTPS (not `git@`). Re-check after writing in case anything slipped through Phase 2's conversion.
2. `analysis_profile` matches one of the five known values.
3. If `confluence.enabled: true`, all four Confluence fields are non-empty.
4. The `analysis_profile` and `page_suffix` are coherent (e.g., `architecture_review` profile with ` - Architecture Review` suffix, not a developer suffix).

If `state.json` exists from prior runs, mention it briefly: "There's a `state.json` from a previous run. The pipeline treats past errors as historical and will retry everything fresh, so it's safe to leave alone. If you want a clean slate (delete `state.json` and `output/`), say so and I'll do it." Don't delete without explicit permission.

### Phase 7 — Hand off

Tell the user the config is ready and how to invoke the run. Two equivalent options:

- **Natural-language invocation:** open a fresh Claude Code session in this directory and say something like "analyze the repos" or "let's run the analysis." The `analyze-repos` skill triggers and drives the pipeline.
- **Legacy CLI form** (still supported):
  ```
  claude -p "Analyze repositories according to CLAUDE.md" \
    --dangerously-skip-permissions \
    --verbose --output-format stream-json > output.json
  ```

Mention that wall-clock for one repo is typically 3–8 minutes depending on profile and repo size.

Do **not** run the pipeline yourself. This skill ends here.

## Common pitfalls

- **Parent Confluence page doesn't exist.** Most common first-run failure. Confirm with the user that the named page exists in the named space *before* writing the config, ideally by them clicking through to it in their browser.
- **`site_alias` mismatch.** The alias in `config.json`'s `confluence.site_alias` must exactly match the alias key in `~/.config/conduit/config.yaml`. A mismatch produces "site not found" with no obvious fix.
- **Profile and page_suffix drift.** Easy to set the profile to `architecture_review` but leave the suffix as ` - Developer Onboarding` from a prior run. Phase 6 catches this.
- **SSH GitHub URLs.** Always convert. Don't write `git@github.com:...` into `config.json`.
- **Forgetting that `state.json` from prior runs is fine to leave alone.** The pipeline treats historical errors as moments in time, not predictions. Don't reflexively wipe it.

## When NOT to use this skill

- The user has not completed `onboard-new-user` yet — redirect them there. This skill assumes the MCP servers, permissions, and Conduit auth are already in place.
- The user wants to actually run the pipeline now — that's `analyze-repos`. Configuration ends here; running starts there.
- The user is debugging a failed run — that's a different task. Help them inspect `state.json` errors and `output.json` transcripts directly.
- The user is asking how the analysis pipeline works under the hood — that's research, not configuration.
