---
description: Connect career-ops to JobLumy — sets up MCP server, API key, and auto-sync hook
---

# Setup: career-ops × JobLumy

Trigger when the user asks to connect career-ops to JobLumy, set up the integration, or run first-time configuration.

## Flow

### Step 1 — API key

Check if `JOBLUMY_API_KEY` is already set in the environment.

If set: confirm with the user that they want to use it. If not, or if they say no:
- Ask: "Paste your JobLumy API key (`sk-jl-...`). You can create one at https://joblumy.com/integrations"

Ask if they are using a custom API URL (e.g. localhost for local dev). Default: `https://joblumy.com`.

Store as:
- `API_KEY` = the key value
- `API_URL` = the base URL (no trailing slash)

### Step 2 — Write the auto-sync script

Write the following Node.js script to `~/.claude/joblumy-careerops.mjs`:

```js
#!/usr/bin/env node
// JobLumy career-ops auto-sync. Fires on PostToolUse Write hooks for
// any reports/*.md file. Parses career-ops's structured evaluation
// (header + Section A) and ships it to JobLumy as a pre-extracted job
// + tracked application. JobLumy honours metadataJson.pluginImported
// and skips its own AI extraction for these jobs — career-ops is the
// source of truth.

import fs from "fs";
import path from "path";
import { homedir } from "os";

const LOG_FILE = path.join(homedir(), ".claude", "joblumy-careerops.log");

function log(level, msg) {
  const ts = new Date().toISOString();
  try {
    fs.appendFileSync(LOG_FILE, `${ts} [${level}] ${msg}\n`);
  } catch {
    /* ignore */
  }
}

function exitAuthError(msg) {
  log("AUTH", msg);
  process.stderr.write(`JobLumy auto-sync: ${msg}\n`);
  process.exit(2); // asyncRewake surfaces this to the user
}

function exitSilent(msg) {
  if (msg) log("SKIP", msg);
  process.exit(0);
}

async function postJson(url, apiKey, body) {
  const res = await fetch(url, {
    method: "POST",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "Content-Type": "application/json",
    },
    body: JSON.stringify(body),
    signal: AbortSignal.timeout(15000),
  });
  return { status: res.status, data: await res.json().catch(() => ({})) };
}

// --- Report parsing ---

function pickHeader(content, label) {
  const re = new RegExp(`\\*\\*${label}:\\*\\*\\s*([^\\n]+)`, "i");
  const m = content.match(re);
  return m ? m[1].trim() : null;
}

// First-line heading: `# Evaluation: {Company} -- {Role}`
// Tolerates "Evaluación" / "Évaluation" and -- / — / – separators.
function parseHeading(content) {
  const m = content.match(
    /^#\s+(?:Evaluation|Evaluaci[oó]n|[ÉE]valuation)\s*:\s*(.+?)\s*(?:--|—|–)\s*(.+)$/im,
  );
  if (!m) return { company: null, role: null };
  return { company: m[1].trim(), role: m[2].trim() };
}

function parseScore(raw) {
  if (!raw) return null;
  const m = raw.match(/^([\d.]+)\s*\/\s*(\d+)/);
  if (m) return { value: Number(m[1]), max: Number(m[2]) };
  const single = Number(raw);
  return Number.isFinite(single) ? { value: single, max: 5 } : null;
}

// Block A is a key/value markdown table immediately under the
// `## A) Role Summary` (or `## Bloque A —`) heading.
function sectionABlock(content) {
  // The /m flag makes $ match end-of-line — we want true end-of-string,
  // so anchor explicitly with (?:^|\n) and drop the m flag.
  const m = content.match(
    /(?:^|\n)##\s+(?:A\)|Bloque\s+A|Block\s+A|Bloc\s+A)[^\n]*\n([\s\S]+?)(?=\n##\s|\n---\s*\n|$)/i,
  );
  if (!m) return {};
  const out = {};
  for (const line of m[1].split("\n")) {
    const r = line.match(/^\|\s*\*\*([^*|]+)\*\*\s*\|\s*([^|]+?)\s*\|/);
    if (r) out[r[1].trim().toLowerCase()] = r[2].trim();
  }
  return out;
}

function deriveRemote(remoteStr) {
  if (!remoteStr) return undefined;
  const s = remoteStr.toLowerCase();
  if (/full\s+remote|fully\s+remote|remote\s+(?:any|worldwide|global)|work\s+from\s+anywhere/.test(s)) return true;
  if (/on[-\s]?site\s+only|in[-\s]?office\s+only|no\s+remote/.test(s)) return false;
  return undefined; // ambiguous (hybrid, partial, regional) — leave null
}

function trimTo(s, max) {
  if (!s) return s;
  return s.length > max ? s.slice(0, max) : s;
}

// --- Main ---

let raw = "";
for await (const chunk of process.stdin) raw += chunk;
if (!raw.trim()) process.exit(0);

let hookData;
try {
  hookData = JSON.parse(raw);
} catch {
  exitSilent("stdin parse error");
}

const toolInput = hookData.tool_input ?? hookData.input ?? {};
const filePath = toolInput.file_path ?? toolInput.path ?? hookData.file_path ?? "";

if (!filePath.endsWith(".md") || !filePath.replace(/\\/g, "/").includes("reports/")) {
  process.exit(0);
}

let content;
try {
  content = fs.readFileSync(filePath, "utf8");
} catch (e) {
  exitSilent(`could not read ${filePath}: ${e.message}`);
}

// Idempotent: skip if already synced
if (/\*\*JobLumy ID:\*\*\s+[a-zA-Z0-9_-]{8,}/.test(content)) exitSilent();

const apiKey = process.env.JOBLUMY_API_KEY ?? "";
const apiUrl = (process.env.JOBLUMY_API_URL ?? "https://joblumy.com").replace(/\/$/, "");
if (!apiKey) exitAuthError("JOBLUMY_API_KEY is not set. Run /joblumy-careerops:setup.");

const jobUrl = pickHeader(content, "URL");
if (!jobUrl) exitSilent(`no URL in ${filePath}`);

const heading = parseHeading(content);
const sectionA = sectionABlock(content);
const score = parseScore(pickHeader(content, "Score"));
const archetype =
  pickHeader(content, "Archetype") ??
  pickHeader(content, "Archétype") ??
  pickHeader(content, "Arquetipo") ??
  sectionA["archetype"] ??
  sectionA["arquetipo"];
const legitimacy = pickHeader(content, "Legitimacy") ?? pickHeader(content, "Legitimidad");
const tldr = sectionA["tl;dr"] ?? sectionA["tldr"] ?? sectionA["resumen"];
const reportRef = `reports/${path.basename(filePath)}`;
const remoteOk = deriveRemote(sectionA["remote"]);
const seniority = trimTo(sectionA["seniority"] ?? sectionA["nivel"], 100);
const domain = trimTo(sectionA["domain"] ?? sectionA["dominio"], 200);
const fn = trimTo(sectionA["function"] ?? sectionA["función"] ?? sectionA["funcion"], 100);

if (!heading.company || !heading.role) {
  log("WARN", `could not parse heading "${filePath}" — falling back to placeholders`);
}

const importBody = {
  url: jobUrl,
  title: heading.role ?? "(career-ops evaluation)",
  company: heading.company ?? "(career-ops)",
  ...(tldr ? { descriptionText: tldr } : {}),
  ...(remoteOk !== undefined ? { remoteOk } : {}),
  externalSource: {
    source: "career-ops",
    ...(tldr ? { summary: tldr } : {}),
    ...(archetype ? { archetype: trimTo(archetype, 300) } : {}),
    ...(seniority ? { seniority } : {}),
    ...(domain ? { domain } : {}),
    ...(fn ? { function: fn } : {}),
    ...(legitimacy ? { legitimacy: trimTo(legitimacy, 100) } : {}),
    reportRef,
  },
};

// Step 1 — Import job (pluginImported: true; no AI re-extraction)
let importRes;
try {
  importRes = await postJson(`${apiUrl}/api/jobs/import`, apiKey, importBody);
} catch (e) {
  log("ERROR", `import request failed for ${jobUrl}: ${e.message}`);
  process.exit(0);
}
if (importRes.status === 401 || importRes.status === 403) {
  exitAuthError(`API key rejected (HTTP ${importRes.status}). Run /joblumy-careerops:setup.`);
}
if (importRes.status !== 200 && importRes.status !== 201) {
  log(
    "ERROR",
    `/api/jobs/import HTTP ${importRes.status} for ${jobUrl}: ${JSON.stringify(importRes.data).slice(0, 200)}`,
  );
  process.exit(0);
}

// Step 2 — Create application with externalScore
const appBody = {
  url: jobUrl,
  stage: "SAVED",
  ...(score
    ? {
        externalScore: {
          source: "career-ops",
          value: score.value,
          max: score.max,
          ...(legitimacy ? { label: trimTo(legitimacy, 200) } : {}),
          reportRef,
        },
      }
    : {}),
};

let appRes;
try {
  appRes = await postJson(`${apiUrl}/api/applications`, apiKey, appBody);
} catch (e) {
  log("ERROR", `application request failed for ${jobUrl}: ${e.message}`);
  process.exit(0);
}
if (appRes.status === 401 || appRes.status === 403) {
  exitAuthError(`API key rejected (HTTP ${appRes.status}). Run /joblumy-careerops:setup.`);
}

let appId = "";
let jobPostId = "";
if (appRes.status === 409) {
  // Already tracked — sync skill can recover by looking up the existing entry.
  log("INFO", `already tracked: ${jobUrl} (run /joblumy-careerops:sync to recover ID)`);
  process.exit(0);
} else if (appRes.status === 422) {
  log("ERROR", `invalid input (HTTP 422) for ${jobUrl}: ${JSON.stringify(appRes.data).slice(0, 200)}`);
  process.exit(0);
} else if (appRes.status >= 500) {
  log("ERROR", `server error HTTP ${appRes.status} for ${jobUrl}: ${JSON.stringify(appRes.data).slice(0, 200)}`);
  process.exit(0);
} else if (appRes.status !== 200 && appRes.status !== 201) {
  log("ERROR", `unexpected HTTP ${appRes.status} for ${jobUrl}: ${JSON.stringify(appRes.data).slice(0, 200)}`);
  process.exit(0);
} else {
  appId = appRes.data?.application?.id ?? appRes.data?.id ?? "";
  jobPostId = appRes.data?.application?.jobPost?.id ?? appRes.data?.jobPost?.id ?? "";
}

if (!appId) {
  log("ERROR", `no applicationId in response for ${jobUrl}: ${JSON.stringify(appRes.data).slice(0, 200)}`);
  process.exit(0);
}

// Step 3 — Write JobLumy ID back into the report
try {
  let updated;
  if (content.includes("**PDF:**")) {
    updated = content.replace(/(\*\*PDF:\*\*[^\n]*\n)/, `$1**JobLumy ID:** ${appId}\n`);
  } else if (content.includes("**Legitimacy:**")) {
    updated = content.replace(/(\*\*Legitimacy:\*\*[^\n]*\n)/, `$1**JobLumy ID:** ${appId}\n`);
  } else {
    updated = content + `\n**JobLumy ID:** ${appId}\n`;
  }
  fs.writeFileSync(filePath, updated, "utf8");
} catch (e) {
  log("ERROR", `could not update report ${filePath}: ${e.message}`);
}

const dashboard = jobPostId ? `${apiUrl}/jobs/${jobPostId}` : `${apiUrl}/applications`;
const scoreNote = score ? ` (score ${score.value}/${score.max})` : "";
log("INFO", `synced ${path.basename(filePath)} → ${appId}${scoreNote}`);
console.log(`→ JobLumy: ${dashboard}`);
```

### Step 2b — Write the resume-upload script

Write the following Node.js script to `~/.claude/joblumy-careerops-resume.mjs`:

```js
#!/usr/bin/env node
// JobLumy career-ops resume-upload hook. Fires on PostToolUse Write
// hooks for output/*.pdf. Locates the matching reports/{stem}.md by
// filename stem, reads its **JobLumy ID:** line, uploads the PDF to
// /api/resumes/upload, then links the new resumeId to the application
// via PATCH. Writes **JobLumy Resume:** {id} back into the report so
// re-runs are idempotent.

import fs from "fs";
import path from "path";
import { homedir } from "os";

const LOG_FILE = path.join(homedir(), ".claude", "joblumy-careerops.log");

function log(level, msg) {
  const ts = new Date().toISOString();
  try {
    fs.appendFileSync(LOG_FILE, `${ts} [${level}] ${msg}\n`);
  } catch {
    /* ignore */
  }
}

function exitAuthError(msg) {
  log("AUTH", msg);
  process.stderr.write(`JobLumy resume sync: ${msg}\n`);
  process.exit(2);
}

function exitSilent(msg) {
  if (msg) log("SKIP", msg);
  process.exit(0);
}

// --- Main ---

let raw = "";
for await (const chunk of process.stdin) raw += chunk;
if (!raw.trim()) process.exit(0);

let hookData;
try {
  hookData = JSON.parse(raw);
} catch {
  exitSilent("stdin parse error");
}

const toolInput = hookData.tool_input ?? hookData.input ?? {};
const filePath = toolInput.file_path ?? toolInput.path ?? hookData.file_path ?? "";

// Only act on output/*.pdf writes
const norm = filePath.replace(/\\/g, "/");
if (!norm.endsWith(".pdf") || !norm.includes("output/")) process.exit(0);

if (!fs.existsSync(filePath)) exitSilent(`pdf not on disk: ${filePath}`);

// Find the matching report by filename stem
const stem = path.basename(filePath, ".pdf");
const projectRoot = norm.split("/output/")[0];
const reportPath = path.join(projectRoot, "reports", `${stem}.md`);
if (!fs.existsSync(reportPath)) exitSilent(`no report at ${reportPath}`);

const report = fs.readFileSync(reportPath, "utf8");

const idMatch = report.match(/\*\*JobLumy ID:\*\*\s+([a-zA-Z0-9_-]{8,})/);
if (!idMatch) exitSilent(`report ${stem}.md has no JobLumy ID yet — run /joblumy-careerops:sync`);
const applicationId = idMatch[1];

// Skip if resume already linked
if (/\*\*JobLumy Resume:\*\*\s+[a-zA-Z0-9_-]{8,}/.test(report)) exitSilent();

const apiKey = process.env.JOBLUMY_API_KEY ?? "";
const apiUrl = (process.env.JOBLUMY_API_URL ?? "https://joblumy.com").replace(/\/$/, "");
if (!apiKey) exitAuthError("JOBLUMY_API_KEY is not set. Run /joblumy-careerops:setup.");

// Step 1 — Upload PDF
let uploadRes;
try {
  const buf = fs.readFileSync(filePath);
  const blob = new Blob([buf], { type: "application/pdf" });
  const form = new FormData();
  form.append("file", blob, path.basename(filePath));
  const res = await fetch(`${apiUrl}/api/resumes/upload`, {
    method: "POST",
    headers: { Authorization: `Bearer ${apiKey}` },
    body: form,
    signal: AbortSignal.timeout(30000),
  });
  uploadRes = { status: res.status, data: await res.json().catch(() => ({})) };
} catch (e) {
  log("ERROR", `upload failed for ${filePath}: ${e.message}`);
  process.exit(0);
}

if (uploadRes.status === 401 || uploadRes.status === 403) {
  exitAuthError(`API key rejected (HTTP ${uploadRes.status}). Run /joblumy-careerops:setup.`);
}
if (uploadRes.status !== 200 && uploadRes.status !== 201) {
  log(
    "ERROR",
    `upload HTTP ${uploadRes.status} for ${filePath}: ${JSON.stringify(uploadRes.data).slice(0, 200)}`,
  );
  process.exit(0);
}

const resumeId = uploadRes.data?.resume?.id ?? uploadRes.data?.id ?? "";
if (!resumeId) {
  log("ERROR", `no resumeId in upload response: ${JSON.stringify(uploadRes.data).slice(0, 200)}`);
  process.exit(0);
}

// Step 2 — Link resume to application
let patchRes;
try {
  const res = await fetch(`${apiUrl}/api/applications/${applicationId}`, {
    method: "PATCH",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "Content-Type": "application/json",
    },
    body: JSON.stringify({ resumeId }),
    signal: AbortSignal.timeout(15000),
  });
  patchRes = { status: res.status, data: await res.json().catch(() => ({})) };
} catch (e) {
  log("ERROR", `link request failed for ${applicationId}: ${e.message}`);
  process.exit(0);
}

if (patchRes.status !== 200) {
  log(
    "ERROR",
    `link HTTP ${patchRes.status} for ${applicationId}: ${JSON.stringify(patchRes.data).slice(0, 200)}`,
  );
  process.exit(0);
}

// Step 3 — Write **JobLumy Resume:** back into the report
try {
  let updated;
  if (report.includes("**JobLumy ID:**")) {
    updated = report.replace(
      /(\*\*JobLumy ID:\*\*[^\n]*\n)/,
      `$1**JobLumy Resume:** ${resumeId}\n`,
    );
  } else {
    updated = report + `\n**JobLumy Resume:** ${resumeId}\n`;
  }
  fs.writeFileSync(reportPath, updated, "utf8");
} catch (e) {
  log("ERROR", `could not update report ${reportPath}: ${e.message}`);
}

log("INFO", `linked resume ${path.basename(filePath)} → ${resumeId} (app ${applicationId})`);
console.log(`→ JobLumy resume linked: ${apiUrl}/applications`);
```

### Step 2c — Write the tracker-status script

Write the following Node.js script to `~/.claude/joblumy-careerops-tracker.mjs`:

```js
#!/usr/bin/env node
// JobLumy career-ops tracker-status hook. Fires on PostToolUse Write
// hooks for data/applications.md. Diffs current row statuses against
// the last-synced cache and PATCHes JobLumy applications whose state
// has changed.

import fs from "fs";
import path from "path";
import { homedir } from "os";

const LOG_FILE = path.join(homedir(), ".claude", "joblumy-careerops.log");
const CACHE_FILE = path.join(homedir(), ".claude", "joblumy-careerops-statuses.json");

function log(level, msg) {
  const ts = new Date().toISOString();
  try {
    fs.appendFileSync(LOG_FILE, `${ts} [${level}] ${msg}\n`);
  } catch {
    /* ignore */
  }
}

function exitAuthError(msg) {
  log("AUTH", msg);
  process.stderr.write(`JobLumy tracker sync: ${msg}\n`);
  process.exit(2);
}

function exitSilent(msg) {
  if (msg) log("SKIP", msg);
  process.exit(0);
}

// career-ops canonical state → JobLumy stage. SKIP intentionally absent
// — those entries should NOT propagate to the tracker. See
// templates/states.yml in career-ops for the source of truth.
const STAGE_MAP = {
  evaluated: "SAVED",
  applied: "APPLIED",
  responded: "SCREENING",
  interview: "INTERVIEW",
  offer: "OFFER",
  rejected: "REJECTED",
  discarded: "WITHDRAWN",
  // Spanish aliases from states.yml
  evaluada: "SAVED",
  aplicado: "APPLIED",
  aplicada: "APPLIED",
  enviada: "APPLIED",
  sent: "APPLIED",
  respondido: "SCREENING",
  entrevista: "INTERVIEW",
  oferta: "OFFER",
  rechazado: "REJECTED",
  rechazada: "REJECTED",
  descartado: "WITHDRAWN",
  descartada: "WITHDRAWN",
  cerrada: "WITHDRAWN",
  cancelada: "WITHDRAWN",
};

function mapStage(raw) {
  if (!raw) return null;
  const norm = raw.replace(/\*+/g, "").trim().toLowerCase();
  return STAGE_MAP[norm] ?? null;
}

// --- Main ---

let raw = "";
for await (const chunk of process.stdin) raw += chunk;
if (!raw.trim()) process.exit(0);

let hookData;
try {
  hookData = JSON.parse(raw);
} catch {
  exitSilent("stdin parse error");
}

const toolInput = hookData.tool_input ?? hookData.input ?? {};
const filePath = toolInput.file_path ?? toolInput.path ?? hookData.file_path ?? "";
const normPath = filePath.replace(/\\/g, "/");

if (!normPath.endsWith("/data/applications.md")) process.exit(0);

let content;
try {
  content = fs.readFileSync(filePath, "utf8");
} catch (e) {
  exitSilent(`could not read ${filePath}: ${e.message}`);
}

const apiKey = process.env.JOBLUMY_API_KEY ?? "";
const apiUrl = (process.env.JOBLUMY_API_URL ?? "https://joblumy.com").replace(/\/$/, "");
if (!apiKey) exitAuthError("JOBLUMY_API_KEY is not set. Run /joblumy-careerops:setup.");

// Parse the table. Expected header:
// | # | Date | Company | Role | Score | Status | PDF | Report | Notes |
// Spanish: | # | Fecha | Empresa | Rol | Score | Estado | PDF | Report | ... |
const rows = [];
for (const line of content.split("\n")) {
  if (!line.startsWith("|")) continue;
  if (/^\|[\s\-:|]+\|/.test(line)) continue;
  const cells = line.split("|").slice(1, -1).map((c) => c.trim());
  if (cells.length < 8) continue;
  const num = cells[0];
  if (!num || Number.isNaN(Number(num))) continue;
  const status = cells[5];
  const report = cells[7];
  const reportMatch = report.match(/\(([^)]+\.md)\)/);
  if (!reportMatch) continue;
  rows.push({ num, status, reportPath: reportMatch[1] });
}

if (rows.length === 0) exitSilent("no parseable rows");

// Bootstrap detection — first run has no cache. Seed the cache without
// syncing so we don't blast PATCHes for every legacy row.
const isBootstrap = !fs.existsSync(CACHE_FILE);

let cache = {};
if (!isBootstrap) {
  try {
    cache = JSON.parse(fs.readFileSync(CACHE_FILE, "utf8"));
  } catch {
    cache = {};
  }
}

const projectRoot = path.dirname(path.dirname(filePath)); // .../<project>/data/applications.md → .../<project>
const updates = [];
const newCache = { ...cache };

for (const row of rows) {
  const stage = mapStage(row.status);
  if (!stage) continue; // SKIP or unknown state — don't touch
  const reportFull = path.join(projectRoot, row.reportPath);
  let reportContent;
  try {
    reportContent = fs.readFileSync(reportFull, "utf8");
  } catch {
    continue;
  }
  const idMatch = reportContent.match(/\*\*JobLumy ID:\*\*\s+([a-zA-Z0-9_-]{8,})/);
  if (!idMatch) continue;
  const appId = idMatch[1];

  newCache[appId] = stage;
  if (cache[appId] === stage) continue;
  updates.push({ appId, stage, num: row.num });
}

if (isBootstrap) {
  try {
    fs.writeFileSync(CACHE_FILE, JSON.stringify(newCache, null, 2));
  } catch (e) {
    log("ERROR", `could not write bootstrap cache: ${e.message}`);
  }
  log("INFO", `bootstrap: cached ${Object.keys(newCache).length} statuses (no sync on first run)`);
  process.exit(0);
}

if (updates.length === 0) {
  try {
    fs.writeFileSync(CACHE_FILE, JSON.stringify(newCache, null, 2));
  } catch {
    /* ignore */
  }
  exitSilent("no status changes");
}

const now = new Date().toISOString();
let okCount = 0;
let failCount = 0;

for (const u of updates) {
  const body = { stage: u.stage };
  if (u.stage === "APPLIED") body.appliedDate = now;
  let res;
  try {
    const r = await fetch(`${apiUrl}/api/applications/${u.appId}`, {
      method: "PATCH",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
      },
      body: JSON.stringify(body),
      signal: AbortSignal.timeout(15000),
    });
    res = { status: r.status, data: await r.json().catch(() => ({})) };
  } catch (e) {
    log("ERROR", `tracker PATCH failed for ${u.appId}: ${e.message}`);
    failCount++;
    continue;
  }
  if (res.status === 401 || res.status === 403) {
    exitAuthError(`API key rejected (HTTP ${res.status}). Run /joblumy-careerops:setup.`);
  }
  if (res.status === 200) {
    okCount++;
    log("INFO", `tracker: row ${u.num} → ${u.stage} (${u.appId})`);
  } else {
    failCount++;
    // Rewind cache for this app so the next write retries it
    delete newCache[u.appId];
    if (cache[u.appId]) newCache[u.appId] = cache[u.appId];
    log(
      "ERROR",
      `tracker PATCH HTTP ${res.status} for ${u.appId}: ${JSON.stringify(res.data).slice(0, 200)}`,
    );
  }
}

try {
  fs.writeFileSync(CACHE_FILE, JSON.stringify(newCache, null, 2));
} catch (e) {
  log("ERROR", `could not write cache: ${e.message}`);
}

const total = okCount + failCount;
console.log(`→ JobLumy tracker: ${okCount}/${total} updates synced`);
```

### Step 3 — Update ~/.claude/settings.json

Read `~/.claude/settings.json`. Merge the following keys, preserving all existing content:

```json
{
  "env": {
    "JOBLUMY_API_KEY": "<API_KEY>",
    "JOBLUMY_API_URL": "<API_URL>"
  },
  "mcpServers": {
    "joblumy": {
      "type": "http",
      "url": "<API_URL>/api/mcp",
      "headers": {
        "Authorization": "Bearer <API_KEY>"
      }
    }
  },
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write",
        "hooks": [
          {
            "type": "command",
            "command": "node ~/.claude/joblumy-careerops.mjs",
            "asyncRewake": true
          },
          {
            "type": "command",
            "command": "node ~/.claude/joblumy-careerops-resume.mjs",
            "asyncRewake": true
          },
          {
            "type": "command",
            "command": "node ~/.claude/joblumy-careerops-tracker.mjs",
            "asyncRewake": true
          }
        ]
      }
    ]
  }
}
```

Replace `<API_KEY>` and `<API_URL>` with actual values.

When merging `hooks.PostToolUse`:
- If the `Write` matcher entry already exists, append the three new hook commands to its `hooks` array instead of duplicating the matcher
- For each command, skip if an entry with that exact `command` already exists. The three commands are:
  - `node ~/.claude/joblumy-careerops.mjs` (report sync)
  - `node ~/.claude/joblumy-careerops-resume.mjs` (PDF / resume sync)
  - `node ~/.claude/joblumy-careerops-tracker.mjs` (tracker status sync)

### Step 4 — Confirm

Tell the user:

```
✓ Setup complete

Auto-sync is now active. Every time career-ops saves a report to reports/, 
it will automatically appear in your JobLumy pipeline.

→ Dashboard: <API_URL>/applications

Restart Claude Code to activate the MCP server and sync hook.

Run /joblumy-careerops:sync after any evaluation to also add career-ops 
score details and get a direct link to that job in JobLumy.
```

## Error Handling

**settings.json read/write failure:**
1. Show the error clearly.
2. Tell the user to manually add the env vars and hook to `~/.claude/settings.json`.
3. Provide the exact JSON snippet to copy.

**Auto-sync hook errors (surfaced via asyncRewake):**
The hook exits with code 2 only for auth errors (missing or rejected API key), which wakes Claude with a message. When that happens, tell the user:
> "JobLumy auto-sync failed: [error message]. Run `/joblumy-careerops:setup` to update your API key."

**Viewing the sync log:**
All hook activity is logged to `~/.claude/joblumy-careerops.log`. If the user reports missing syncs or unexpected behaviour, read that file to diagnose:
```bash
tail -50 ~/.claude/joblumy-careerops.log
```
