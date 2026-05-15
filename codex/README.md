# Codex integration

Codex doesn't have a plugin installer like Claude Code, so installation is a two-step copy.

## Install

From the plugin root:

```bash
# 1. Copy the skill files (the actual sync logic)
mkdir -p ~/.codex/skills/joblumy-careerops
cp codex/skills/setup.md  ~/.codex/skills/joblumy-careerops/setup.md
cp skills/sync/SKILL.md   ~/.codex/skills/joblumy-careerops/sync.md

# 2. Copy the slash-command prompts
mkdir -p ~/.codex/prompts
cp codex/prompts/*.md ~/.codex/prompts/
```

> The Codex setup skill (`codex/skills/setup.md`) is different from the
> Claude Code one — it only writes env vars to your shell config. No
> MCP registration, no hooks. The sync skill is identical for both.

Then in Codex:

```
/joblumy-careerops-setup
```

Follow the prompts — Codex will ask for your JobLumy API key (get it from
[joblumy.com/integrations](https://joblumy.com/integrations)) and write
the env vars to your shell config.

## Commands

| Command | When to use |
|---------|-------------|
| `/joblumy-careerops-setup` | First-time setup or to re-configure your API key |
| `/joblumy-careerops-sync`  | After a career-ops evaluation — ships score + resume to JobLumy |
| `/joblumy-careerops-sync` (with "I applied") | After applying — updates the JobLumy stage to Applied |

## What's different from the Claude Code version

| Feature | Claude Code | Codex |
|---------|-------------|-------|
| Install | `claude plugin install …` | manual `cp` (above) |
| Slash commands | ✅ | ✅ (via `~/.codex/prompts/`) |
| MCP server | ✅ (registered by setup) | Skipped — Codex sync uses curl, not MCP, so JobLumy's AI extraction stays out of the loop |
| Auto-sync on report write | ✅ (PostToolUse hook) | ❌ — run `/joblumy-careerops-sync` manually after each evaluation |
| Auto-sync of tailored CVs | ✅ (PostToolUse hook) | ❌ — handled by `/joblumy-careerops-sync` |
| Auto-sync of tracker statuses | ✅ (PostToolUse hook) | ❌ — handled by `/joblumy-careerops-sync` |

If you want auto-sync on Codex, you can run the watcher daemon from
`skills/setup/SKILL.md` standalone via launchd/systemd — see the Claude
Code setup skill for the script source.

## Why curl, not MCP?

career-ops already produces a complete evaluation: role summary, archetype,
fit score, seniority, remote policy, and a tailored CV. JobLumy's MCP
tools (`mcp__joblumy__*`) route through its standard application flow,
which kicks off JobLumy's own AI extraction — duplicating work career-ops
already did and risking divergent metadata.

The integration endpoints `/api/jobs/import` and `/api/applications`
accept `externalSource` and `externalScore` payloads with a
`pluginImported: true` flag that tells JobLumy to **skip its AI** and
store the career-ops data verbatim. The sync skill uses these endpoints
directly via curl.
