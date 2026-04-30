# MCP code-understanding cache handling

The `code-understanding` MCP server caches cloned repositories on disk for fast subsequent analysis. The cache has a configurable size limit (default 20 in our setup); when the limit is exceeded, the server purges older repositories.

## The trap this creates

`state.json` may say `"cloned": true` for a repo, but the MCP server has since purged it from cache. If a sub-agent trusts state.json and calls `mcp__code-understanding__refresh_repo` on a purged repo, it fails.

## The fix: always verify before deciding clone vs refresh

For every repo, the sub-agent should:

1. Call `mcp__code-understanding__get_repo_structure` first.
2. **If the response indicates the repo is not in cache:** the cache purged it. Call `mcp__code-understanding__clone_repo` regardless of what `state.json` says about `cloned: true`.
3. **If the response confirms the repo is in cache:** call `mcp__code-understanding__refresh_repo` to pick up any new commits.
4. **Only after a successful `get_repo_structure` confirms cache presence** should you choose `refresh_repo` over `clone_repo`.

## Why this order matters

`get_repo_structure` is the cheapest way to probe cache state — it doesn't transfer the repo, just asks the server "do you have this?" Calling `refresh_repo` blindly when the repo isn't cached produces a confusing error. Calling `clone_repo` blindly when it IS cached wastes time re-cloning. Probe first, then decide.
