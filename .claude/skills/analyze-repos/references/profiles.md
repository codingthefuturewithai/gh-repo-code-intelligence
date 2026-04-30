# Analysis profiles

The `analysis_options.analysis_profile` field in `config.json` selects who the documentation is for. Each profile generates a complete, self-contained `docs.md` optimized for its intended audience.

The profile changes:
- Which sections to emphasize and which to compress
- The tone (technical vs. business)
- The vocabulary and amount of jargon
- Which diagram types to prioritize
- The expected page count

## The five profiles

### 1. `standard` (default)

- General-purpose technical documentation.
- Balanced coverage of architecture, code, and functionality.
- Suitable for mixed technical audiences.
- Output: ~15–20 pages.
- Suggested `page_suffix`: empty (`""`).

### 2. `developer_onboarding`

- Focused on helping new developers understand and work with the codebase.
- Emphasizes: technology stack, development setup, code organization, design patterns.
- Includes: how to navigate code, key workflows, integration points, common pitfalls.
- Tone: friendly, oriented toward "what do I need to know to start contributing."
- Output: ~25–30 pages.
- Suggested `page_suffix`: ` - Developer Onboarding`.

### 3. `architecture_review`

- Designed for architects and technical leads.
- Emphasizes: system design, architectural decisions, patterns, scalability, technology rationale.
- Includes: service boundaries, extensibility points, trade-offs, where the system might bend or break.
- Tone: analytical, focused on "why these choices, what are the implications."
- Output: ~25–30 pages.
- Suggested `page_suffix`: ` - Architecture Review`.

### 4. `business_understanding`

- Targeted at product managers and business stakeholders.
- Emphasizes: business capabilities, domain concepts, user-facing workflows.
- Includes: user journeys, business rules, what the system does in plain language. Less technical detail.
- Tone: outcome-oriented, light on code references and jargon.
- Output: ~20–25 pages.
- Suggested `page_suffix`: ` - Business Overview`.

### 5. `operations_handover`

- Created for DevOps and operations teams.
- Emphasizes: deployment, configuration, monitoring, dependencies, runbook-style information.
- Includes: infrastructure needs, operational workflows, integration points, what to watch in production.
- Tone: pragmatic, focused on "what can break and how to keep it running."
- Output: ~20–25 pages.
- Suggested `page_suffix`: ` - Operations Handover`.

## How profiles affect the sub-agent

When a sub-agent receives its prompt, the profile is passed through. The sub-agent must:

1. Adjust its analysis focus per the emphasis listed above.
2. Tailor `docs.md` tone, content, and section weight for the named audience.
3. Generate diagrams that support the profile's goals (e.g., a deployment diagram for `operations_handover` may make more sense than a class diagram).
4. Stay within the page-count budget — each profile creates a complete story within ~30 pages excluding images.

## Profile ↔ page_suffix consistency check (Phase 1)

The Confluence page title formula is `{repo_name}{page_suffix}`. For consistency across runs, the suggested `page_suffix` for each profile is shown above. Phase 1 should warn the user if `analysis_profile` and `confluence.page_suffix` look inconsistent (e.g., `architecture_review` profile with a developer-flavored suffix). It's not a hard error — the user may have a reason — but flag it.
