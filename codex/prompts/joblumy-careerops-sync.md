---
description: Sync a career-ops evaluation to JobLumy — ships full report + score + tailored resume
---

Sync career-ops evaluation to JobLumy. $ARGUMENTS

Read and follow the instructions in `~/.codex/skills/joblumy-careerops/sync.md`.

That skill file covers:
- Finding the target report (most recent, by company/role, or bulk-unsynced)
- Parsing the report header and Section A table
- POSTing to `/api/jobs/import` with `externalSource` (bypasses JobLumy AI extraction)
- POSTing to `/api/applications` with `externalScore` (career-ops fit score)
- Uploading the tailored CV to `/api/resumes/upload` and linking it
- Writing `**JobLumy ID:**` back into the report for idempotency
- Marking applications as Applied when the user has applied
- Handling errors (401, 409, 422, 429, 5xx)

If `~/.codex/skills/joblumy-careerops/sync.md` is missing, tell the user:
"The joblumy-careerops skill files aren't installed. Run /joblumy-careerops-setup
first, or see https://github.com/rafsuntaskin/careerops-joblumy-plugin"
