---
description: Configure the JobLumy API key + URL for the career-ops Codex integration
---

## What this skill does

Connects this Codex environment to a JobLumy account so the
`/joblumy-careerops-sync` skill can ship career-ops evaluations into
the user's JobLumy pipeline.

Codex has no plugin installer and no hook system, so setup is much
smaller than the Claude Code equivalent — just two env vars in the
user's shell config.

## Prerequisites

The user needs a JobLumy account and an API key from
[joblumy.com/integrations](https://joblumy.com/integrations).

If the user hasn't got one, tell them to grab a key first and stop.

## Steps

### 1 — Prompt for the API key

Ask the user to paste their JobLumy API key. Trim whitespace.

If the key is empty or obviously malformed (e.g. shorter than 20 chars),
tell them and stop — don't write garbage to their shell config.

### 2 — Verify the key

Set a default base URL if the user hasn't overridden it:

```
JOBLUMY_API_URL="${JOBLUMY_API_URL:-https://joblumy.com}"
```

Then probe a cheap authenticated endpoint:

```bash
curl -sS -o /dev/null -w "%{http_code}" \
  -H "Authorization: Bearer <key>" \
  "$JOBLUMY_API_URL/api/applications?limit=1"
```

- `200` — key works, continue.
- `401` / `403` — key rejected. Tell the user and stop.
- Anything else — show the status code and ask the user whether to
  continue anyway (network blip vs. real failure).

### 3 — Persist the env vars

Detect the user's shell from `$SHELL`:

- `*/zsh`  → write to `~/.zshrc`
- `*/bash` → write to `~/.bashrc` (macOS: also check `~/.bash_profile`)
- `*/fish` → write to `~/.config/fish/config.fish` (use `set -gx`)

Append two lines (idempotent — if a previous `JOBLUMY_API_KEY` export
exists, replace it rather than duplicating):

```sh
export JOBLUMY_API_KEY="…"
export JOBLUMY_API_URL="https://joblumy.com"
```

For fish, use `set -gx JOBLUMY_API_KEY "…"` syntax instead.

**Do not** print the full API key back to the user — show only the
first 6 and last 4 characters.

### 4 — Report

```
✓ JobLumy connected
  API URL: https://joblumy.com
  Key:     jb_xxxx…abcd
  Saved to: ~/.zshrc

Next steps:
  1. Restart your shell (or `source ~/.zshrc`) so Codex picks up the env vars.
  2. From your career-ops project, run /joblumy-careerops-sync after an evaluation.
```

## What this skill intentionally does NOT do (vs Claude Code)

- **No MCP server registration.** career-ops → JobLumy sync uses curl
  against the integration endpoints (`/api/jobs/import`,
  `/api/applications`) with `externalSource` / `externalScore` payloads.
  Those bypass JobLumy's AI extraction — exactly what we want, since
  career-ops already produced the evaluation.
- **No background hooks.** Codex doesn't have a `PostToolUse`-equivalent.
  Auto-sync isn't available; the user runs `/joblumy-careerops-sync`
  manually after each career-ops evaluation. Users who want auto-sync
  can install the watcher daemon from the Claude Code setup skill
  standalone (launchd/systemd).
