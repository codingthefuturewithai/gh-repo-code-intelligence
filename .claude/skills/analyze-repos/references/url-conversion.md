# SSH → HTTPS GitHub URL conversion

In Phase 1, before processing any repo, normalize every `repos[].github_url` to HTTPS. SSH URLs prompt for an SSH passphrase (or fail outright in unattended sessions) and break the pipeline.

## Detect

A URL is SSH if it starts with `git@github.com:` (or any `git@<host>:` form).

## Convert

Replace `git@github.com:` with `https://github.com/`. Keep the `org/repo-name` part unchanged. The trailing `.git` can be left in or stripped — both work.

## Examples

| Before | After |
|---|---|
| `git@github.com:acme/payments-service.git` | `https://github.com/acme/payments-service.git` |
| `git@github.com:acme/payments-service` | `https://github.com/acme/payments-service` |
| `https://github.com/acme/payments-service` | (already HTTPS — no change) |

## Behavior

- Convert silently in memory or in a copy of config.
- Tell the user once that the conversion happened (so they can update `config.json` themselves at their leisure).
- Do NOT modify the user's `config.json` on disk without their explicit permission.
