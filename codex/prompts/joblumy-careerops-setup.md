---
description: Configure JobLumy API key for career-ops syncing
---

Set up the JobLumy × career-ops integration. $ARGUMENTS

Read and follow the instructions in `~/.codex/skills/joblumy-careerops/setup.md`.

That skill file walks through:
1. Prompting the user for their JobLumy API key (from joblumy.com/integrations)
2. Verifying the key works against `/api/applications`
3. Writing `JOBLUMY_API_KEY` and `JOBLUMY_API_URL` to the user's shell config
4. Reporting next steps to the user

If `~/.codex/skills/joblumy-careerops/setup.md` is missing, tell the user:
"The joblumy-careerops skill files aren't installed. See the plugin README
at https://github.com/rafsuntaskin/careerops-joblumy-plugin for setup."
