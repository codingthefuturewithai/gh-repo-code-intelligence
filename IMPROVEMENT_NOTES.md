# Improvement Notes

Living log of issues found while dry-running `gh-repo-code-intelligence` as if it were a brand-new user. Each entry: symptom → evidence → root cause → proposed fix. Append new findings; do not edit historical entries (they're a record of what users will hit).

---

## Run 1 — 2026-04-29 — Fresh-clone dry run, target `codingthefuturewithai/rag-memory`, `architecture_review` profile, Confluence to space `DEMO` / parent `RAG Memory Docs`

End state: `success: false` in `state.json`, no diagrams generated, no Confluence upload, but a 1357-line `docs.md` was produced via fallback path.

### Issue 1: Newly-registered MCP servers default to **disabled** in the session, but `claude mcp list` reports them as **Connected**

**Symptom.** After `claude mcp add-json -s user code-understanding ...` (and the same for `mermaid_image_generator` and `Conduit`), the user restarted Claude Code and verified with `claude mcp list` — all three showed `✓ Connected`. They proceeded to run the analysis. Inside the spawned sub-agent, those three servers were `disabled` (per the sub-agent's startup banner in `output.json`), so none of their tools could be called.

**Evidence.**
- `output.json` `system/init` event lists: `disabled: code-understanding`, `disabled: Conduit`, `disabled: mermaid_image_generator` — alongside ~20 other servers correctly marked `connected` or `needs-auth`.
- Screenshot of `claude mcp list` (post-restart) shows all three as `✓ Connected`.
- Screenshot of `/mcp` panel inside the same session shows all three as `○ disabled`.

**Root cause.** Two independent pieces of state, both sometimes called "connected":
- `claude mcp list` performs a reachability/process-launch check. "Connected" there means "the server can be spawned and responds."
- `/mcp` reflects a per-user **enable/disable** flag persisted in `~/.claude.json`. New servers added via `claude mcp add-json` are registered but **not automatically enabled** — the user must visit `/mcp` once and enable each new server. Sub-agents spawned via `claude -p` inherit the persisted enable state, so a disabled server's tools are unreachable in the sub-agent.

**Proposed fixes.**
- **`onboard-new-user` skill, Phase 3:** Add an explicit `/mcp` step. After the user restarts Claude Code, instruct them to: open `/mcp` inside the session, find each of the three newly-added servers, and Enable any that show `○ disabled`. Re-running `claude mcp list` is **not** sufficient verification.
- **`onboard-new-user` skill, Phase 2 detection table:** Update the existing table to be sharper about this — "Tools available to you in session" is the only authoritative signal; `claude mcp list` is a secondary cross-reference and can mislead.
- **`CLAUDE.md`, Startup Verification section:** Strengthen the "test MCP tool availability" instruction. The current language ("attempt a simple operation") is too soft. Should be: "If `mcp__code-understanding__*` tools are not in your tool registry, halt the run and write a clear failure to `state.json`. Do not proceed under any circumstances."

### Issue 2: Sub-agent silently fell back to `git clone` + filesystem reads, violating CLAUDE.md's no-fallback rule

**Symptom.** With `mcp__code-understanding__*` unavailable, the sub-agent ran `git clone https://github.com/codingthefuturewithai/rag-memory.git /tmp/rag-memory-clone/` and proceeded to read files directly. It produced a 1357-line `docs.md` with ASCII text "diagrams" embedded in markdown. From the user's perspective, this looked like a partial success — there's a docs file — but it bypassed the entire purpose of the tool.

**Evidence.**
- `output.json` tool tally for the sub-agent: 5× Bash, 3× Read, 1× Write, 1× ToolSearch. **Zero** `mcp__code-understanding__*` calls. **Zero** `mcp__mermaid_image_generator__*` calls.
- `state.json` errors array: `"mcp__code-understanding and mcp__mermaid_image_generator tools not available in this session. Used git clone + file system tools for analysis. Diagrams skipped (no mermaid generator)."`
- `docs.md` exists but contains no PNG references, only inline pseudo-diagrams.

**Root cause.** `CLAUDE.md` explicitly says (Tool Selection Requirements §1): "Use ONLY the MCP code-understanding tools... Do NOT fall back to web fetching or other non-MCP methods if these tools fail. If a repository clone fails, mark it as failed in state.json and move on." But the sub-agent prompt that spawns from CLAUDE.md doesn't *enforce* this — it gives the rule and then describes the analysis steps. The model interpreted the fallback to `git clone` as "doing its best given the constraint," not as a violation.

**Proposed fixes.**
- **`CLAUDE.md` Critical Rules block:** Promote the no-fallback rule into the top-of-file Critical Rule callout (currently only the "spawn sub-agents" rule is there). Phrase as a hard prohibition with a named consequence: "If MCP tools are unavailable, you MUST set `success: false`, write a clear error, and exit. You MUST NOT use `git clone`, `curl`, file-reading, or any other non-MCP path to substitute for `mcp__code-understanding__*` tools. This is non-negotiable. Producing fallback documentation hides the real failure from the user."
- **Sub-agent spawn prompt template (Phase 2 of CLAUDE.md):** Add an explicit pre-flight at the top of the sub-agent's prompt: "Before starting, verify your tool registry contains `mcp__code-understanding__get_repo_structure`. If it does not, write the appropriate error to `state.json`, set `success: false`, and exit immediately. Do not attempt any analysis."

### Issue 3: Confluence "parent page not found" message in state.json was hallucinated

**Symptom.** `state.json` recorded: `"Parent page 'RAG Memory Docs' not found in Confluence space DEMO (site alias: ctf). Page must be created manually before re-running analysis."` This implied the page check was actually performed and the page genuinely doesn't exist.

**Evidence.**
- `output.json` shows zero `mcp__Conduit__*` tool calls (Conduit was also disabled). The sub-agent could not have queried Confluence.
- The error message reads as if it was the result of a real lookup, but no lookup occurred.

**Root cause.** When the Conduit MCP was unavailable, the sub-agent reasoned "we can't reach Confluence, therefore the page is not found" and wrote a misleading error. This conflates **tool unavailable** with **legitimate not-found**. A user reading the message would naturally try to create or rename the parent page, when the actual problem is upstream.

**Proposed fixes.**
- **`CLAUDE.md` Confluence Upload Workflow §6 (Error Handling):** Distinguish two error classes explicitly:
  - "MCP Conduit tool unavailable" → record verbatim, do not infer about the parent page.
  - "Confluence query returned not-found" → only when an actual API call returned no results.
- **Sub-agent reasoning rule:** If a tool was never called, the agent must not write claims about results from that tool. Add as a general principle in CLAUDE.md error-handling section.

### Issue 4: `claude mcp list` is given undue authority by the onboarding flow

**Symptom.** Phase 3 of the `onboard-new-user` skill reads: "Once you're back, run `claude mcp list` and paste the output. I need to see all the servers we just registered show up there before we go further." This frames `claude mcp list` as the verification step. The skill's own Phase 2 detection table acknowledges that registered-but-disabled is possible — but Phase 3 doesn't repeat the warning, and the skill never instructs the user to actually open `/mcp` and confirm enabled status.

**Evidence.** The flow we just executed went exactly as the skill prescribes, and we ended up running with three disabled servers. The bug is in the skill, not in execution.

**Proposed fix.** Rewrite Phase 3 to require *both* checks: `claude mcp list` for reachability, then `/mcp` for enable status, with a screenshot-or-paste confirmation that none of the three new servers show `○ disabled`.

### Issue 5: No state-reset / clean-slate utility

**Symptom.** After a failed run, the user wants to reset to a fresh-clone state and try again. There's no documented or scripted way to do this — they have to manually delete `state.json`, `output.json`, and `output/` and remember which other files might have been created.

**Proposed fix.** Optional. Either:
- Add a `RESET.md` or `make reset` / `npm run reset` target that clears the runtime state but leaves config.
- Or document the reset procedure in `README.md` so future users (and AI agents like me) don't have to figure it out by inference.

---

## Cross-cutting themes from Run 1

- **Soft instructions get treated as suggestions.** "Do NOT fall back" appears in CLAUDE.md but the model fell back anyway. Hard prohibitions need to be paired with explicit failure paths and "if X then halt" framing, not stated as a preference.
- **State that *looks* like progress can mask total failure.** The user saw a `docs.md` and `success: false` and had to dig to learn that nothing of value was produced. State.json should distinguish "partial success with caveats" from "real success."
- **Verification steps that ask the wrong question are worse than none.** Running `claude mcp list` and seeing Connected gave false confidence. A verification step should test the actual capability, not a proxy for it.
