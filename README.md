# JobLumy × career-ops Integration Plugin

Sync your [career-ops](https://github.com/santifer/career-ops) job evaluations to [JobLumy](https://joblumy.com) for visual pipeline tracking.

## What it does

- **Auto-sync**: Every time career-ops saves an evaluation report, it automatically appears in your JobLumy pipeline (Saved stage)
- **Score sync**: Run `/joblumy-careerops:sync` to enrich the JobLumy entry with career-ops score, archetype, and legitimacy tier
- **Apply tracking**: When you apply to a job, update the JobLumy stage to Applied in one command
- **Visual UI**: Every sync returns a direct link to the job in the JobLumy dashboard

## Install

### Claude Code

```bash
claude plugin install github:rafsuntaskin/careerops-joblumy-plugin
```

### OpenCode

OpenCode doesn't have a plugin installer. Copy the command files into your career-ops project:

```bash
cp career-ops-plugin/opencode/commands/*.md .opencode/commands/
```

The skills are shared with Claude Code — if you've already installed the plugin via Claude Code, OpenCode will find them automatically.

## Get your JobLumy API key

Before running setup, you'll need to connect this plugin to your JobLumy account:

1. Go to **[joblumy.com/integrations](https://joblumy.com/integrations)**
2. Look for "API Keys" or "Developer" section
3. Create a new API key (copy the full key — you won't see it again)
4. Keep this key handy for the setup step below

> **Note:** You need a [JobLumy](https://joblumy.com) account. Sign up at joblumy.com if you don't have one yet.

## First-time setup

After installing the plugin and getting your API key, run:

```
/joblumy-careerops:setup
```

The setup command will:

1. **Prompt for your API key** — Paste the key you copied from joblumy.com/integrations
2. **Verify the connection** — Tests that your key is valid by connecting to JobLumy
3. **Register the MCP server** — Adds JobLumy API tools to Claude Code (you'll see this in your Tools sidebar)
4. **Install auto-sync hooks** — Sets up three background watchers:
   - When you save a career-ops evaluation report → creates a **Saved** entry in JobLumy
   - When career-ops generates a tailored PDF → uploads it as your resume for that job
   - When you update your career-ops tracker → syncs your application stage (Applied, Interview, etc.)
5. **Write configuration** — Saves your API key and URL to `~/.claude/settings.json`

**After setup completes, restart Claude Code** to activate the MCP server and hooks. You'll see a notification when your next career-ops report auto-syncs to JobLumy.

## Usage

| Command | When to use |
|---------|-------------|
| `/joblumy-careerops:setup` | First-time setup or re-configure API key |
| `/joblumy-careerops:sync` | After an evaluation — adds score details + returns dashboard link |
| `/joblumy-careerops:sync` (with "I applied") | After applying — updates stage to Applied in JobLumy |

## Troubleshooting connection issues

**"API key was rejected" during setup**
- Double-check you copied the full key from joblumy.com/integrations (no extra spaces)
- Make sure your JobLumy account is active and not suspended
- Run `/joblumy-careerops:setup` again to re-enter your API key

**Auto-sync isn't working**
- Check that Claude Code restarted after setup
- Run `/joblumy-careerops:setup` to verify your API key is still valid
- Check the auto-sync log: `tail -50 ~/.claude/joblumy-careerops.log`
- Make sure you're saving reports to the `reports/` directory in career-ops

**"JobLumy is not configured" error**
- This means the setup command didn't complete. Run `/joblumy-careerops:setup` again
- Verify that `JOBLUMY_API_KEY` appears in your environment: `echo $JOBLUMY_API_KEY`

## How auto-sync works

The setup installs a background PostToolUse hook in `~/.claude/settings.json`. Whenever career-ops writes a report to `reports/*.md`, the hook:

1. Reads the report and extracts the job URL
2. Creates a Saved entry in your JobLumy tracker
3. Writes the `**JobLumy ID:**` back into the report for future reference

The hook runs silently in the background — no interruption to your career-ops workflow. Run `/joblumy-careerops:sync` to enrich the entry with career-ops score details.

## Requirements

- [career-ops](https://github.com/santifer/career-ops) installed and configured
- [JobLumy](https://joblumy.com) account with an API key
- `curl` available in PATH (standard on macOS/Linux)
