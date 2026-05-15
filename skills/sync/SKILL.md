---
description: Sync a career-ops evaluation to JobLumy — ships full report + career-ops fit score + tailored resume
---

## Prerequisites

Verify env vars `JOBLUMY_API_KEY` and `JOBLUMY_API_URL` are set in the
session environment (written there by `/joblumy-careerops:setup`).

If either is missing — tell the user: "JobLumy is not configured. Run
`/joblumy-careerops:setup` first." Stop.

Verify this is a career-ops project — `reports/` directory must exist.
If not, tell the user: "No `reports/` directory found. Run this from
your career-ops project." Stop.

## career-ops is the source of truth

career-ops already produces an archetype, role summary, fit score,
seniority, remote policy, and a tailored CV PDF. **Do not regenerate
any of this.** Read what's in the report, ship it to JobLumy verbatim.
JobLumy honours `metadataJson.pluginImported: true` on `/api/jobs/import`
and skips its own AI extraction for these jobs.

## Triggers

Run when the user:

- Asks to sync a specific report or "all unsynced" reports to JobLumy
- Confirms they applied to a job and wants to update its status
- Recovers from a 409 (already-tracked) error from the auto-sync hook

## Finding the target report

**Default — most recently modified:** `ls -t reports/*.md | head -1`

**By company or role:** grep filenames and the first-line heading
of each report.

**Bulk sync:** list all `reports/*.md` whose contents do NOT contain
`**JobLumy ID:**`.

## Parsing the report

Read the file and pull the following fields. Field patterns tolerate
English, Spanish (`Evaluación`, `Arquetipo`), and French (`Évaluation`)
plus `--` / `—` / `–` separators in headings.

| Field | Source pattern |
|-------|---------------|
| Company / Role | First line `# Evaluation: {Company} -- {Role}` |
| URL | `**URL:**` |
| Date | `**Date:**` |
| Score | `**Score:**` (parse `4.2/5` → `{ value: 4.2, max: 5 }`) |
| Archetype | `**Archetype:**` (also `**Arquetipo:**`) |
| Legitimacy | `**Legitimacy:**` |
| PDF | `**PDF:**` |
| Existing JobLumy ID | `**JobLumy ID:**` |
| Section A | Under heading `## A) Role Summary`, rows `\| **{Key}** \| {Value} \|` — capture: Archetype, Domain, Function, Seniority, Remote, Team size, TL;DR |

**Deriving `remoteOk` from Section A Remote:**

- "Full remote", "Fully remote", "Remote worldwide", "Work from anywhere" → `true`
- "Onsite only", "In-office only", "No remote" → `false`
- Hybrid, partial, regional, anything else → omit (leave null)

**Resolving the PDF path from `**PDF:**`:**

- File path (contains `/` or ends in `.pdf`) — use it directly
- `✅` — derive `output/{report-stem}.pdf` (e.g. `reports/001-acme-2026-05-10.md` → `output/001-acme-2026-05-10.pdf`)
- `❌`, `pending`, missing — no resume to upload

**If `**URL:**` is missing:** skip and note "no URL — track manually at joblumy.com".

**If `**JobLumy ID:**` is already set with a real ID:**

- **View existing:** call `mcp__joblumy__get_application` with the stored ID, show the dashboard link, done.
- **Re-sync:** proceed with the flow below but use the stored ID for the PATCH; skip the `POST /api/applications` step (the application already exists).

## Sync flow

For each report:

### Step 1 — Import the job

`POST $JOBLUMY_API_URL/api/jobs/import` with the parsed body. Use Bash:

```bash
curl -sS -X POST "$JOBLUMY_API_URL/api/jobs/import" \
  -H "Authorization: Bearer $JOBLUMY_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "<job-url>",
    "title": "<role>",
    "company": "<company>",
    "descriptionText": "<TL;DR>",
    "remoteOk": <true | false>,
    "externalSource": {
      "source": "career-ops",
      "summary": "<TL;DR>",
      "archetype": "<archetype>",
      "seniority": "<seniority>",
      "domain": "<domain>",
      "function": "<function>",
      "legitimacy": "<legitimacy tier>",
      "reportRef": "reports/<filename>"
    }
  }'
```

Omit any optional field whose value is empty. Response: `{ jobPostId, created }`.

### Step 2 — Create the application with the career-ops score

```bash
curl -sS -X POST "$JOBLUMY_API_URL/api/applications" \
  -H "Authorization: Bearer $JOBLUMY_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "<job-url>",
    "stage": "SAVED",
    "externalScore": {
      "source": "career-ops",
      "value": <score value>,
      "max": <score max>,
      "label": "<legitimacy tier>",
      "reportRef": "reports/<filename>"
    }
  }'
```

Response shape: `{ application: { id, jobPost: { id }, ... } }`. Capture `application.id` and `application.jobPost.id`.

### Step 3 — Upload the tailored CV (if present)

If the PDF resolves to a real file:

```bash
curl -sS -X POST "$JOBLUMY_API_URL/api/resumes/upload" \
  -H "Authorization: Bearer $JOBLUMY_API_KEY" \
  -F "file=@<pdf-path>"
```

Response: `{ resume: { id, ... } }`. Then link it to the application:

```bash
curl -sS -X PATCH "$JOBLUMY_API_URL/api/applications/<applicationId>" \
  -H "Authorization: Bearer $JOBLUMY_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"resumeId":"<resumeId>"}'
```

### Step 4 — Write `**JobLumy ID:** {applicationId}` back into the report

Insert immediately after `**PDF:**` (or after `**Legitimacy:**` if PDF
line is absent, or at end-of-file as last resort). Same logic as the
auto-sync hook — keeps the report idempotent on re-runs.

### Step 5 — Show result

```
✓ {Company} — {Role}
  Score: {value}/{max} ({legitimacy})
  Resume: {pdf-path}      (omit this line if no PDF)
  → View: {API_URL}/jobs/{jobPostId}
```

## Marking as Applied

When the user says they applied (or asks to update status to Applied):

1. Read the report. If `**JobLumy ID:**` is missing, run the sync flow above first.
2. PATCH the application:
   ```bash
   curl -sS -X PATCH "$JOBLUMY_API_URL/api/applications/<applicationId>" \
     -H "Authorization: Bearer $JOBLUMY_API_KEY" \
     -H "Content-Type: application/json" \
     -d '{"stage":"APPLIED","appliedDate":"<today ISO 8601>"}'
   ```
3. Confirm: `✓ Marked as Applied → {API_URL}/jobs/{jobPostId}`

## Bulk sync output

```
✓  Acme Corp — Senior Engineer     score 4.2/5  resume ✓  → joblumy.com/jobs/abc123
✓  Beta Inc — Backend Engineer     score 3.7/5             → joblumy.com/jobs/def456
⚠  Widget Inc — PM                 no URL, skipped
✗  Initech — Backend Lead          duplicate (409)
```

Final summary:

```
Synced {n}/{total}  →  View pipeline: {API_URL}/applications
```

## Error handling

| Status | Cause | Action |
|--------|-------|--------|
| 401 / 403 | Bad or expired API key | Stop. "Your JobLumy API key was rejected. Run `/joblumy-careerops:setup` to update it." |
| 404 | Application ID not found on PATCH | Re-run the sync flow to obtain a fresh ID. |
| 409 | Application already tracked | Not a failure — call `mcp__joblumy__search_jobs` with the job title to find the existing entry's ID, then continue with `mcp__joblumy__add_note` or the PDF upload using that ID. |
| 422 | Invalid input | Show the API's error detail verbatim. For bad URLs, ask the user to confirm. |
| 429 | Rate limited | Wait 5s, retry once. If still 429, stop and tell user to retry in a minute. |
| 5xx | Server error | Retry once. If still failing, stop. "JobLumy is having server issues — try again shortly." |
| Network / timeout | No connection | Stop. "Could not reach JobLumy. Check your internet connection." |

**Single sync:** stop on first error.

**Bulk sync:** log `✗ {entry}  error: {reason}` and continue with the next.

**PDF upload failure:** non-fatal — log it and continue. The application is already created; the user can attach a resume later in JobLumy.

**`update_application` rejecting stage transition:** If the API rejects
a transition because required fields are missing (e.g. `firstResponseDate`
for SCREENING), show the missing fields and ask the user before
retrying once.

## Checking the auto-sync log

If the user reports a report wasn't auto-synced, read the hook's log:

```bash
tail -50 ~/.claude/joblumy-careerops.log
```

> Node.js powers both this skill (via curl) and the auto-sync script
> (`~/.claude/joblumy-careerops.mjs`). No additional dependencies beyond
> what career-ops already requires.
