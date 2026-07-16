# Business Control Center — Build Guide

**Purpose:** This document is a complete, self-contained instruction set. Attach it to a fresh Claude (Cowork) chat and Claude will reproduce the **Business Control Center** — a branded dashboard artifact that runs a business's skills, saves each run to Google Drive, and lets the owner browse past runs — exactly as built.

**How to use this document (Claude, read this first):**
1. Work in **Cowork mode**. You will build a persistent **artifact** (via `create_artifact`), create **scheduled tasks** (one per skill), and use the **Google Drive** connector as the data bus.
2. **Do Section 1 (the interview) FIRST.** Collect every piece of business-specific information from the user before building anything. Use the `AskUserQuestion` tool for multiple-choice items and normal conversation for free-form items (logo, company description).
3. Then follow Section 4 (Build Steps) in order.
4. Read Section 3 (Platform Constraints) carefully — it is the set of hard-won rules that make this work inside the Cowork sandbox. Violating them is the #1 cause of failure.
5. Appendix A is the **full reference artifact HTML**. Appendix B is the **scheduled-task prompt templates**. Appendix C shows **example result fragments**. Reproduce from these, customizing only the marked parts.

---

## Section 1 — Collect this from the user first (the interview)

Do not build until you have all of this.

**Ask ONE question at a time.** Ask a single question, wait for the answer, then ask the next. Do not paste this list at the user, do not batch several questions into one message, and never ask something they've already told you. Use the `AskUserQuestion` tool with **one question per call**, offering sensible options where you can so they can click rather than type. Keep each question short and in plain language — the person is a business owner, not a developer. Work through the groups below in order (1.1 → 1.5), then confirm the whole picture back to them once, at Step A, before building.

### 1.1 Brand & identity
- **Company / dashboard name** (e.g. "arcAId") — used in the header and the Drive folder name.
- **One-line tagline** for the header subtitle (e.g. "Business Control Center").
- **Logo** — ask the user to attach it. **Prefer an SVG** (it inlines cleanly and scales); a PNG works too (you'll embed it as a `data:` URI). If they have neither, offer to render a simple gradient monogram.
- **Brand colors** — either (a) the user gives you hex values, or (b) you derive a palette from the logo. You need, at minimum: a primary/heading color, an accent color, a background tint, a border tint, body-text color, and muted-text color. (The reference uses a indigo→lilac scheme; see Section 5 for the exact variable slots to fill.)
- **Font preference** — the reference uses a serif for headings (Georgia fallback) and system sans for body. Confirm or adjust. NOTE: web fonts do **not** load in the sandbox; use system/serif stacks only.

> **The user's branding is the source of truth for the entire build — not the values in this document.** Everything shipped here (the indigo/lilac palette, the arcAId logo, the header name) is a **placeholder**. The moment the user gives you their brand, replace all of it and build only in their brand.
>
> The trap: the reference hexes appear **literally inside the task prompts (Appendix B) and the example fragments (Appendix C)**, because tasks generate results in a fresh session and *cannot* read the artifact's CSS variables. If you only re-theme the artifact's `:root`, the shell will be on-brand while **every result card still renders in the reference brand**. Re-theming the shell and forgetting the task prompts is the single easiest mistake to make in this build. When you set the palette, propagate the user's hexes into every task prompt in the same pass.

### 1.2 Company context (so skill outputs are accurate)
- **What the business does** (industry, product/service, who the customers are). This gives skills like a morning brief or content strategy the right context.
- **Location / timezone** (for calendar/hour analytics and local-event lookups).
- Any **names/entities** that should be recognized (team, key clients) — optional.

### 1.3 Skills / modules & their layout
For each skill the user wants on the board, collect:
- **Skill name** (display) and a **short one-line description**.
- **Which section** it belongs to (sections are the top-level nav groups, e.g. Home / Finance / Marketing — ask the user for their section names).
- **Underlying skill** it runs (e.g. `morning-briefing`, `bank-statement-analyzer`, `instagram-export-dashboard`, or a custom skill).
- **Does it need a file upload?** If yes: what file type (PDF, ZIP, CSV…) and a short button label (e.g. "PDF", "ZIP").
- **Which connectors** the skill needs (see 1.4).
- **What the result should look like** (KPIs, tables, charts). Charts are allowed but must be **static inline SVG** — see Section 3.

The reference build ships three skills as a model:
| Skill | Section | Underlying skill | Upload | Connectors |
|---|---|---|---|---|
| Morning Routine | Home | `morning-briefing` | none | Gmail, Google Calendar |
| Bank Statement Analysis | Finance | `bank-statement-analyzer` | PDF | (parses the uploaded file) |
| Instagram Dashboard | Marketing | `instagram-export-dashboard` | ZIP | (parses the uploaded export) |

### 1.4 Connectors (authorization) — REQUIRED
Tell the user which connectors must be authorized in their Claude connector settings **before** the dashboard can work, and confirm each is connected:
- **Google Drive — with WRITE access (create/edit files).** This is mandatory: Drive is where every result is stored and read from. A read-only Drive connection will fail on the first save with a "requires additional permissions" error — have the user reconnect with edit access.
- **Per-skill connectors**, e.g. **Gmail + Google Calendar** for a morning brief; **QuickBooks/PayPal/Stripe** for finance skills, etc. Only require what the chosen skills actually use.
- Note: connectors are authorized by the user in claude.ai connector settings (or `/mcp` in an interactive session). You cannot authorize them for the user.

### 1.5 Where to build
- Confirm the Google Drive account and that it's OK to create a top-level folder named after the company's control center (e.g. "Business Control Center").

---

## Section 2 — What you're building (architecture)

- A single **persistent artifact** (`create_artifact`) = the dashboard UI. It has a top bar (logo + name), a left nav of **sections**, and per-section **skill cards** plus **result panels**.
- Each **skill** maps to a **scheduled task** (created with the scheduled-tasks tool). Clicking **Run** on a card calls `window.cowork.runScheduledTask(taskId)` — the artifact cannot talk to the chat, so a scheduled task is the engine.
- **Google Drive is the data bus.** There is one **folder per skill** under a parent "Business Control Center" folder. When a skill runs, its task writes the result as a **new HTML file** in that skill's folder. The artifact **lists** the folder (newest first) and **downloads + renders** the selected file. This gives **run history**: each panel has a dropdown of past runs.
- **File inputs** (e.g. a bank PDF) are uploaded from the dashboard straight to Drive via `callMcpTool(create_file)`; large files are **chunked**. The task downloads the input, parses it, and writes the result.

Data flow per run:
```
[Run button] → runScheduledTask → task: (optionally read input from Drive) → run skill →
render result as static HTML → create_file into the skill's Drive folder (new timestamped file) →
[artifact] lists folder → downloads newest → renders it in the panel (+ adds it to the run dropdown)
```

---

## Section 3 — Platform constraints (the rules that make this work)

These are non-obvious and essential. The reference implementation already respects all of them.

1. **The sidebar artifact cannot send a message to the chat.** Its runtime exposes only `window.cowork.callMcpTool(name,args)`, `window.cowork.askClaude(prompt,data)`, `window.cowork.runScheduledTask(taskId)`, and `localStorage`. There is **no `sendPrompt`**. Therefore **Run buttons trigger scheduled tasks**, and results come back **through Drive**, not the chat.
2. **Injected result HTML does not run `<script>`.** The artifact renders results by setting `innerHTML`, which never executes scripts. So **every chart in a result must be static inline SVG** (bars = `<rect>`, donuts = `<circle>` with `stroke-dasharray`). Do not rely on Chart.js or any JS in results. Wrapping in an iframe does **not** help — the sandbox CSP blocks it too.
3. **Sandbox CSP blocks external resources.** No external fonts, scripts, images, or stylesheets (only a tiny CDN allowlist for Chart.js/Grid.js/Mermaid, which you won't need). **Inline all CSS**, use system/serif font stacks, and embed any images as `data:` URIs / inline SVG. Design for light mode.
4. **Store results as real `.html` files, not Google Docs.** When you `create_file` with `contentMimeType:"text/html"`, set **`disableConversionToGoogleType:true`** so Drive keeps it as a raw HTML file. If you let it become a Google Doc, reading it back via `read_file_content` **mangles the content** (escapes markdown punctuation, drops tags). Read results back with **`download_file_content`** (returns exact bytes as base64) and decode.
5. **`read_file_content` cannot extract uploaded PDFs/ZIPs.** For uploaded binary inputs, the task must `download_file_content` (raw bytes) and parse them itself (`pdftotext`/`pdfplumber` for PDFs, `unzip`/`zipfile` for ZIPs). Drive's text extraction returns empty for uploaded PDFs.
6. **Large results must be gzipped.** A big dashboard (>~20 KB) is unreliable to transfer as one payload. The task **gzips** the HTML (`gzip -9`), base64s the `.gz`, and `create_file`s it as `text/html`. The artifact detects the **gzip magic bytes (0x1f 0x8b)** and decompresses with `DecompressionStream("gzip")`. Small results can be stored as plain HTML; the artifact handles both.
7. **Large uploads must be chunked.** The in-browser upload bridge rejects payloads of a few MB. The dashboard slices files into ~300 KB parts and uploads each as a separate titled Drive file (`… :: U<id> :: 01/07`); the task reassembles the bytes in order and unzips. Small files upload as one.
8. **Google Drive connector needs WRITE access** and has **no delete/move** tools available — you can create, copy, search, read, and download only. Plan around it (you can't clean up old files programmatically; the folder-per-skill structure keeps things tidy going forward).
9. **Scheduled tasks run in a fresh session.** They can call connectors and `create_file`, but the **first run pauses for tool approvals** — tell the user to click **Run now** once in the Scheduled sidebar to approve. Don't assume the task can read the local working folder.
10. **Reads are cached.** After a brand-new file lands in Drive, the panel may show the previous one until the artifact's **header Reload** button is pressed (it busts the cache). The in-card ↻ can still serve a cached list.
11. **`localStorage` migration.** The artifact persists its skill config to `localStorage`. When you add new fields to the seed config, backfill them onto saved state on load, or older saved skills won't pick up new fields (folder IDs, upload flags, etc.). The reference has a `migrate()` step for this.
12. **MCP tool names are connection-specific.** The reference hardcodes Drive tool names like `mcp__a2f86dc4-…__search_files`. In the new environment the Google Drive server has a **different id**. You MUST discover the actual Drive tool names in this environment and substitute them into (a) the artifact's `DRIVE_*` constants and (b) the `create_artifact` `mcp_tools` list. The tools you need: `search_files`, `create_file`, `download_file_content`, `copy_file` (and the folder mimetype trick for creating folders).

---

## Section 4 — Build steps (do these in order)

### Step A — Interview & confirm
Complete Section 1 — **one question at a time**, waiting for each answer before asking the next. Then confirm the whole picture back in a single summary: the sections, the list of skills (with sections, descriptions, uploads, connectors), the brand palette, the logo, and that Drive (write) + per-skill connectors are authorized. From this point on, **the user's brand — never the reference values in this document — is what you build with**, in the artifact *and* in every task prompt.

### Step B — Create the Drive folder structure
1. Create a folder named after the control center (e.g. **"Business Control Center"**) at Drive root: `create_file` with `contentMimeType:"application/vnd.google-apps.folder"` and no content. Record its `id`.
2. For **each skill**, create a subfolder inside it (`parentId` = the parent folder id, same folder mimetype). Use a clean slug for the title (e.g. `morning-routine`). **Record every subfolder id** — you'll wire these into the artifact and the tasks.

### Step C — Build the artifact
1. Start from **Appendix A** (the full reference HTML). Write it to a file, then customize only these parts:
   - **`:root` CSS variables** → the user's brand palette (Section 5 lists each slot). Also the `--grad` gradient. Replace **every** reference hex — leave none behind.
   - **The header logo** → replace the inline `<svg>` in `.brand .mark` with the user's logo SVG (or a `data:` URI `<img>` for a PNG). Keep it ~40–44px.
   - **The header name + `<title>`** → the user's company/dashboard name.
   - **`<script id="cc-results" type="application/json">`** → **leave it exactly `{}`**. This is the embed channel for results; anything placed here loads on open and looks like a real run. The board ships empty (see Step E).
   - **`SECTIONS`** object → the user's section keys, titles, eyebrows, and ledes.
   - **`SEED`** array → one entry per skill: `{ id, section, name, desc, tag, taskId, driveFolderId, fav, needsFile?, accept?, uploadLabel? }`. `driveFolderId` = the subfolder id from Step B. `taskId` = the scheduled-task id you'll create in Step D.
   - **`TASK_MAP`** → `{ skillId: taskId }` fallback for older saved state.
   - **`DRIVE_SEARCH / DRIVE_CREATE / DRIVE_DOWNLOAD`** constants → the actual Google Drive MCP tool names in this environment (see constraint #12).
2. **Do not change** the machinery: upload+chunking+spinner, the run-history dropdown, gzip decompression, folder listing/rendering, `localStorage` + `migrate()`. It's all in Appendix A.
3. Validate the JS parses (e.g. wrap the main `<script>` body in `new Function(...)` in a quick check) before publishing.
4. `create_artifact` with the HTML file and `mcp_tools: [<drive search_files>, <drive create_file>, <drive download_file_content>]`.

### Step D — Create one scheduled task per skill
For each skill, create a scheduled task (scheduled-tasks tool) using the matching template in **Appendix B**, customizing:
- the underlying skill to run and how to build the result,
- the **result palette** — rewrite **every** reference hex in the Appendix B prompt with the user's hexes. The task runs in a fresh session and cannot read the artifact's CSS variables, so any hex you leave behind ships in the user's result,
- the skill's **Drive folder id** (`parentId` for the output file),
- for upload skills: the input title to search for + chunk reassembly,
- **gzip** the output if it's large (Instagram-style dashboards); plain `text/html` if small (a brief).
Each run must write a **new** file (a timestamped human title) into the folder — never overwrite — so history accrues.

### Step E — Do not seed sample results (the board ships empty)
The board must load **empty**. Every panel shows its empty state — *"No runs yet — click ▶ Run to generate one."* — until the user runs a skill and a real result lands in Drive. Concretely:

- Leave `<script id="cc-results" type="application/json">{}</script>` exactly as-is. Never bake a sample, mock, or placeholder result into it.
- Do not paste example HTML into a panel, and do not `create_file` a fake run into a skill's Drive folder to "demonstrate" the layout. A mock run is indistinguishable from real output and will be mistaken for the user's own data.
- The fragments in **Appendix C are reference models for writing the task prompts only** — they are never loaded into the board.
- The first real run belongs to the user. To see it work, they click **▶ Run** (or "Run now" on the task, which also approves its tools on first use).

### Step F — Verify
- On a **fresh load, before any run**: every panel shows "No runs yet" and there is no sample content anywhere on the board.
- Run one skill. Reload the artifact (header Reload) — that panel now loads its newest run.
- Test an upload skill: pick a file → watch the spinner/part-progress → "Uploaded ✓" → Run → the new run appears and the run dropdown activates (dropdown shows once a folder has 2+ files).
- Confirm the brand palette and logo render correctly — in the shell **and** inside the first real result card (if the result is off-brand, you missed the hexes in the task prompt).

---

## Section 5 — Brand palette mapping

The artifact themes everything through CSS variables in `:root`. Map the user's brand to these slots (reference values shown; replace with the user's):

| Variable | Role | Reference value |
|---|---|---|
| `--cream` | page background tint | `#F7F6FE` |
| `--blush` | soft surface | `#EFEBFC` |
| `--blush-deep` | borders | `#DFD9F6` |
| `--rose` | accent (buttons, labels) | `#9B5FD6` |
| `--rose-soft` | light accent | `#C79BE6` |
| `--wine` | headings / primary | `#4F2FE3` |
| `--wine-soft` | muted text | `#6E63A0` |
| `--ink` | body text | `#1F1546` |
| `--brass` | small accent ticks | `#6D3BE6` |
| `--paper` | card background | `#FFFFFF` |
| `--ok` | success/positive | `#4f9d7c` |
| `--grad` | logo gradient (buttons, mark) | `linear-gradient(135deg,#5A34E6,#9B5FD6)` |

**Result fragments** (Appendix C) use the same hexes literally (they're generated by tasks, which have no access to CSS vars). When you set the palette, also feed those hexes into the task prompts so results match the shell. Keep **green for positive/pass and red for negative/fail** regardless of brand — they read as status.

The result fragments use scoped wrapper classes so their styles never leak: `.cc-morning`, `.cc-finance`, `.cc-ig`. Each is `<div class="cc-X"><style>/* rules all prefixed .cc-X */</style> … </div>` — no `<html>/<head>/<body>`, no `<script>`, no external resources.

---

## Section 6 — Chart recipes (static SVG, no JS)

**Bar chart (used for cadence/day/hour and cash-flow-by-month):** each bar is a flex column; the bar height is baked in as an inline `style="height:H%"` where `H = round(value/max*100)` (use a small min like 3% for nonzero, 0% for zero). Colors are inline. No script.

**Donut (expenses by category):** one `<circle r=93>` per segment inside `<g transform="rotate(-90 110 110)">`, `stroke-width=34`, circumference ≈ `2π·93 = 584.34`. For a segment with fraction `f`: `stroke-dasharray = "(f*584.34) (584.34 − f*584.34)"` and `stroke-dashoffset = -(cumulativeFractionBefore * 584.34)`. Legend is plain HTML with colored dots.

See Appendix C for complete, working examples of both.

---

## Section 7 — Adding a new skill to the board (the integration template)

There is **no auto-creation**. Nothing in this build scaffolds a skill for you — a skill only appears on the board when you add it by hand in the four coordinated places below. Follow this contract exactly and the new card behaves like the seeded ones: it triggers a background task and shows a browsable run history. The **Morning Routine** skill (pulls calendar, inbox, a priority, a lift, and a headline into one brief) is used as the worked example throughout, because it's the simplest shape — a pure Run button with no upload. Where a skill needs a file upload instead, the extra steps are called out inline.

### 7.1 The four places a skill lives

A skill is not one object; it's four pieces that must agree on the same IDs and titles:

1. **A `SEED` entry** in the artifact (the card).
2. **A `SECTIONS` entry + nav icon** (only if it needs a new section).
3. **A Drive subfolder** under `arcAId Business Control Center` (its run history).
4. **A scheduled task** (the engine that does the work and writes results to that folder).

The dashboard's `migrate()` already merges any new `SEED` skill onto existing saved boards, so returning users see the new card without resetting `localStorage`.

### 7.2 Step 1 — Add the `SEED` entry

In the artifact's `SEED` array, add one object. Every field is meaningful:

```js
{ id:"morning-routine",         // unique slug; also the localStorage key and TASK_MAP key
  section:"home",               // which nav section this card sits under
  name:"Morning Routine",       // card title
  desc:"Pulls your calendar, inbox, top priority, a lift for the day, and a headline or two into one brief.",
  tag:"morning-briefing",       // the skill name this card represents
  taskId:"morning-routine-run", // MUST equal the scheduled task's taskId
  driveTitle:"arcAId CC - morning-routine",
  driveFolderId:"<paste the Drive folder id from step 3>",
  fav:true }
```

Rules that keep it wired: `taskId` must be identical to the scheduled task's id, and `driveFolderId` must be the folder the task writes into. Morning Routine reads live Calendar/Gmail, so it has **no upload** — omit `needsFile` and the card is a pure Run button.

**If the skill needs a file upload instead**, add four more fields and the card grows an upload button:

```js
  needsFile:true,                         // shows the upload button
  accept:"application/pdf,.pdf",          // input file filter
  uploadLabel:"PDF",                      // button label
  driveInput:"arcAId CC - <skill>-input", // the exact title the upload is saved under
```

Also add the id to the `TASK_MAP` fallback (`"morning-routine":"morning-routine-run"`) so older saved boards still resolve the task.

### 7.3 Step 2 — Add a section (only if needed)

Morning Routine sits in the existing **Home** section, so nothing to do here. To add a *new* section, add a `SECTIONS` entry and a nav glyph:

```js
// SECTIONS
home: { eyebrow:"Start your day", title:"Home",
        lede:"Your daily kickoff. Run your morning routine and anything you want waiting for you each morning." }
// renderNav icons map
const icons = { home:"⌂", finance:"$", marketing:"◎" };
```

A section only renders once at least one skill points `section` at it.

### 7.4 Step 3 — Create the Drive folder

Create a subfolder named after the skill inside the `arcAId Business Control Center` folder (Drive `create_file` with `contentMimeType:"application/vnd.google-apps.folder"` and `parentId` set to the parent folder). Copy the returned folder `id` into the `SEED` entry's `driveFolderId`. This folder **is** the run history — every run is a separate file in it, and the dashboard lists them newest-first with no extra code.

### 7.5 Step 4 — The upload contract (only for skills that take a file)

Morning Routine has no upload, so **it skips this step**. For any skill with `needsFile:true`: when a file is chosen, the dashboard base64-uploads it to Drive under `driveInput`. Small files land as a single file titled exactly `driveInput`. Files larger than **300 KB are split into chunks** titled:

```
arcAId CC - <skill>-input :: U<timestamp> :: 01/NN
arcAId CC - <skill>-input :: U<timestamp> :: 02/NN
...
```

Such a task must handle **both** shapes (single file, or a chunk set grouped by the `U<timestamp>` token, sorted by the leading number and concatenated as raw bytes) — copy the reassembly logic from the Instagram task in Appendix B.3.

### 7.6 Step 5 — The scheduled-task contract (the engine)

Create a scheduled task whose `taskId` equals the `SEED` `taskId`. The prompt must be fully self-contained (each run is a fresh session). Morning Routine's shape (Appendix B.1):

1. **Gather** — use the `morning-briefing` skill to assemble today's brief from Calendar + Gmail (plus a web-searched quote / local event / AI headline). *(An upload skill inserts two steps ahead of this: find the newest `driveInput` file or chunk group via `search_files`, then `download_file_content` → base64-decode → reassemble the bytes. Never use `read_file_content` for binary input.)*
2. **Render ONE result fragment** — see 7.7.
3. **Save as a new run** — `create_file` into `driveFolderId` with `contentMimeType:"text/html"`, `disableConversionToGoogleType:true`, and a **dated, unique title** (e.g. `Jul 9 - 9:14am`). Never reuse a fixed title or overwrite older runs — each run is its own history entry. If a fragment is large (image-heavy or a full dashboard), gzip it (`gzip -9`) and pass the gzipped bytes as `base64Content`; the dashboard auto-detects the gzip magic bytes and decompresses. (Morning Routine's brief is small, so plain `textContent` HTML is fine.)
4. **Confirm and stop.** Never edit the artifact.

### 7.7 Step 6 — The result-HTML styling contract

The dashboard injects the task's HTML into a panel with `innerHTML`, so results must obey the same rules as every other fragment (Appendix C):

- **One wrapper, scoped styles.** Wrap everything in a single `<div class="cc-morning">` (name it `cc-<skill>`) with ONE `<style>` block whose every selector is prefixed by that class. No `<html>/<head>/<body>`.
- **No scripts, no external resources.** Injected `<script>` never runs and the sandbox CSP blocks external fonts/scripts/images. Charts must be **inline static SVG** (see Section 6). Any image must be an inline **`data:` URI**, never a remote URL.
- **Use the brand hexes literally.** Tasks have no access to the artifact's CSS variables, so paste the palette values from Section 5: page cream `#F7F6FE`, cards `#fff`, borders `#DFD9F6`, blush `#EFEBFC`, serif (Georgia) headings in indigo `#4F2FE3`, small uppercase mono labels in lilac `#9B5FD6`, body `#1F1546`, muted `#6E63A0`. Keep **green for positive/pass, red for negative/fail** regardless of brand.
- **Fluid width.** `width:100%; box-sizing:border-box` on the wrapper so it fills the panel.

Morning Routine's fragment follows this exactly: a `.cc-morning` wrapper, a title row with the date, then Top Priorities, Today's Schedule, Needs a Reply, a Quote-of-the-Day blockquote (blush `#EFEBFC` background, lilac `#9B5FD6` left border), Near You, and AI News — the full example is in Appendix C.1.

### 7.8 Checklist

The skill is correctly integrated when: (a) the card shows in its section (with an upload button only if `needsFile`); (b) pressing Run triggers `taskId`, which does the work and writes a dated `text/html` run into `driveFolderId`; (c) for upload skills, choosing a file writes `driveInput` (or its chunks) to Drive and the task reassembles it; (d) after the header **Reload**, the panel renders the run and the run-history dropdown lists it. First run only: approve the task's tools once via "Run now".


## Appendix A — Full reference artifact HTML

Reproduce this artifact, customizing only the parts called out in Step C (palette, logo, `SECTIONS`, `SEED`, `TASK_MAP`, and the Drive tool-name constants). Everything else — upload/chunking/spinner, run-history dropdown, gzip decompress, folder reading, migration — should stay as-is.

```html
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>arcAId · Business Control Center</title>
<style>
  :root{
    color-scheme: light;
    /* arcAId brand — indigo + lilac logo */
    --cream:#F7F6FE;        /* page background (lavender white) */
    --blush:#EFEBFC;        /* soft violet surface */
    --blush-deep:#DFD9F6;   /* borders */
    --rose:#9B5FD6;         /* lilac accent */
    --rose-soft:#C79BE6;
    --wine:#4F2FE3;         /* violet (headings / primary) */
    --wine-soft:#6E63A0;    /* muted violet-gray */
    --ink:#1F1546;          /* dark plum body text */
    --brass:#6D3BE6;        /* purple accent ticks */
    --paper:#FFFFFF;
    --ok:#4f9d7c;
    --grad:linear-gradient(135deg,#5A34E6,#9B5FD6);  /* the logo gradient */
    --display:'Fraunces','Georgia','Times New Roman',serif;
    --body:'Hanken Grotesk',system-ui,-apple-system,'Segoe UI',sans-serif;
    --mono:'IBM Plex Mono',ui-monospace,'SFMono-Regular',Menlo,monospace;
  }
  *{box-sizing:border-box;margin:0;padding:0}
  body{
    font-family:var(--body);
    color:var(--ink);
    background:
      radial-gradient(120% 80% at 8% 4%, rgba(203,58,157,.08), transparent 55%),
      radial-gradient(120% 120% at 100% 100%, rgba(124,58,237,.05), transparent 50%),
      var(--cream);
    min-height:100vh;
    -webkit-font-smoothing:antialiased;
    font-size:15px;
  }

  /* ---- top bar ---- */
  .topbar{
    display:flex;align-items:center;justify-content:space-between;
    padding:1.1rem clamp(1.2rem,3vw,2.4rem);
    border-bottom:1px solid var(--blush-deep);
    position:sticky;top:0;z-index:30;
    background:rgba(250,248,254,.85);backdrop-filter:blur(8px);
  }
  .brand{display:flex;align-items:center;gap:.85rem}
  .brand .mark{display:flex;align-items:center}
  .brand .mark svg{width:44px;height:44px;display:block}
  .brand .name{font-family:var(--display);font-weight:600;font-size:1.32rem;color:var(--wine);line-height:1}
  .brand .tag{font-family:var(--mono);font-size:.6rem;letter-spacing:.24em;text-transform:uppercase;color:var(--wine-soft);margin-top:.28rem}
  .add-top{
    font-family:var(--mono);font-size:.7rem;letter-spacing:.12em;text-transform:uppercase;
    background:var(--wine);color:var(--cream);border:none;border-radius:999px;
    padding:.6rem 1.1rem;cursor:pointer;transition:transform .15s,background .2s;
  }
  .add-top:hover{background:var(--rose)}
  .add-top:active{transform:scale(.95)}
  .topbar-right{display:flex;align-items:center;gap:.7rem}
  .bridge-pill{font-family:var(--mono);font-size:.58rem;letter-spacing:.1em;text-transform:uppercase;
    padding:.4rem .65rem;border-radius:999px;border:1px solid var(--blush-deep);color:var(--wine-soft);
    display:inline-flex;align-items:center;gap:.4rem;cursor:pointer;transition:border-color .2s,color .2s}
  .bridge-pill::before{content:"";width:6px;height:6px;border-radius:50%;background:currentColor;flex:none}
  .bridge-pill.on{color:var(--ok);border-color:#bfe0d0}
  .bridge-pill.off{color:var(--rose)}
  .bridge-pill.off:hover{border-color:var(--rose)}

  /* ---- shell ---- */
  .shell{display:grid;grid-template-columns:230px 1fr;gap:clamp(1rem,2.5vw,2rem);
    padding:clamp(1.2rem,2.5vw,2rem) clamp(1.2rem,3vw,2.4rem);max-width:1320px;margin:0 auto}

  /* ---- sidebar ---- */
  .side{display:flex;flex-direction:column;gap:1.6rem;position:sticky;top:88px;align-self:start}
  .side .label{font-family:var(--mono);font-size:.62rem;letter-spacing:.2em;text-transform:uppercase;color:var(--brass);margin-bottom:.7rem;padding-left:.2rem}
  .nav{display:flex;flex-direction:column;gap:.3rem}
  .nav button{
    display:flex;align-items:center;gap:.7rem;width:100%;text-align:left;
    background:transparent;border:1px solid transparent;border-radius:11px;
    padding:.7rem .8rem;cursor:pointer;font-family:var(--body);font-size:.98rem;color:var(--ink);
    transition:background .18s,border-color .18s,color .18s;
  }
  .nav button .ico{width:1.5rem;text-align:center;font-size:1rem}
  .nav button .cnt{margin-left:auto;font-family:var(--mono);font-size:.72rem;color:var(--wine-soft);
    background:var(--blush);border-radius:999px;padding:.05rem .5rem}
  .nav button:hover{background:var(--paper);border-color:var(--blush-deep)}
  .nav button.on{background:var(--paper);border-color:var(--rose);color:var(--wine);font-weight:600;box-shadow:0 10px 26px -20px rgba(124,58,237,.5)}
  .nav button.on .cnt{background:var(--blush-deep);color:var(--wine)}

  .fav-rail{display:flex;flex-direction:column;gap:.4rem}
  .fav-chip{display:flex;align-items:center;gap:.55rem;font-size:.9rem;color:var(--wine);
    background:var(--paper);border:1px solid var(--blush-deep);border-radius:10px;padding:.5rem .65rem;cursor:pointer;
    transition:border-color .18s,transform .15s}
  .fav-chip:hover{border-color:var(--rose);transform:translateX(2px)}
  .fav-chip .star{color:var(--brass)}
  .fav-empty{font-size:.82rem;color:var(--wine-soft);line-height:1.4;padding:.2rem}

  /* ---- main ---- */
  .main{min-width:0}
  .sec-head{margin-bottom:1.5rem}
  .eyebrow{font-family:var(--mono);font-size:.7rem;letter-spacing:.26em;text-transform:uppercase;color:var(--rose);
    display:flex;align-items:center;gap:.7rem;margin-bottom:.7rem}
  .eyebrow::before{content:"";width:34px;height:1px;background:var(--brass)}
  .sec-head h2{font-family:var(--display);font-weight:600;font-size:clamp(1.9rem,4vw,2.7rem);color:var(--wine);line-height:1.02;letter-spacing:-.015em}
  .sec-head .lede{font-size:1.05rem;line-height:1.5;color:var(--ink);max-width:54ch;margin-top:.6rem}

  /* ---- cards ---- */
  .grid{display:grid;grid-template-columns:repeat(auto-fill,minmax(290px,1fr));gap:1.1rem}
  .card{
    background:var(--paper);border:1px solid var(--blush-deep);border-radius:16px;
    padding:1.3rem 1.3rem 1.1rem;position:relative;overflow:hidden;display:flex;flex-direction:column;gap:.55rem;
    transition:box-shadow .2s,transform .2s,border-color .2s;
  }
  .card::before{content:"";position:absolute;top:0;left:0;width:100%;height:3px;background:var(--rose)}
  .card:hover{box-shadow:0 20px 44px -28px rgba(124,58,237,.5);transform:translateY(-2px)}
  .card.is-fav{border-color:var(--rose)}
  .card-top{display:flex;align-items:flex-start;gap:.8rem}
  .mono-badge{width:42px;height:42px;border-radius:12px;flex:none;display:grid;place-items:center;
    font-family:var(--display);font-weight:600;font-size:1.25rem;color:#fff;
    background:var(--grad)}
  .card h3{font-family:var(--display);font-weight:600;font-size:1.18rem;color:var(--wine);line-height:1.1}
  .card .skill-tag{font-family:var(--mono);font-size:.6rem;letter-spacing:.12em;text-transform:uppercase;color:var(--wine-soft);margin-top:.2rem}
  .star-btn{margin-left:auto;background:none;border:none;cursor:pointer;font-size:1.2rem;line-height:1;color:var(--blush-deep);transition:transform .15s,color .2s;padding:.1rem}
  .star-btn.on{color:var(--brass)}
  .star-btn:hover{transform:scale(1.2)}
  .card .desc{font-size:.92rem;line-height:1.45;color:var(--ink);opacity:.9;flex:1}
  .card .status{font-family:var(--mono);font-size:.68rem;letter-spacing:.04em;color:var(--wine-soft);min-height:1em;display:flex;align-items:center;gap:.45rem}
  .dot{width:7px;height:7px;border-radius:50%;background:var(--blush-deep);flex:none}
  .dot.run{background:var(--rose);animation:pulse 1.1s infinite}
  .dot.done{background:var(--ok)}
  @keyframes pulse{0%,100%{opacity:1}50%{opacity:.35}}
  .card-actions{display:flex;align-items:center;gap:.5rem;margin-top:.4rem}
  .run-btn{
    flex:1;font-family:var(--mono);font-size:.74rem;letter-spacing:.14em;text-transform:uppercase;
    background:var(--grad);color:#fff;border:none;border-radius:10px;padding:.7rem;cursor:pointer;
    transition:filter .2s,transform .15s,box-shadow .2s;
  }
  .run-btn:hover{filter:brightness(1.07);box-shadow:0 8px 18px -8px rgba(124,58,237,.6)}
  .run-btn:active{transform:scale(.97)}
  .run-btn:disabled{opacity:.55;cursor:default}
  .icon-btn{background:var(--paper);border:1px solid var(--blush-deep);border-radius:10px;width:38px;height:38px;
    cursor:pointer;color:var(--wine-soft);font-size:1rem;display:grid;place-items:center;transition:border-color .2s,color .2s}
  .icon-btn:hover{border-color:var(--rose);color:var(--rose)}
  .upload-btn{font-family:var(--mono);font-size:.72rem;letter-spacing:.08em;text-transform:uppercase;
    background:var(--blush);border:1px solid var(--blush-deep);border-radius:10px;padding:.7rem .8rem;cursor:pointer;
    color:var(--wine);display:inline-flex;align-items:center;white-space:nowrap;transition:border-color .2s,color .2s,background .2s}
  .upload-btn:hover{border-color:var(--rose);color:var(--rose);background:var(--paper)}
  .upload-btn.busy{pointer-events:none;opacity:.92;border-color:var(--rose);color:var(--rose);background:var(--paper)}
  .spin{display:inline-block;width:12px;height:12px;border:2px solid var(--blush-deep);border-top-color:var(--rose);
    border-radius:50%;animation:ccspin .65s linear infinite;vertical-align:-1px;margin-right:.3rem}
  @keyframes ccspin{to{transform:rotate(360deg)}}

  /* add card */
  .add-card{
    border:1.5px dashed var(--blush-deep);background:transparent;border-radius:16px;
    display:grid;place-items:center;min-height:170px;cursor:pointer;color:var(--wine-soft);
    font-family:var(--mono);font-size:.74rem;letter-spacing:.14em;text-transform:uppercase;
    transition:border-color .2s,color .2s,background .2s;
  }
  .add-card:hover{border-color:var(--rose);color:var(--rose);background:rgba(203,58,157,.04)}
  .add-card .plus{font-size:1.8rem;display:block;text-align:center;margin-bottom:.3rem;font-family:var(--display)}

  /* ---- results zone ---- */
  .results-wrap{margin-top:2rem}
  .results-wrap > .eyebrow{margin-bottom:1rem}
  .result-card{background:var(--paper);border:1px solid var(--blush-deep);border-radius:16px;
    overflow:hidden;margin-bottom:1.1rem}
  .result-card .rc-head{display:flex;align-items:center;gap:.6rem;padding:.85rem 1.1rem;
    border-bottom:1px solid var(--blush);background:linear-gradient(90deg,var(--blush),#fff)}
  .result-card .rc-head .rc-name{font-family:var(--display);font-weight:600;color:var(--wine);font-size:1.02rem}
  .result-card .rc-head .rc-when{font-family:var(--mono);font-size:.64rem;letter-spacing:.08em;color:var(--wine-soft);margin-left:auto}
  .rc-refresh{font-family:var(--mono);font-size:.8rem;line-height:1;color:var(--wine-soft);background:var(--cream);
    border:1px solid var(--blush-deep);border-radius:8px;padding:.3rem .5rem;cursor:pointer;transition:border-color .2s,color .2s}
  .rc-refresh:hover{border-color:var(--rose);color:var(--rose)}
  .run-select{font-family:var(--mono);font-size:.64rem;color:var(--wine);background:var(--paper);
    border:1px solid var(--blush-deep);border-radius:8px;padding:.28rem .5rem;cursor:pointer;max-width:170px;transition:border-color .2s}
  .run-select:hover{border-color:var(--rose)}
  .rc-head .run-select{margin-left:auto}
  .rc-head .run-select ~ .rc-refresh, .rc-head .run-select ~ .rc-when{margin-left:0}
  .result-body{padding:1.1rem 1.2rem}
  .result-empty{color:var(--wine-soft);font-size:.92rem;line-height:1.5;text-align:center;padding:1.4rem .5rem}
  .result-empty b{color:var(--wine)}

  /* rendered-markdown brief (e.g. Morning Routine) */
  .md-brief{font-size:.93rem;line-height:1.5;color:var(--ink)}
  .md-brief h1{font-family:var(--display);font-weight:600;color:var(--wine);font-size:1.3rem;margin:.1rem 0 .9rem;line-height:1.1}
  .md-brief h2{font-family:var(--mono);font-size:.66rem;letter-spacing:.16em;text-transform:uppercase;color:var(--rose);
    margin:1.1rem 0 .5rem;display:flex;align-items:center;gap:.55rem}
  .md-brief h2::before{content:"";width:18px;height:1px;background:var(--brass);flex:none}
  .md-brief ul{list-style:none;margin:.2rem 0;padding:0;display:flex;flex-direction:column;gap:.45rem}
  .md-brief li{padding-left:1.05rem;position:relative;line-height:1.4}
  .md-brief li::before{content:"";position:absolute;left:0;top:.55em;width:5px;height:5px;border-radius:50%;background:var(--rose)}
  .md-brief strong{color:var(--wine);font-weight:600}
  .md-brief em{color:var(--wine-soft);font-style:italic}
  .md-brief blockquote{margin:.55rem 0;padding:.65rem .95rem;border-left:3px solid var(--rose);background:var(--blush);
    border-radius:0 9px 9px 0;font-family:var(--display);font-style:italic;color:var(--wine);font-size:1.02rem}
  .md-brief a{color:var(--rose);text-decoration:underline;text-underline-offset:2px}
  .md-brief hr{border:none;border-top:1px dashed var(--blush-deep);margin:1rem 0}
  .md-brief p{margin:.45rem 0}
  .md-brief .md-table{border-collapse:collapse;width:100%;margin:.7rem 0;font-size:.9rem}
  .md-brief .md-table th,.md-brief .md-table td{border:1px solid var(--blush-deep);padding:.45rem .7rem;text-align:left;vertical-align:top}
  .md-brief .md-table th{background:var(--blush);color:var(--wine);font-family:var(--mono);font-size:.66rem;letter-spacing:.06em;text-transform:uppercase;font-weight:600}
  .md-brief .md-table td{color:var(--ink)}
  .md-brief .md-table tr:nth-child(even) td{background:#fffdfc}

  /* ---- modal ---- */
  .overlay{position:fixed;inset:0;background:rgba(44,27,34,.45);backdrop-filter:blur(3px);
    display:none;place-items:center;z-index:50;padding:1.5rem}
  .overlay.show{display:grid}
  .modal{background:var(--paper);border-radius:18px;max-width:460px;width:100%;
    padding:1.8rem;box-shadow:0 30px 70px -30px rgba(124,58,237,.6);border:1px solid var(--blush-deep)}
  .modal h3{font-family:var(--display);font-weight:600;color:var(--wine);font-size:1.5rem;margin-bottom:.3rem}
  .modal p.sub{color:var(--wine-soft);font-size:.9rem;margin-bottom:1.3rem;line-height:1.4}
  .field{margin-bottom:1rem}
  .field label{display:block;font-family:var(--mono);font-size:.66rem;letter-spacing:.14em;text-transform:uppercase;color:var(--wine-soft);margin-bottom:.4rem}
  .field input,.field select,.field textarea{width:100%;border:1px solid var(--blush-deep);border-radius:10px;
    padding:.7rem .8rem;font-family:var(--body);font-size:.95rem;color:var(--ink);background:var(--cream)}
  .field input:focus,.field select:focus,.field textarea:focus{outline:2px solid var(--rose);outline-offset:1px;border-color:var(--rose)}
  .field textarea{resize:vertical;min-height:64px}
  .field .hint{font-size:.74rem;color:var(--wine-soft);margin-top:.3rem;line-height:1.35}
  .modal-actions{display:flex;gap:.7rem;margin-top:1.4rem}
  .btn-primary{flex:1;background:var(--rose);color:#fff;border:none;border-radius:10px;padding:.8rem;
    font-family:var(--mono);font-size:.74rem;letter-spacing:.14em;text-transform:uppercase;cursor:pointer;transition:background .2s}
  .btn-primary:hover{background:var(--wine)}
  .btn-ghost{background:transparent;color:var(--wine-soft);border:1px solid var(--blush-deep);border-radius:10px;
    padding:.8rem 1.1rem;font-family:var(--mono);font-size:.74rem;letter-spacing:.14em;text-transform:uppercase;cursor:pointer}
  .btn-ghost:hover{border-color:var(--rose);color:var(--rose)}

  /* toast */
  .toast{position:fixed;bottom:1.4rem;left:50%;transform:translateX(-50%) translateY(140%);
    background:var(--wine);color:var(--cream);padding:.8rem 1.3rem;border-radius:12px;font-size:.9rem;
    box-shadow:0 18px 40px -18px rgba(124,58,237,.7);z-index:60;transition:transform .35s cubic-bezier(.76,0,.24,1);max-width:90vw}
  .toast.show{transform:translateX(-50%) translateY(0)}
  .toast b{color:var(--rose-soft)}

  .hidden{display:none !important}
  @media (max-width:760px){
    .shell{grid-template-columns:1fr}
    .side{position:static;flex-direction:row;flex-wrap:wrap;gap:1rem}
    .side > div{flex:1;min-width:160px}
  }
</style>
</head>
<body>

<div class="topbar">
  <div class="brand">
    <div class="mark"><svg viewBox="0 0 100 100" fill="none" xmlns="http://www.w3.org/2000/svg" role="img" aria-label="arcAId logo"><circle cx="50" cy="50" r="46" stroke="#4F2FE3" stroke-width="4" fill="none"/><path d="M25 76 L25 47 A16 16 0 0 1 57 47 L57 76" stroke="#B98BD9" stroke-width="11" stroke-linecap="round" fill="none"/><path d="M34 76 L34 49 A7 7 0 0 1 48 49 L48 76" stroke="#4F2FE3" stroke-width="8" stroke-linecap="round" fill="none"/><rect x="63" y="47" width="8" height="29" rx="3" fill="#4F2FE3"/><path d="M67 26 L69.6 33.4 L77 36 L69.6 38.6 L67 46 L64.4 38.6 L57 36 L64.4 33.4 Z" fill="#B98BD9"/></svg></div>
    <div>
      <div class="name">arcAId</div>
      <div class="tag">Business Control Center</div>
    </div>
  </div>
  <div class="topbar-right">
    <span class="bridge-pill" id="bridgePill" title="">checking…</span>
    <button class="add-top" id="addTop">＋ Add skill</button>
  </div>
</div>

<div class="shell">
  <!-- sidebar -->
  <aside class="side">
    <div>
      <div class="label">Sections</div>
      <div class="nav" id="nav"></div>
    </div>
    <div>
      <div class="label">★ Favorites</div>
      <div class="fav-rail" id="favRail"></div>
    </div>
  </aside>

  <!-- main -->
  <main class="main" id="main"></main>
</div>

<!-- add/edit modal -->
<div class="overlay" id="overlay">
  <div class="modal">
    <h3 id="modalTitle">Add a skill</h3>
    <p class="sub">Drop any Claude skill into your control center. Give it a name, pick a section, and point it at the skill trigger.</p>
    <div class="field">
      <label>Skill name</label>
      <input id="fName" placeholder="e.g. Cash Flow Snapshot">
    </div>
    <div class="field">
      <label>What it does</label>
      <textarea id="fDesc" placeholder="One line the future you will understand at a glance."></textarea>
    </div>
    <div class="field">
      <label>Section</label>
      <select id="fSection">
        <option value="home">Home</option>
        <option value="finance">Finance</option>
        <option value="marketing">Marketing</option>
      </select>
    </div>
    <div class="field">
      <label>Scheduled task ID</label>
      <input id="fTask" placeholder="e.g. cash-flow-run">
      <div class="hint">The scheduled task this Run button triggers. Leave blank if there's no task yet — ask Claude to create one that runs your skill and writes its result into control-center-results/.</div>
    </div>
    <div class="modal-actions">
      <button class="btn-primary" id="saveSkill">Save skill</button>
      <button class="btn-ghost" id="cancelSkill">Cancel</button>
    </div>
  </div>
</div>

<div class="toast" id="toast"></div>

<!--
  RESULTS CHANNEL
  A finished skill run writes its dashboard here. Keyed by skill id.
  Each value is { "when": "Jun 24 · 9:14am", ... } plus ONE of:
    "md":  "<base64 of a Markdown brief>"   -> the UI renders + brand-styles it (use this for the morning routine)
    "b64": "<base64 of a self-contained HTML fragment>"  -> injected as-is (use for real HTML dashboards: finance, instagram)
  Base64 avoids any quoting/escaping problems. The UI decodes and renders it into #result-<id>.
-->
<script id="cc-results" type="application/json">{}</script>

<script>
(function(){
  "use strict";
  const LS_KEY = "feranse_cc_v1";

  // results injected by finished runs (read-only from the artifact HTML)
  const RESULTS = (function(){
    try{ return JSON.parse(document.getElementById("cc-results").textContent || "{}"); }
    catch(e){ return {}; }
  })();
  function b64decode(b64){
    try{
      return decodeURIComponent(Array.prototype.map.call(atob(b64),
        c=>"%"+("00"+c.charCodeAt(0).toString(16)).slice(-2)).join(""));
    }catch(e){ try{ return atob(b64); }catch(_){ return ""; } }
  }

  const SECTIONS = {
    home:      { eyebrow:"Start your day", title:"Home", lede:"Your daily kickoff. Run your morning routine and anything you want waiting for you each morning." },
    finance:   { eyebrow:"Money in motion", title:"Finance", lede:"Understand your cash. Run your statement analysis and see the dashboard right here, in one place." },
    marketing: { eyebrow:"Reach & resonance", title:"Marketing", lede:"Grow your audience. Audit your content and turn your own data into a posting strategy." }
  };

  // Seed skills — the three you asked for. Everything here is editable, removable, and favoritable.
  const SEED = [
    { id:"morning-routine", section:"home", name:"Morning Routine",
      desc:"Pulls your calendar, inbox, top priority, a lift for the day, and a headline or two into one brief.",
      tag:"morning-briefing", taskId:"morning-routine-run", driveTitle:"arcAId CC - morning-routine", driveFolderId:"1eH4aiIEgTV3WlBEufOHdpJ_oM2kE22KI", fav:true },
    { id:"bank-analysis", section:"finance", name:"Bank Statement Analysis",
      desc:"Upload your bank statement PDF — it categorizes every transaction and summarizes your cash flow.",
      tag:"bank-statement-analyzer", taskId:"bank-analysis-run",
      driveTitle:"arcAId CC - bank-analysis", driveFolderId:"16s4C40drZQy24z43uGtukMLGiPP0SkUi", driveInput:"arcAId CC - bank-input", needsFile:true, accept:"application/pdf,.pdf", uploadLabel:"PDF", fav:true },
    { id:"instagram", section:"marketing", name:"Instagram Dashboard",
      desc:"Upload your Instagram data export (.zip) — it audits your content and builds a posting strategy.",
      tag:"instagram-export-dashboard", taskId:"instagram-dashboard-run",
      driveTitle:"arcAId CC - instagram", driveFolderId:"1QFflEcDWBw8Hz_U68KNeH9ENg5YRc2KS", driveInput:"arcAId CC - instagram-input", needsFile:true, accept:".zip,application/zip", uploadLabel:"ZIP", fav:false }
  ];
  // Fallback map so older saved state (without taskId) still triggers the right task.
  const TASK_MAP = { "morning-routine":"morning-routine-run", "bank-analysis":"bank-analysis-run", "instagram":"instagram-dashboard-run" };

  function load(){
    try{
      const raw = localStorage.getItem(LS_KEY);
      if(raw){ const d = JSON.parse(raw); if(Array.isArray(d.skills)) return d; }
    }catch(e){}
    return { skills: SEED.map(s=>({...s})), active:"home" };
  }
  function save(){ try{ localStorage.setItem(LS_KEY, JSON.stringify(state)); }catch(e){} }

  let state = load();

  // Backfill seed-managed fields (taskId, driveTitle, tag) onto skills that were
  // persisted to localStorage before those fields existed — otherwise an old saved
  // Morning Routine has no driveTitle and the Drive read + Refresh button never appear.
  (function migrate(){
    let changed = false;
    for(const seed of SEED){
      const s = state.skills.find(x => x.id === seed.id);
      if(!s) continue;
      ["taskId","driveTitle","driveFolderId","driveInput","needsFile","accept","uploadLabel","tag"].forEach(k=>{ if(seed[k] && !s[k]){ s[k] = seed[k]; changed = true; } });
    }
    if(changed) save();
  })();

  let editingId = null;

  const main   = document.getElementById("main");
  const nav    = document.getElementById("nav");
  const favRail= document.getElementById("favRail");
  const overlay= document.getElementById("overlay");
  const toastEl= document.getElementById("toast");

  function monogram(name){ return (name.trim()[0]||"•").toUpperCase(); }
  function esc(s){ return (s||"").replace(/[&<>"]/g,c=>({"&":"&amp;","<":"&lt;",">":"&gt;",'"':"&quot;"}[c])); }

  // Tiny, safe Markdown renderer — built for the morning-briefing skill's output
  // structure (#, ##, **bold**, > quote, - lists, --- rules, [links](url)).
  function renderMarkdown(md){
    function inline(t){
      return esc(t)
        .replace(/\[([^\]]+)\]\((https?:\/\/[^)\s]+)\)/g,'<a href="$2" target="_blank" rel="noopener">$1</a>')
        .replace(/\*\*([^*]+)\*\*/g,"<strong>$1</strong>")
        .replace(/(^|[^*])\*([^*]+)\*/g,"$1<em>$2</em>");
    }
    const isRow = (l)=> /^\s*\|.*\|\s*$/.test(l||"");
    const cellsOf = (l)=>{ let s=(l||"").trim(); if(s.startsWith("|")) s=s.slice(1); if(s.endsWith("|")) s=s.slice(0,-1); return s.split("|").map(c=>c.trim()); };
    const isSep = (l)=>{ if(!isRow(l)) return false; const cs=cellsOf(l); return cs.length>0 && cs.every(c=>/^:?-+:?$/.test(c.replace(/\s/g,""))); };

    const lines = (md||"").replace(/\r/g,"").split("\n");
    let out = "", inList = false;
    const closeList = ()=>{ if(inList){ out += "</ul>"; inList = false; } };
    for(let i=0;i<lines.length;i++){
      const raw = lines[i];
      const line = raw.trimEnd();
      // table: a header row, then a |---|---| separator — tolerating blank lines
      // between rows (Google Docs inserts a blank line after every paragraph).
      if(isRow(line)){
        let p = i+1;
        while(p<lines.length && lines[p].trim()==="") p++;      // skip blanks to find the separator
        if(p<lines.length && isSep(lines[p])){
          closeList();
          const header = cellsOf(line);
          const body = [];
          let j = p+1;
          while(j<lines.length){
            if(lines[j].trim()===""){ j++; continue; }           // skip blank spacer lines between rows
            if(isRow(lines[j]) && !isSep(lines[j])){ body.push(cellsOf(lines[j])); j++; }
            else break;                                          // first non-row line ends the table
          }
          out += '<table class="md-table"><thead><tr>' + header.map(c=>"<th>"+inline(c)+"</th>").join("") +
                 "</tr></thead><tbody>" + body.map(r=>"<tr>"+r.map(c=>"<td>"+inline(c)+"</td>").join("")+"</tr>").join("") +
                 "</tbody></table>";
          i = j-1; continue;
        }
      }
      if(/^#\s+/.test(line)){ closeList(); out += "<h1>"+inline(line.replace(/^#\s+/,""))+"</h1>"; }
      else if(/^##\s+/.test(line)){ closeList(); out += "<h2>"+inline(line.replace(/^#+\s+/,""))+"</h2>"; }
      else if(/^>\s?/.test(line)){ closeList(); out += "<blockquote>"+inline(line.replace(/^>\s?/,""))+"</blockquote>"; }
      else if(/^(-{3,}|\*{3,})$/.test(line)){ closeList(); out += "<hr>"; }
      else if(/^[-*]\s+/.test(line)){ if(!inList){ out += "<ul>"; inList = true; } out += "<li>"+inline(line.replace(/^[-*]\s+/,""))+"</li>"; }
      else if(line === ""){ closeList(); }
      else { closeList(); out += "<p>"+inline(line)+"</p>"; }
    }
    closeList();
    return '<div class="md-brief">'+out+"</div>";
  }

  function toast(msg){
    toastEl.innerHTML = msg;
    toastEl.classList.add("show");
    clearTimeout(toast._t);
    toast._t = setTimeout(()=>toastEl.classList.remove("show"), 4200);
  }

  // ----- navigation -----
  function renderNav(){
    nav.innerHTML = "";
    const icons = { home:"⌂", finance:"$", marketing:"◎" };
    Object.keys(SECTIONS).forEach(key=>{
      const count = state.skills.filter(s=>s.section===key).length;
      const b = document.createElement("button");
      b.className = "nav-item" + (state.active===key ? " on":"");
      b.innerHTML = `<span class="ico">${icons[key]||"•"}</span><span>${SECTIONS[key].title}</span><span class="cnt">${count}</span>`;
      b.onclick = ()=>{ state.active=key; save(); render(); };
      nav.appendChild(b);
    });
  }

  function renderFav(){
    favRail.innerHTML = "";
    const favs = state.skills.filter(s=>s.fav);
    if(!favs.length){ favRail.innerHTML = `<div class="fav-empty">Star the skills you run most and they'll live here for one-click access.</div>`; return; }
    favs.forEach(s=>{
      const c = document.createElement("div");
      c.className = "fav-chip";
      c.innerHTML = `<span class="star">★</span><span>${esc(s.name)}</span>`;
      c.onclick = ()=>{ state.active=s.section; save(); render(); setTimeout(()=>runSkill(s.id), 60); };
      favRail.appendChild(c);
    });
  }

  // ----- main section -----
  function render(){
    renderNav(); renderFav();
    const sec = SECTIONS[state.active];
    const skills = state.skills.filter(s=>s.section===state.active);

    main.innerHTML = `
      <div class="sec-head">
        <div class="eyebrow">${esc(sec.eyebrow)}</div>
        <h2>${esc(sec.title)}</h2>
        <p class="lede">${esc(sec.lede)}</p>
      </div>
      <div class="grid" id="grid"></div>
      <div class="results-wrap" id="resultsWrap"></div>
    `;

    const grid = document.getElementById("grid");
    skills.forEach(s=> grid.appendChild(skillCard(s)) );
    const add = document.createElement("div");
    add.className = "add-card";
    add.innerHTML = `<div><span class="plus">＋</span>Add a skill</div>`;
    add.onclick = ()=> openModal(state.active);
    grid.appendChild(add);

    // results zone
    const rw = document.getElementById("resultsWrap");
    if(skills.length){
      rw.innerHTML = `<div class="eyebrow">Results</div>`;
      skills.forEach(s=> rw.appendChild(resultCard(s)) );
    }
  }

  function skillCard(s){
    const el = document.createElement("div");
    el.className = "card" + (s.fav ? " is-fav":"");
    el.innerHTML = `
      <div class="card-top">
        <div class="mono-badge">${monogram(s.name)}</div>
        <div style="min-width:0">
          <h3>${esc(s.name)}</h3>
          <div class="skill-tag">/${esc(s.tag||s.id)}</div>
        </div>
        <button class="star-btn ${s.fav?"on":""}" title="Favorite">${s.fav?"★":"☆"}</button>
      </div>
      <p class="desc">${esc(s.desc)}</p>
      <div class="status" data-status></div>
      <div class="card-actions">
        ${s.needsFile ? `<label class="upload-btn" title="Upload your file to Google Drive"><span class="ub-txt">⬆ ${esc(s.uploadLabel || "Upload")}</span><input type="file" accept="${esc(s.accept||"")}" hidden data-upload></label>` : ""}
        <button class="run-btn">▶ Run</button>
        <button class="icon-btn" title="Edit" data-edit>✎</button>
        <button class="icon-btn" title="Remove" data-del>×</button>
      </div>
    `;
    el.querySelector(".star-btn").onclick = ()=>{ s.fav=!s.fav; save(); render(); };
    el.querySelector(".run-btn").onclick  = ()=> runSkill(s.id, el);
    if(s.needsFile){
      const inp = el.querySelector("[data-upload]");
      if(inp) inp.onchange = ()=> handleUpload(s, inp, el.querySelector("[data-status]"));
    }
    el.querySelector("[data-edit]").onclick = ()=> openModal(s.section, s.id);
    el.querySelector("[data-del]").onclick  = ()=>{
      if(confirm(`Remove "${s.name}" from your control center? (The skill itself isn't deleted.)`)){
        state.skills = state.skills.filter(x=>x.id!==s.id); save(); render();
      }
    };
    return el;
  }

  // A skill's results live in its own Drive folder. The panel shows a run picker
  // (past runs, newest first) and renders the selected run. Custom skills without a
  // folder fall back to the legacy in-memory RESULTS slot.
  function resultCard(s){
    const wrap = document.createElement("div");
    wrap.className = "result-card";
    const folder = !!s.driveFolderId;
    wrap.innerHTML = `
      <div class="rc-head">
        <span class="rc-name">${esc(s.name)}</span>
        ${folder ? `<select class="run-select" title="Past runs" style="display:none"></select><button class="rc-refresh" title="Refresh runs from Drive">↻</button>` : ""}
        <span class="rc-when">${folder ? "loading…" : "awaiting first run"}</span>
      </div>
      <div class="result-body"><div class="result-canvas" id="result-${esc(s.id)}"></div></div>`;
    const canvas = wrap.querySelector(".result-canvas");
    if(folder){
      wrap.querySelector(".run-select").onchange = (e)=> showRun(s, e.target.value);
      wrap.querySelector(".rc-refresh").onclick = ()=> loadSkillRuns(s, {});
      canvas.innerHTML = `<div class="result-empty">Loading runs…</div>`;
      if(!runsLoaded.has(s.id)){ setTimeout(()=> loadSkillRuns(s, {}), 0); }
      else {                                         // already listed this session — render from cache
        fillRunSelect(s);
        if(CUR[s.id]) showRun(s, CUR[s.id]);
        else if((RUNS[s.id]||[]).length===0) canvas.innerHTML = `<div class="result-empty">No runs yet — click <b>▶ Run</b> to generate one.</div>`;
      }
    } else {
      const res = RESULTS[s.id];
      wrap.querySelector(".rc-when").textContent = res && res.when ? esc(res.when) : "awaiting first run";
      let body;
      if(res && res.md){ body = renderMarkdown(b64decode(res.md)); }
      else if(res && res.mdRaw){ body = renderMarkdown(res.mdRaw); }
      else if(res && res.b64){ body = b64decode(res.b64); }
      else if(res && res.htmlRaw){ body = res.htmlRaw; }
      else { body = `<div class="result-empty">Run <b>${esc(s.name)}</b> and its result will appear here.</div>`; }
      canvas.innerHTML = body;
    }
    return wrap;
  }

  // ----- run a skill by triggering its scheduled task -----
  function taskOf(s){ return s.taskId || TASK_MAP[s.id] || null; }

  // The runtime exposes window.cowork.runScheduledTask for artifacts. Detect it
  // and reflect readiness in the top-bar pill.
  let CAN_RUN = false;
  function detectRunner(){
    CAN_RUN = !!(window.cowork && typeof window.cowork.runScheduledTask === "function");
    const pill = document.getElementById("bridgePill");
    if(!pill) return;
    if(CAN_RUN){
      pill.className = "bridge-pill on";
      pill.textContent = "Run ready";
      pill.title = "Run buttons trigger each skill's scheduled task directly.";
      pill.onclick = ()=> toast("Run buttons trigger each skill's scheduled task. Reload a panel after a run to see its result.");
    } else {
      pill.className = "bridge-pill off";
      pill.textContent = "Open in Cowork";
      pill.title = "Triggering runs needs the Cowork runtime. Open this dashboard from the Cowork sidebar.";
      pill.onclick = ()=> toast("Open this dashboard from the Cowork sidebar to trigger runs.");
    }
  }

  function runSkill(id, cardEl){
    const s = state.skills.find(x=>x.id===id);
    if(!s) return;
    const statusEl = cardEl ? cardEl.querySelector("[data-status]") : null;
    const setStatus = (html)=>{ if(statusEl) statusEl.innerHTML = html; };
    const taskId = taskOf(s);

    if(!taskId){
      setStatus(`<span class="dot"></span> No task wired`);
      toast(`<b>${esc(s.name)}</b> has no scheduled task yet — ask Claude to create one.`);
      return;
    }
    if(!(window.cowork && typeof window.cowork.runScheduledTask === "function")){
      setStatus(`<span class="dot"></span> Open in Cowork to run`);
      toast(`Open this dashboard from the Cowork sidebar to trigger <b>${esc(s.name)}</b>.`);
      return;
    }

    const watching = !!s.driveFolderId;
    setStatus(`<span class="dot run"></span> Running…${watching ? " watching for the new run" : " reload when it finishes"}`);
    Promise.resolve(window.cowork.runScheduledTask(taskId)).then(()=>{
      s.lastRun = Date.now(); save();
      if(watching){
        setStatus(`<span class="dot run"></span> Running… new run will appear here`);
        toast(`Running <b>${esc(s.name)}</b>. Watching its folder — the new run drops in when it's ready.`);
        pollRuns(s);
      } else {
        setStatus(`<span class="dot done"></span> Started — result lands below`);
        toast(`Running <b>${esc(s.name)}</b>. Hit Reload on this panel when it's done.`);
      }
    }).catch(()=>{
      setStatus(`<span class="dot"></span> Couldn't start`);
      toast(`Couldn't start <b>${esc(s.name)}</b>. In the Scheduled sidebar, click "Run now" on <b>${esc(taskId)}</b> once to approve it.`);
    });
  }

  // ----- modal: add / edit -----
  function openModal(section, id){
    editingId = id || null;
    document.getElementById("modalTitle").textContent = id ? "Edit skill" : "Add a skill";
    const s = id ? state.skills.find(x=>x.id===id) : null;
    document.getElementById("fName").value    = s ? s.name : "";
    document.getElementById("fDesc").value    = s ? s.desc : "";
    document.getElementById("fSection").value = s ? s.section : (section||"home");
    document.getElementById("fTask").value    = s ? (s.taskId||"") : "";
    overlay.classList.add("show");
    setTimeout(()=>document.getElementById("fName").focus(), 50);
  }
  function closeModal(){ overlay.classList.remove("show"); editingId=null; }

  function saveSkill(){
    const name = document.getElementById("fName").value.trim();
    if(!name){ toast("Give your skill a name first."); return; }
    const desc = document.getElementById("fDesc").value.trim() || "Custom skill.";
    const section = document.getElementById("fSection").value;
    const taskId = document.getElementById("fTask").value.trim();

    if(editingId){
      const s = state.skills.find(x=>x.id===editingId);
      Object.assign(s, { name, desc, section, taskId });
    } else {
      const baseId = name.toLowerCase().replace(/[^a-z0-9]+/g,"-").replace(/^-|-$/g,"") || ("skill-"+Date.now());
      let uid = baseId, n=2; while(state.skills.some(x=>x.id===uid)){ uid = baseId+"-"+(n++); }
      state.skills.push({ id:uid, section, name, desc, tag:uid, taskId, fav:false });
    }
    save(); closeModal();
    state.active = section; render();
    toast(editingId ? "Saved." : `Added <b>${esc(name)}</b> to ${SECTIONS[section].title}.`);
  }

  document.getElementById("addTop").onclick = ()=> openModal(state.active);
  document.getElementById("saveSkill").onclick = saveSkill;
  document.getElementById("cancelSkill").onclick = closeModal;
  overlay.onclick = (e)=>{ if(e.target===overlay) closeModal(); };
  document.addEventListener("keydown", e=>{ if(e.key==="Escape") closeModal(); });

  // ----- local results directory -----
  // When this file is opened directly from the folder, results are read from
  // ./control-center-results/ : a results.json manifest { id: {when,file,fmt} }
  // plus one fragment file per skill. In the Cowork render these fetches just
  // fail (sandboxed) and we fall back to the embedded cc-results JSON above.
  const RESULTS_DIR = "control-center-results";
  async function loadLocalResults(){
    let manifest;
    try{
      const r = await fetch(RESULTS_DIR + "/results.json", { cache:"no-store" });
      if(!r.ok) return;
      manifest = await r.json();
    }catch(e){ return; }            // not in local-file mode — keep cc-results
    let changed = false;
    for(const id in manifest){
      const m = manifest[id] || {};
      if(!m.file) continue;
      try{
        const fr = await fetch(RESULTS_DIR + "/" + m.file, { cache:"no-store" });
        if(!fr.ok) continue;
        const txt = await fr.text();
        RESULTS[id] = { when: m.when || "", [ m.fmt === "md" ? "mdRaw" : "htmlRaw" ]: txt };
        changed = true;
      }catch(e){}
    }
    if(changed) render();
  }

  // ----- Google Drive results (a scheduled task writes a Doc, the dashboard reads it) -----
  // The skill's run writes its brief to a Google Doc titled `driveTitle`; we search for it,
  // read it back, and render it. Google Docs export escapes markdown punctuation with
  // backslashes (\#, \*\*, \-, \>), so we strip those before rendering.
  const DRIVE_SEARCH   = "mcp__a2f86dc4-aa35-4e4a-b76a-83f5f6480797__search_files";
  const DRIVE_READ     = "mcp__a2f86dc4-aa35-4e4a-b76a-83f5f6480797__read_file_content";
  const DRIVE_CREATE   = "mcp__a2f86dc4-aa35-4e4a-b76a-83f5f6480797__create_file";
  const DRIVE_DOWNLOAD = "mcp__a2f86dc4-aa35-4e4a-b76a-83f5f6480797__download_file_content";

  // Read a file as base64 (strip the data: URL prefix) for upload to Drive.
  function fileToB64(file){
    return new Promise((res,rej)=>{ const r=new FileReader(); r.onload=()=>res(String(r.result).split(",")[1]||""); r.onerror=()=>rej(new Error("couldn't read file")); r.readAsDataURL(file); });
  }
  // Upload a picked file to Drive under the skill's input title, so the task can fetch it.
  // Large files exceed the single-call upload limit, so we slice them into parts and upload
  // each separately; the task stitches the parts back together. Small files upload as one.
  const UPLOAD_CHUNK = 300 * 1024;   // raw bytes per part (~400KB base64 per call)
  function pad2(n){ return String(n).padStart(2,"0"); }
  async function uploadOne(title, blob, mime){
    const b64 = await fileToB64(blob);
    const r = await window.cowork.callMcpTool(DRIVE_CREATE, { title, base64Content: b64, contentMimeType: mime, disableConversionToGoogleType: true });
    if(r && r.isError) throw new Error((r.content && r.content[0] && r.content[0].text) || "upload error");
  }
  async function handleUpload(s, inp, statusEl){
    const file = inp.files && inp.files[0]; if(!file) return;
    const setStatus = (h)=>{ if(statusEl) statusEl.innerHTML = h; };
    const lbl = inp.closest(".upload-btn");
    const txt = lbl ? lbl.querySelector(".ub-txt") : null;
    const orig = txt ? txt.innerHTML : "";
    const setBtn = (h)=>{ if(txt) txt.innerHTML = h; };
    if(!(window.cowork && typeof window.cowork.callMcpTool === "function")){
      setStatus(`<span class="dot"></span> Open in Cowork to upload`);
      toast("Open this dashboard from the Cowork sidebar to upload files.");
      inp.value = ""; return;
    }
    const mime = file.type || "application/octet-stream";
    const total = Math.max(1, Math.ceil(file.size / UPLOAD_CHUNK));
    if(lbl) lbl.classList.add("busy");
    try{
      if(total === 1){
        setBtn(`<span class="spin"></span> Uploading`);
        setStatus(`<span class="spin"></span> Uploading ${esc(file.name)}…`);
        await uploadOne(s.driveInput, file, mime);
      } else {
        const uploadId = "U" + Date.now();
        for(let i=0;i<total;i++){
          setBtn(`<span class="spin"></span> ${i+1}/${total}`);
          setStatus(`<span class="spin"></span> Uploading ${esc(file.name)} — part ${i+1} of ${total}…`);
          const blob = file.slice(i*UPLOAD_CHUNK, (i+1)*UPLOAD_CHUNK);
          await uploadOne(`${s.driveInput} :: ${uploadId} :: ${pad2(i+1)}/${pad2(total)}`, blob, mime);
        }
      }
      setBtn(`✓ Uploaded`);
      setStatus(`<span class="dot done"></span> Uploaded ✓ — now click ▶ Run`);
      toast(`✓ <b>${esc(file.name)}</b> uploaded${total>1?` in ${total} parts`:""} — ready to run.`);
      setTimeout(()=>{ setBtn(orig); if(lbl) lbl.classList.remove("busy"); }, 2800);
    }catch(e){
      setBtn(orig); if(lbl) lbl.classList.remove("busy");
      setStatus(`<span class="dot"></span> Upload failed`);
      toast(`Upload failed: <b>${esc(String(e.message||e)).slice(0,180)}</b>`);
    }
    inp.value = "";
  }
  function parseMcp(r){ try{ return r.structuredContent ?? JSON.parse(r.content[0].text); }catch(e){ return null; } }
  function unescapeMd(t){ return (t||"").replace(/\\([#*_>\-\[\]()`~!+.])/g, "$1"); }
  function driveWhen(iso){ if(!iso) return "from Drive"; try{ return new Date(iso).toLocaleString(undefined,{month:"short",day:"numeric",hour:"numeric",minute:"2-digit"}); }catch(e){ return "from Drive"; } }
  function b64ToBytes(b64){ const bin = atob(b64); const u = new Uint8Array(bin.length); for(let i=0;i<bin.length;i++) u[i] = bin.charCodeAt(i); return u; }
  async function gunzip(bytes){ const s = new Blob([bytes]).stream().pipeThrough(new DecompressionStream("gzip")); return await new Response(s).text(); }

  // ----- per-skill run history (each skill has its own Drive folder) -----
  const RUNS = {};               // skillId -> [ {id, when, stamp, mimeType} ] newest first
  const CUR  = {};               // skillId -> selected fileId
  const HTMLC = {};              // fileId -> decoded html (cache)
  const runsLoaded = new Set();  // skills whose folder we've listed this session

  async function listRuns(folderId){
    if(!(window.cowork && typeof window.cowork.callMcpTool === "function")) return null;
    try{
      const r = await window.cowork.callMcpTool(DRIVE_SEARCH, {
        query: "parentId = '" + folderId + "' and mimeType != 'application/vnd.google-apps.folder'",
        pageSize: 50
      });
      const d = parseMcp(r);
      const files = ((d && d.files) || []).filter(f => (f.mimeType||"").indexOf("folder") < 0);
      files.sort((a,b)=> new Date(b.createdTime||b.modifiedTime||0) - new Date(a.createdTime||a.modifiedTime||0));
      return files.map(f => ({ id:f.id, when:(f.title||"").trim() || driveWhen(f.createdTime||f.modifiedTime), stamp:f.createdTime||f.modifiedTime, mimeType:f.mimeType }));
    }catch(e){ return null; }
  }

  async function loadRunHtml(f){
    if(HTMLC[f.id]) return HTMLC[f.id];
    let rd;
    try{ rd = parseMcp(await window.cowork.callMcpTool(DRIVE_DOWNLOAD, { fileId: f.id })); }
    catch(e){ return null; }
    if(!rd || !rd.content) return null;
    const bytes = b64ToBytes(rd.content);
    let html;
    if(bytes.length > 2 && bytes[0] === 0x1f && bytes[1] === 0x8b){          // gzip magic → decompress
      try{ html = await gunzip(bytes); }
      catch(e){ html = '<div style="padding:1rem;color:#b3433f">Couldn\'t decompress this run.</div>'; }
    } else {
      try{ html = new TextDecoder("utf-8").decode(bytes); }catch(e){ html = ""; }
    }
    HTMLC[f.id] = html;
    return html;
  }

  function panelEls(s){
    const canvas = document.getElementById("result-" + s.id);
    const card = canvas ? canvas.closest(".result-card") : null;
    return { canvas, sel: card && card.querySelector(".run-select"), whenEl: card && card.querySelector(".rc-when") };
  }
  function fillRunSelect(s){
    const { sel } = panelEls(s); if(!sel) return;
    const runs = RUNS[s.id] || [];
    sel.innerHTML = runs.map(r => `<option value="${esc(r.id)}">${esc(r.when)}</option>`).join("");
    sel.style.display = runs.length > 1 ? "" : "none";
    if(CUR[s.id]) sel.value = CUR[s.id];
  }
  async function showRun(s, fileId){
    const { canvas, whenEl } = panelEls(s);
    const run = (RUNS[s.id]||[]).find(r=>r.id===fileId);
    if(!canvas || !run) return;
    CUR[s.id] = fileId;
    if(whenEl) whenEl.textContent = run.when;
    if(!HTMLC[fileId]) canvas.innerHTML = `<div class="result-empty">Loading run…</div>`;
    const html = await loadRunHtml(run);
    if(CUR[s.id] !== fileId) return;                 // selection changed while loading — drop
    canvas.innerHTML = html || `<div class="result-empty">Couldn't load this run.</div>`;
  }

  // List a skill's folder, fill the run picker, render the chosen run (newest by default).
  async function loadSkillRuns(s, opts){
    opts = opts || {};
    if(!s.driveFolderId) return;
    runsLoaded.add(s.id);
    const { canvas, whenEl } = panelEls(s);
    if(!(window.cowork && typeof window.cowork.callMcpTool === "function")){
      if(canvas) canvas.innerHTML = `<div class="result-empty">Open this dashboard from the Cowork sidebar to load runs.</div>`;
      return;
    }
    const runs = await listRuns(s.driveFolderId);
    if(runs === null){ if(canvas) canvas.innerHTML = `<div class="result-empty">Couldn't reach Drive — try ↻.</div>`; return; }
    RUNS[s.id] = runs;
    if(!runs.length){ fillRunSelect(s); if(canvas) canvas.innerHTML = `<div class="result-empty">No runs yet — click <b>▶ Run</b> to generate one.</div>`; if(whenEl) whenEl.textContent = "no runs yet"; return; }
    let chosen = opts.forceId || CUR[s.id] || runs[0].id;
    if(!runs.some(r=>r.id===chosen)) chosen = runs[0].id;
    CUR[s.id] = chosen;
    fillRunSelect(s);
    await showRun(s, chosen);
  }

  // After triggering a task, watch the skill's folder for a brand-new run and switch to it.
  function pollRuns(s){
    const before = (RUNS[s.id] && RUNS[s.id][0] && RUNS[s.id][0].stamp) || null;
    let n = 0; const max = 30;
    const tick = async ()=>{
      n++;
      const runs = await listRuns(s.driveFolderId);
      if(runs && runs.length && (!before || (runs[0].stamp && runs[0].stamp !== before))){
        RUNS[s.id] = runs; CUR[s.id] = runs[0].id;
        fillRunSelect(s); await showRun(s, runs[0].id);
        toast(`<b>${esc(s.name)}</b> — new run added.`);
        return;
      }
      if(n < max) setTimeout(tick, 6000);
    };
    setTimeout(tick, 6000);
  }

  detectRunner();
  render();
  loadLocalResults();
})();
</script>
</body>
</html>
```

## Appendix B — Scheduled-task prompt templates

Create one task per skill. These are the reference prompts; customize the skill, folder id, palette hexes, input handling, and gzip-vs-plain per skill.

### B.1 — Morning brief (no upload, plain HTML output)
```text
Background run for the arcAId Business Control Center. Produce today's morning brief as a self-contained HTML card and save it as a new run in the skill's Drive folder. Keep it fast.

1) BUILD: Use the morning-briefing skill to assemble today's brief from the user's Google Calendar and Gmail (plus web search for the quote, a local event, and a short AI-news digest): today's schedule, the emails that need attention, the top 1-3 priorities, one short uplifting line, a nearby event if available, and 1-2 AI-news headlines. If a connector isn't authorized, keep a one-line note for that section.

2) RENDER AS HTML (HTML only): a SINGLE self-contained fragment - NO <html>/<head>/<body>, NO <script>, NO external resources (all inline). Wrap in <div class="cc-morning"> with ONE <style> block scoped under .cc-morning. Brand palette: background cream #F7F6FE; borders #DFD9F6; soft surface #EFEBFC; serif (Georgia) headings in violet #4F2FE3; small uppercase mono section labels in lilac #9B5FD6; accent tick purple #6D3BE6; body #1F1546; muted #6E63A0. Sections: title row with today's date (optional small inline <svg> sun in #9B5FD6), then Top Priorities, Today's Schedule, Needs a Reply, Quote of the Day (blockquote, #EFEBFC bg + #9B5FD6 left border), Near You, AI News. width:100%; box-sizing:border-box.

3) SAVE AS A NEW RUN: create_file with parentId = "<MORNING_FOLDER_ID>" (the morning-routine folder inside "Business Control Center"), title = a short human label of the CURRENT date and time (e.g. "Jul 6 - 9:14am"), textContent = your HTML, contentMimeType "text/html", disableConversionToGoogleType true. Do NOT reuse a fixed title and do NOT overwrite older files - each run is its own history entry in that folder.

4) Confirm the file was created in the folder, then stop. Never edit the artifact.
```

### B.2 — Bank statement (PDF upload, plain HTML output)
```text
Background run for the arcAId Business Control Center. Analyze the user's uploaded bank statement and save a brand-themed HTML cash-flow dashboard as a new run in the skill's Drive folder. Output is HTML.

1) FIND THE INPUT: Drive search_files for files titled exactly "arcAId CC - bank-input"; pick the most recent. If none, skip to step 4 and save a small HTML note telling the user to upload their statement with the upload button first.

2) DOWNLOAD & PARSE: download_file_content on that id -> base64 -> decode to a .pdf -> extract text with pdftotext (or pdfplumber). Don't use read_file_content on the PDF.

3) ANALYZE (reuse the skill's math): Run the bank-statement-analyzer skill - build_report.py computes the KPIs (revenue in, expenses out, net, avg monthly burn), a month-by-month bar chart, an expenses-by-category donut + legend (amount + %), and tables for largest expenses, top vendors, recurring/subscriptions, all as INLINE SVG.

4) RE-THEME INTO A CONTROL-CENTER HTML FRAGMENT: a SINGLE self-contained fragment - NO <html>/<head>/<body>, NO <script>, NO external resources (charts stay INLINE SVG). Wrap in <div class="cc-finance"> with ONE <style> block scoped under .cc-finance. Brand palette: card/page background cream #F7F6FE and white #fff; borders #DFD9F6; section titles (small UPPERCASE mono) lilac #9B5FD6; report title serif (Georgia) violet #4F2FE3; body #1F1546; muted #6E63A0 / #A79ED0; positive #4f9d7c, negative #b3433f. Charts (keep data/geometry, recolor): bars "money in" #4f9d7c, "money out" violet #4F2FE3, gridlines #EFEBFC. Donut category sequence largest->smallest (cycle if >10): #4F2FE3, #9B5FD6, #6D3BE6, #6E63A0, #C79BE6, #7C4DE8, #B24BD6, #D8C2F0, #5A3BD0, #B98BD9. Donut math: circle r=93 (circumference ~584.34); each segment stroke-dasharray "(fraction*584.34) (584.34 - fraction*584.34)", stroke-dashoffset negative cumulative-fraction*584.34, inside <g transform="rotate(-90 110 110)"> stroke-width 34. Keep KPI cards, bar-chart card, a row [donut+legend | recurring], a row [largest expenses | top vendors], footer.

5) SAVE AS A NEW RUN: create_file with parentId = "<BANK_FOLDER_ID>" (the bank-analysis folder inside "Business Control Center"), title = a short human label like the statement period + current date (e.g. "May 2026 - Jul 6"), textContent = your HTML, contentMimeType "text/html", disableConversionToGoogleType true. Do NOT overwrite older files - each run is its own history entry.

6) Confirm the file was created in the folder, then stop. Never edit the artifact. HTML only.
```

### B.3 — Instagram (ZIP upload, chunk reassembly, gzipped output)
```text
Background run for the arcAId Business Control Center. Analyze the user's Instagram export and save a brand-themed HTML audit as a new run in the skill's Drive folder. Output is HTML.

1) FIND INPUT: Drive search_files with query: title contains 'arcAId CC - instagram-input'. Input may be (a) a single file titled exactly "arcAId CC - instagram-input", OR (b) a CHUNK SET - parts titled "arcAId CC - instagram-input :: U<id> :: 01/07", "... :: 02/07", ... If chunk parts exist, group by the U<id> token, pick the group with the most recent createdTime, confirm all <total> parts, sort by NN. Use whichever is newest overall. If nothing found, skip to step 5 and save a short HTML note telling the user to upload their export with the upload button.

2) DOWNLOAD & REASSEMBLE: single file -> download_file_content, base64-decode to bytes, write .zip. Chunk set -> download_file_content each part in order, base64-decode each to raw bytes, CONCATENATE in order (binary) to reconstruct the .zip. Verify with `unzip -t`; if invalid, report missing parts. Then unzip.

3) ANALYZE with the skill: parse_export.py computes metrics.json (KPIs; timeline/day-of-week/hour arrays; auto-scored tests); read the contact sheet to write pillars + strategy_links + calendar.

4) RENDER AS A STATIC, BRAND-THEMED HTML FRAGMENT (the skill draws bars with JavaScript, but the control center does NOT run scripts - PRE-RENDER everything static). One <div class="cc-ig"> wrapper, ONE <style> block scoped under .cc-ig. NO <html>/<head>/<body>, NO <script>, NO external fonts/resources. Bars: <div class="bar"><div class="n">VALUE</div><div class="col cy" style="height:H%"></div><div class="x">LABEL</div></div>, H = round(value/max*100) (min 3% for nonzero). Peak month bar: class "col hi" + <div class="annot">peak</div>. Pillar bars: <div class="fill" style="width:P%;background:COLOR"></div>. Sections: hero (eyebrow + serif h1 + lede); KPI grid; cadence bars; timing (day-of-week, hour); fundamentals (pass/fail/partial rows); pillars; strategy (from->fix, mark strengths); weekly calendar; data note; footer. Brand palette: bg #F7F6FE, cards #fff, borders #DFD9F6, blush #EFEBFC; serif (Georgia) violet #4F2FE3 headings; lilac #9B5FD6 eyebrow/accents; purple #6D3BE6 "hi"/peak; secondary bars gradient #9B5FD6->#6D3BE6, peak/hi bars #6D3BE6->#3A1FA8; muted #6E63A0, faint #A79ED0; pass #4f9d7c, fail #b3433f, partial #cda05c; mono labels via ui-monospace.

5) SAVE AS A NEW RUN (gzip - it's large): write the HTML to a file, gzip it (gzip -9), base64 the .gz bytes, and create_file with parentId = "<INSTAGRAM_FOLDER_ID>" (the instagram folder inside "Business Control Center"), title = a short human label (e.g. "<handle> - Jul 6"), base64Content = that base64, contentMimeType "text/html", disableConversionToGoogleType true. The control center auto-decompresses the gzip. Do NOT overwrite older files - each run is its own history entry.

6) Confirm the file was created in the folder, then stop. Never edit the artifact. HTML only.
```

## Appendix C — Example result fragments (what a task should output)

Each is a self-contained, scoped, no-JS HTML fragment. Use them as the visual/structure model; the task fills in real data and matches the brand palette.

### C.1 — Morning brief (list/schedule/quote layout)
```html
<div class="cc-morning"><style>
.cc-morning{font-family:-apple-system,'Segoe UI',sans-serif;color:#1F1546;background:#F7F6FE;border:1px solid #DFD9F6;border-radius:14px;padding:20px 22px;width:100%;box-sizing:border-box;line-height:1.5}
.cc-morning .top{display:flex;align-items:center;gap:12px;border-bottom:1px solid #DFD9F6;padding-bottom:13px;margin-bottom:6px}
.cc-morning h1{font-family:Georgia,'Times New Roman',serif;color:#4F2FE3;font-size:23px;margin:0;font-weight:600;line-height:1}
.cc-morning .date{font-size:12px;color:#6E63A0;margin-top:4px}
.cc-morning .lbl{font-family:ui-monospace,'SFMono-Regular',Menlo,monospace;font-size:10px;letter-spacing:.18em;text-transform:uppercase;color:#9B5FD6;display:flex;align-items:center;gap:8px;margin:18px 0 9px}
.cc-morning .lbl::before{content:"";width:18px;height:1px;background:#6D3BE6}
.cc-morning ul{list-style:none;margin:0;padding:0;display:flex;flex-direction:column;gap:9px}
.cc-morning li{padding-left:16px;position:relative}
.cc-morning li::before{content:"";position:absolute;left:0;top:8px;width:5px;height:5px;border-radius:50%;background:#9B5FD6}
.cc-morning b{color:#4F2FE3;font-weight:600}
.cc-morning .sched{display:flex;gap:12px;padding:7px 0;border-bottom:1px solid #EFEBFC}
.cc-morning .sched:last-of-type{border-bottom:none}
.cc-morning .sched .t{font-family:ui-monospace,Menlo,monospace;font-size:13px;color:#4F2FE3;min-width:66px;font-weight:600}
.cc-morning blockquote{margin:0;padding:13px 16px;background:#EFEBFC;border-left:3px solid #9B5FD6;border-radius:0 9px 9px 0;font-family:Georgia,serif;font-style:italic;color:#4F2FE3;font-size:15px}
.cc-morning .foot{margin-top:18px;border-top:1px dashed #DFD9F6;padding-top:11px;font-size:11px;color:#6E63A0;font-style:italic}
</style>
<div class="top">
<svg width="34" height="34" viewBox="0 0 24 24" fill="none" stroke="#9B5FD6" stroke-width="1.6" stroke-linecap="round"><circle cx="12" cy="12" r="4" fill="#EFEBFC"></circle><line x1="12" y1="2" x2="12" y2="4.5"></line><line x1="12" y1="19.5" x2="12" y2="22"></line><line x1="2" y1="12" x2="4.5" y2="12"></line><line x1="19.5" y1="12" x2="22" y2="12"></line><line x1="4.9" y1="4.9" x2="6.7" y2="6.7"></line><line x1="17.3" y1="17.3" x2="19.1" y2="19.1"></line><line x1="4.9" y1="19.1" x2="6.7" y2="17.3"></line><line x1="17.3" y1="6.7" x2="19.1" y2="4.9"></line></svg>
<div><h1>Morning Brief</h1><div class="date">Thursday, June 25, 2026</div></div>
</div>
<div class="lbl">Top Priorities</div>
<ul>
<li><b>Renew your Atlassian API token</b> — the "n8n agents" token expires tomorrow (Jun 26); if it lapses your n8n automations lose API access.</li>
<li><b>Clear the feranse-agents review backlog</b> — five PRs are waiting on you (#31, #33, #34, #63, #64).</li>
<li><b>Reply to Stoyan Ivanov (AGF)</b> — Google Chat about creating an invoice; waiting on your next step.</li>
<li><b>Chase the VAL automation bug</b> — Alessandro confirmed the order→inventory push didn't work in his test.</li>
</ul>
<div class="lbl">Today's Schedule</div>
<div class="sched"><span class="t">8:30am</span><span>arcAId Daily SCRUM</span></div>
<div class="sched"><span class="t">8:45am</span><span>Daily Scrum, Exec Inner Circle · with Eric</span></div>
<div class="sched"><span class="t">9:30am</span><span>arcAId OST (Daily SCRUM) · with Ali</span></div>
<div class="sched"><span class="t">10:00am</span><span>VAL x arcAId · with Alessandro Labeque</span></div>
<div class="lbl">Needs a Reply</div>
<ul>
<li><b>Stoyan Ivanov (AGF)</b> — invoice question on Google Chat.</li>
<li><b>Alessandro Labeque (VAL)</b> — order→inventory automation didn't push data.</li>
<li><b>Atlassian</b> — API token expiry (renew before tomorrow).</li>
</ul>
<div class="lbl">Quote of the Day</div>
<blockquote>Whether you think you can, or you think you can't — you're right. — Henry Ford</blockquote>
<div class="lbl">Near You</div>
<ul><li>Jazz Festival kicks off tonight in the Quartier des Spectacles, Montreal.</li></ul>
<div class="lbl">AI News</div>
<ul>
<li><b>Anthropic keeps poaching Google AI talent</b> — Gemini researchers Jonas Adler and Alexander Pritzel are the latest to jump.</li>
<li><b>Anthropic expands in Korea</b> — new Seoul office plus TCS and DXC partnerships.</li>
<li><b>OpenAI &amp; Anthropic eye IPOs</b> — and are spending big on midterm AI-policy messaging.</li>
</ul>
<div class="foot">Pulled from your calendar &amp; inbox · Jun 25, 11:33am.</div>
</div>
```

### C.2 — Finance dashboard (KPIs + inline-SVG bar chart + inline-SVG donut + tables)
```html
<div class="cc-finance"><style>
.cc-finance{font-family:-apple-system,'Segoe UI',sans-serif;color:#1F1546;background:#F7F6FE;border:1px solid #DFD9F6;border-radius:14px;padding:20px 22px;width:100%;box-sizing:border-box;line-height:1.5}
.cc-finance .fh1{font-family:Georgia,'Times New Roman',serif;color:#4F2FE3;font-size:21px;margin:0;font-weight:600}
.cc-finance .fsub{color:#6E63A0;font-size:13px;margin-top:3px}
.cc-finance .kgrid{display:grid;grid-template-columns:repeat(auto-fit,minmax(140px,1fr));gap:12px;margin:18px 0}
.cc-finance .kpi{background:#fff;border:1px solid #DFD9F6;border-radius:12px;padding:14px}
.cc-finance .kl{color:#6E63A0;font-family:ui-monospace,Menlo,monospace;font-size:10px;text-transform:uppercase;letter-spacing:.1em}
.cc-finance .kv{font-size:21px;font-weight:650;margin-top:5px;color:#1F1546}
.cc-finance .pos{color:#4f9d7c} .cc-finance .neg{color:#b3433f}
.cc-finance .fcard{background:#fff;border:1px solid #DFD9F6;border-radius:12px;padding:18px;margin:14px 0}
.cc-finance .fcard h2{margin:0 0 12px;font-family:ui-monospace,Menlo,monospace;font-size:11px;text-transform:uppercase;letter-spacing:.12em;color:#9B5FD6}
.cc-finance .frow{display:grid;grid-template-columns:1.1fr 1fr;gap:16px}
@media(max-width:680px){.cc-finance .frow{grid-template-columns:1fr}}
.cc-finance .donut{display:flex;align-items:center;gap:16px;flex-wrap:wrap;justify-content:center}
.cc-finance .leg{flex:1;min-width:210px}
.cc-finance .lg{display:flex;align-items:center;gap:8px;font-size:12.5px;margin:5px 0}
.cc-finance .dot{width:11px;height:11px;border-radius:3px;flex:0 0 auto}
.cc-finance .ll{flex:1;min-width:110px} .cc-finance .lv{color:#6E63A0;font-variant-numeric:tabular-nums}
.cc-finance .lv em{color:#A79ED0;font-style:normal;margin-left:4px}
.cc-finance table{width:100%;border-collapse:collapse;font-size:12.5px}
.cc-finance th,.cc-finance td{text-align:left;padding:7px 9px;border-bottom:1px solid #EFEBFC}
.cc-finance th{color:#6E63A0;font-family:ui-monospace,Menlo,monospace;font-weight:600;font-size:10px;text-transform:uppercase;letter-spacing:.08em}
.cc-finance td.num,.cc-finance th.num{text-align:right;font-variant-numeric:tabular-nums}
.cc-finance .bars{display:flex;gap:16px;font-size:11.5px;color:#6E63A0;margin-top:8px}
.cc-finance .bars span::before{content:"";display:inline-block;width:10px;height:10px;border-radius:2px;margin-right:5px;vertical-align:middle}
.cc-finance .bin::before{background:#4f9d7c} .cc-finance .bout::before{background:#4F2FE3}
.cc-finance .muted{color:#A79ED0;text-align:center}
.cc-finance .ffoot{color:#A79ED0;font-size:11px;margin-top:16px;text-align:center;font-style:italic}
</style>
<div class="fh1">Bytewell Labs Inc. — Business Visa (****4417)</div>
<div class="fsub">Cash flow — 2026-05-01 to 2026-05-31 · 36 transactions</div>
<div class="kgrid">
  <div class="kpi"><div class="kl">Revenue in</div><div class="kv pos">$129.00</div></div>
  <div class="kpi"><div class="kl">Expenses out</div><div class="kv neg">$8,138.18</div></div>
  <div class="kpi"><div class="kl">Net cash flow</div><div class="kv neg">-$8,009.18</div></div>
  <div class="kpi"><div class="kl">Avg monthly burn</div><div class="kv">$8,138.18</div></div>
</div>
<div class="fcard"><h2>Cash flow by month</h2>
<svg viewBox="0 0 560 240" width="100%" height="240"><line x1="48" y1="212" x2="548" y2="212" stroke="#EFEBFC"/><text x="42" y="215" text-anchor="end" font-size="10" fill="#A79ED0">$0</text><line x1="48" y1="162" x2="548" y2="162" stroke="#EFEBFC"/><text x="42" y="165" text-anchor="end" font-size="10" fill="#A79ED0">$2,035</text><line x1="48" y1="112" x2="548" y2="112" stroke="#EFEBFC"/><text x="42" y="115" text-anchor="end" font-size="10" fill="#A79ED0">$4,069</text><line x1="48" y1="62" x2="548" y2="62" stroke="#EFEBFC"/><text x="42" y="65" text-anchor="end" font-size="10" fill="#A79ED0">$6,104</text><line x1="48" y1="12" x2="548" y2="12" stroke="#EFEBFC"/><text x="42" y="15" text-anchor="end" font-size="10" fill="#A79ED0">$8,138</text><rect x="270" y="208.8" width="26" height="3.2" fill="#4f9d7c" rx="2"/><rect x="300" y="12" width="26" height="200" fill="#4F2FE3" rx="2"/><text x="298" y="230" text-anchor="middle" font-size="10" fill="#6E63A0">2026-05</text></svg>
<div class="bars"><span class="bin">Money in</span><span class="bout">Money out</span></div>
</div>
<div class="frow">
  <div class="fcard"><h2>Expenses by category</h2>
  <div class="donut">
    <svg viewBox="0 0 220 220" width="200" height="200"><g transform="rotate(-90 110 110)"><circle cx="110" cy="110" r="93" fill="none" stroke="#4F2FE3" stroke-width="34" stroke-dasharray="157.173 427.163" stroke-dashoffset="0"/><circle cx="110" cy="110" r="93" fill="none" stroke="#9B5FD6" stroke-width="34" stroke-dasharray="125.581 458.756" stroke-dashoffset="-157.173"/><circle cx="110" cy="110" r="93" fill="none" stroke="#6D3BE6" stroke-width="34" stroke-dasharray="93.034 491.303" stroke-dashoffset="-282.754"/><circle cx="110" cy="110" r="93" fill="none" stroke="#6E63A0" stroke-width="34" stroke-dasharray="72.146 512.190" stroke-dashoffset="-375.788"/><circle cx="110" cy="110" r="93" fill="none" stroke="#C79BE6" stroke-width="34" stroke-dasharray="46.671 537.665" stroke-dashoffset="-447.934"/><circle cx="110" cy="110" r="93" fill="none" stroke="#7C4DE8" stroke-width="34" stroke-dasharray="32.311 552.025" stroke-dashoffset="-494.605"/><circle cx="110" cy="110" r="93" fill="none" stroke="#B24BD6" stroke-width="34" stroke-dasharray="24.190 560.146" stroke-dashoffset="-526.916"/><circle cx="110" cy="110" r="93" fill="none" stroke="#D8C2F0" stroke-width="34" stroke-dasharray="13.680 570.657" stroke-dashoffset="-551.106"/><circle cx="110" cy="110" r="93" fill="none" stroke="#5A3BD0" stroke-width="34" stroke-dasharray="12.363 571.973" stroke-dashoffset="-564.786"/><circle cx="110" cy="110" r="93" fill="none" stroke="#B98BD9" stroke-width="34" stroke-dasharray="7.187 577.149" stroke-dashoffset="-577.149"/></g></svg>
    <div class="leg">
      <div class="lg"><span class="dot" style="background:#4F2FE3"></span><span class="ll">Marketing &amp; advertising</span><span class="lv">$2,188.99 <em>27%</em></span></div>
      <div class="lg"><span class="dot" style="background:#9B5FD6"></span><span class="ll">Equipment &amp; hardware</span><span class="lv">$1,748.99 <em>21%</em></span></div>
      <div class="lg"><span class="dot" style="background:#6D3BE6"></span><span class="ll">Travel</span><span class="lv">$1,295.70 <em>16%</em></span></div>
      <div class="lg"><span class="dot" style="background:#6E63A0"></span><span class="ll">Software &amp; SaaS</span><span class="lv">$1,004.80 <em>12%</em></span></div>
      <div class="lg"><span class="dot" style="background:#C79BE6"></span><span class="ll">Office &amp; rent</span><span class="lv">$650.00 <em>8%</em></span></div>
      <div class="lg"><span class="dot" style="background:#7C4DE8"></span><span class="ll">Professional services</span><span class="lv">$450.00 <em>6%</em></span></div>
      <div class="lg"><span class="dot" style="background:#B24BD6"></span><span class="ll">Meals &amp; entertainment</span><span class="lv">$336.90 <em>4%</em></span></div>
      <div class="lg"><span class="dot" style="background:#D8C2F0"></span><span class="ll">Supplies &amp; materials</span><span class="lv">$190.52 <em>2%</em></span></div>
      <div class="lg"><span class="dot" style="background:#5A3BD0"></span><span class="ll">Bank fees &amp; interest</span><span class="lv">$172.18 <em>2%</em></span></div>
      <div class="lg"><span class="dot" style="background:#B98BD9"></span><span class="ll">Shipping &amp; postage</span><span class="lv">$100.10 <em>1%</em></span></div>
    </div>
  </div>
  </div>
  <div class="fcard"><h2>Recurring / subscriptions</h2>
  <table><thead><tr><th>Vendor</th><th class="num">Count</th><th class="num">Avg</th><th class="num">Total</th></tr></thead>
  <tbody><tr><td colspan="4" class="muted">None detected</td></tr></tbody></table>
  </div>
</div>
<div class="frow">
  <div class="fcard"><h2>Largest expenses</h2>
  <table><thead><tr><th>Date</th><th>Description</th><th class="num">Amount</th></tr></thead>
  <tbody>
  <tr><td>2026-05-12</td><td>APPLE STORE #R123</td><td class="num">$1,299.00</td></tr>
  <tr><td>2026-05-03</td><td>GOOGLE ADS 8829471</td><td class="num">$1,240.00</td></tr>
  <tr><td>2026-05-11</td><td>WEWORK MONTREAL</td><td class="num">$650.00</td></tr>
  <tr><td>2026-05-05</td><td>AIR CANADA</td><td class="num">$612.30</td></tr>
  <tr><td>2026-05-19</td><td>FACEBOOK ADS *META</td><td class="num">$520.00</td></tr>
  <tr><td>2026-05-26</td><td>NOTARY LEGAL SERVICES</td><td class="num">$450.00</td></tr>
  <tr><td>2026-05-24</td><td>BEST BUY #960</td><td class="num">$449.99</td></tr>
  <tr><td>2026-05-01</td><td>AMAZON WEB SERVICES</td><td class="num">$418.22</td></tr>
  </tbody></table>
  </div>
  <div class="fcard"><h2>Top vendors by spend</h2>
  <table><thead><tr><th>Vendor</th><th class="num">Total</th></tr></thead>
  <tbody>
  <tr><td>Apple Store</td><td class="num">$1,299.00</td></tr>
  <tr><td>Google Ads</td><td class="num">$1,240.00</td></tr>
  <tr><td>WeWork</td><td class="num">$650.00</td></tr>
  <tr><td>Air Canada</td><td class="num">$612.30</td></tr>
  <tr><td>Meta / Facebook Ads</td><td class="num">$520.00</td></tr>
  <tr><td>Notary Legal Services</td><td class="num">$450.00</td></tr>
  <tr><td>Best Buy</td><td class="num">$449.99</td></tr>
  <tr><td>Amazon Web Services</td><td class="num">$418.22</td></tr>
  </tbody></table>
  </div>
</div>
<div class="ffoot">Generated from the uploaded statement. Transfers and owner draws are excluded from revenue/expense totals.</div>
</div>
```

### C.3 — Instagram audit (KPIs + pre-rendered static bar charts + scorecard + pillars + strategy + calendar)
```html
<div class="cc-ig"><style>
.cc-ig{font-family:-apple-system,'Segoe UI',sans-serif;color:#1F1546;background:#F7F6FE;border:1px solid #DFD9F6;border-radius:14px;padding:22px 24px;width:100%;box-sizing:border-box;line-height:1.5}
.cc-ig .eyebrow{font-family:ui-monospace,Menlo,monospace;font-size:11px;letter-spacing:.16em;text-transform:uppercase;color:#9B5FD6;margin:0 0 10px}
.cc-ig h1{font-family:Georgia,'Times New Roman',serif;font-weight:600;font-size:26px;line-height:1.12;margin:0 0 12px;color:#4F2FE3}
.cc-ig h1 .at{color:#6D3BE6;font-style:italic}
.cc-ig .lede{font-size:15px;color:#6E63A0;max-width:64ch;margin:0}.cc-ig .lede b{color:#1F1546;font-weight:600}
.cc-ig .kpis{display:grid;grid-template-columns:repeat(3,1fr);gap:10px;margin:20px 0 8px}
.cc-ig .kpi{background:#fff;border:1px solid #DFD9F6;border-radius:11px;padding:14px}
.cc-ig .kpi .v{font-family:ui-monospace,Menlo,monospace;font-weight:700;font-size:21px;color:#1F1546}
.cc-ig .kpi .v.fail{color:#b3433f}.cc-ig .kpi .v.gold{color:#6D3BE6}
.cc-ig .kpi .l{font-size:11px;color:#6E63A0;margin-top:3px;line-height:1.3}
.cc-ig section{margin-top:30px}
.cc-ig .sec-head{display:flex;align-items:baseline;gap:12px;margin-bottom:4px}
.cc-ig .sec-num{font-family:ui-monospace,Menlo,monospace;font-size:12px;color:#6D3BE6;font-weight:600}
.cc-ig h2{font-family:Georgia,serif;font-size:20px;font-weight:600;margin:0;color:#4F2FE3}
.cc-ig .sec-sub{color:#6E63A0;font-size:13.5px;margin:4px 0 16px;max-width:72ch}
.cc-ig .card{background:#fff;border:1px solid #DFD9F6;border-radius:12px;padding:18px 20px}
.cc-ig .grid2{display:grid;grid-template-columns:1fr 1fr;gap:14px}
.cc-ig .card h3{margin:0 0 3px;font-size:14px;font-family:ui-monospace,Menlo,monospace;font-weight:600;color:#4F2FE3}
.cc-ig .cap{color:#A79ED0;font-size:11.5px;margin:0 0 16px;font-family:ui-monospace,Menlo,monospace}
.cc-ig .bars{display:flex;align-items:flex-end;gap:5px;height:160px}.cc-ig .bars.tall{height:190px}
.cc-ig .bar{flex:1;display:flex;flex-direction:column;justify-content:flex-end;align-items:center;gap:5px;min-width:0}
.cc-ig .bar .col{width:100%;background:#EFEBFC;border-radius:3px 3px 0 0;position:relative}
.cc-ig .bar .col.hi{background:linear-gradient(#6D3BE6,#3A1FA8)}.cc-ig .bar .col.cy{background:linear-gradient(#9B5FD6,#6D3BE6)}
.cc-ig .bar .n{font-family:ui-monospace,Menlo,monospace;font-size:10px;color:#6E63A0;height:12px}
.cc-ig .bar .x{font-family:ui-monospace,Menlo,monospace;font-size:9px;color:#A79ED0;white-space:nowrap}
.cc-ig .annot{position:absolute;top:-18px;left:50%;transform:translateX(-50%);background:#b3433f;color:#fff;font-family:ui-monospace,Menlo,monospace;font-weight:700;font-size:9px;padding:2px 6px;border-radius:5px}
.cc-ig .tests{display:flex;flex-direction:column;gap:9px}
.cc-ig .test{display:grid;grid-template-columns:74px 1fr auto;align-items:center;gap:14px;background:#fff;border:1px solid #DFD9F6;border-left-width:3px;border-radius:9px;padding:12px 15px}
.cc-ig .test.pass{border-left-color:#4f9d7c}.cc-ig .test.fail{border-left-color:#b3433f}.cc-ig .test.partial{border-left-color:#cda05c}
.cc-ig .badge{font-family:ui-monospace,Menlo,monospace;font-weight:700;font-size:10px;letter-spacing:.06em;text-align:center;padding:5px 0;border-radius:6px}
.cc-ig .badge.pass{background:rgba(79,157,124,.16);color:#3c7d62}.cc-ig .badge.fail{background:rgba(179,67,63,.14);color:#b3433f}.cc-ig .badge.partial{background:rgba(205,160,92,.18);color:#9a7330}
.cc-ig .tname{font-weight:600;font-size:14px;color:#1F1546}.cc-ig .why{color:#6E63A0;font-size:12px;margin-top:2px}
.cc-ig .metric{font-family:ui-monospace,Menlo,monospace;font-size:14px;font-weight:700;text-align:right}
.cc-ig .test.pass .metric{color:#3c7d62}.cc-ig .test.fail .metric{color:#b3433f}.cc-ig .test.partial .metric{color:#9a7330}
.cc-ig .pillar{margin-bottom:14px}.cc-ig .pillar-top{display:flex;justify-content:space-between;font-size:13px;margin-bottom:6px}
.cc-ig .pillar-top .pct{font-family:ui-monospace,Menlo,monospace;color:#6E63A0}.cc-ig .sig{color:#6D3BE6;font-family:ui-monospace,Menlo,monospace;font-size:10px}
.cc-ig .track{height:9px;background:#EFEBFC;border-radius:6px;overflow:hidden}.cc-ig .fill{height:100%;border-radius:6px}
.cc-ig .ex{color:#A79ED0;font-size:11px;font-family:ui-monospace,Menlo,monospace;margin-top:5px}
.cc-ig .links{display:flex;flex-direction:column;gap:12px}
.cc-ig .lk{display:grid;grid-template-columns:1fr 40px 1fr;align-items:stretch;border:1px solid #DFD9F6;border-radius:12px;overflow:hidden}
.cc-ig .lk .from{background:#fff;padding:15px 17px;border-left:3px solid #b3433f}
.cc-ig .lk .from.passdrv{border-left-color:#4f9d7c}
.cc-ig .lk .arrow{display:flex;align-items:center;justify-content:center;background:#EFEBFC;color:#6D3BE6;font-family:ui-monospace,Menlo,monospace;font-size:17px}
.cc-ig .lk .to{background:#FBF7FE;padding:15px 17px;border-left:3px solid #6D3BE6}
.cc-ig .lk .tag{font-family:ui-monospace,Menlo,monospace;font-size:10px;letter-spacing:.1em;text-transform:uppercase;margin:0 0 6px}
.cc-ig .lk .from .tag{color:#b3433f}.cc-ig .lk .from.passdrv .tag{color:#4f9d7c}.cc-ig .lk .to .tag{color:#6D3BE6}
.cc-ig .lk h4{margin:0 0 5px;font-size:14px;font-weight:600;color:#1F1546}.cc-ig .lk p{margin:0;font-size:12.5px;color:#6E63A0}.cc-ig .lk p b{color:#1F1546}
.cc-ig .cal{display:grid;grid-template-columns:repeat(7,1fr);gap:1px;background:#DFD9F6;border:1px solid #DFD9F6;border-radius:11px;overflow:hidden;margin-top:6px}
.cc-ig .cal .d{background:#fff;padding:11px 10px;min-height:108px}
.cc-ig .cal .dh{font-family:ui-monospace,Menlo,monospace;font-size:11px;color:#6E63A0;margin-bottom:8px;display:flex;justify-content:space-between}
.cc-ig .cal .dh .t{color:#A79ED0}
.cc-ig .slot{font-size:11px;border-radius:6px;padding:6px 8px;margin-bottom:5px;line-height:1.3}
.cc-ig .slot.reel{background:rgba(147,51,234,.15);border:1px solid rgba(147,51,234,.35)}
.cc-ig .slot.story{background:rgba(203,58,157,.12);border:1px solid rgba(203,58,157,.3)}
.cc-ig .slot.rest{color:#A79ED0;font-family:ui-monospace,Menlo,monospace;font-size:10px}
.cc-ig .slot .k{font-family:ui-monospace,Menlo,monospace;font-size:9px;text-transform:uppercase;letter-spacing:.08em;display:block;opacity:.9}
.cc-ig .slot.reel .k{color:#3A1FA8}.cc-ig .slot.story .k{color:#6D3BE6}
.cc-ig .note{background:#fff;border:1px solid #DFD9F6;border-radius:11px;padding:16px 18px;font-size:12.5px;color:#6E63A0;margin-top:20px}.cc-ig .note b{color:#1F1546}
.cc-ig .ffoot{margin-top:24px;border-top:1px dashed #DFD9F6;padding-top:16px;color:#A79ED0;font-family:ui-monospace,Menlo,monospace;font-size:11px;display:flex;justify-content:space-between;flex-wrap:wrap;gap:10px}
@media(max-width:680px){.cc-ig .kpis{grid-template-columns:repeat(2,1fr)}.cc-ig .grid2{grid-template-columns:1fr}.cc-ig .lk{grid-template-columns:1fr}.cc-ig .cal{grid-template-columns:1fr}}
</style><p class="eyebrow">Account Audit · Instagram Data Export · Jun 24 2026</p><h1>The memes were right.<br>The <span class="at">posting</span> failed the test cases.</h1><p class="lede">A University of Waterloo ECE meme account with genuinely strong content DNA — niche, named, share-ready humor — that never reached escape velocity. The export shows why: a launch-week sprint, then near-total silence. This is the autograder report, and the strategy to make it pass.</p><div class="kpis"><div class="kpi"><div class="v ">34</div><div class="l">posts ever published</div></div><div class="kpi"><div class="v fail">21</div><div class="l">followers reached</div></div><div class="kpi"><div class="v gold">79%</div><div class="l">of posts in peak month</div></div><div class="kpi"><div class="v fail">7.5 yrs</div><div class="l">since last post</div></div><div class="kpi"><div class="v ">97%</div><div class="l">static images · 3% video</div></div><div class="kpi"><div class="v fail">0%</div><div class="l">of posts use hashtags</div></div></div><section><div class="sec-head"><span class="sec-num">01</span><h2>The cadence</h2></div><p class="sec-sub">Posting volume over the account's life. The shape of this chart is usually the single most important fact about an account.</p><div class="card"><h3>Posts per period</h3><p class="cap">// 79% of posts in 2017-11; max gap 192d</p><div class="bars tall"><div class="bar"><div class="n">27</div><div class="col hi" style="height:100.0%"><div class="annot">peak</div></div><div class="x">Nov&#x27;17</div></div><div class="bar"><div class="n"></div><div class="col cy" style="height:0.0%"></div><div class="x">Dec</div></div><div class="bar"><div class="n"></div><div class="col cy" style="height:0.0%"></div><div class="x">Jan&#x27;18</div></div><div class="bar"><div class="n"></div><div class="col cy" style="height:0.0%"></div><div class="x">Feb</div></div><div class="bar"><div class="n"></div><div class="col cy" style="height:0.0%"></div><div class="x">Mar</div></div><div class="bar"><div class="n"></div><div class="col cy" style="height:0.0%"></div><div class="x">Apr</div></div><div class="bar"><div class="n">4</div><div class="col cy" style="height:14.8%"></div><div class="x">May</div></div><div class="bar"><div class="n">2</div><div class="col cy" style="height:7.4%"></div><div class="x">Jun</div></div><div class="bar"><div class="n"></div><div class="col cy" style="height:0.0%"></div><div class="x">Jul</div></div><div class="bar"><div class="n"></div><div class="col cy" style="height:0.0%"></div><div class="x">Aug</div></div><div class="bar"><div class="n"></div><div class="col cy" style="height:0.0%"></div><div class="x">Sep</div></div><div class="bar"><div class="n"></div><div class="col cy" style="height:0.0%"></div><div class="x">Oct</div></div><div class="bar"><div class="n"></div><div class="col cy" style="height:0.0%"></div><div class="x">Nov</div></div><div class="bar"><div class="n">1</div><div class="col cy" style="height:3.7%"></div><div class="x">Dec</div></div></div></div></section><section><div class="sec-head"><span class="sec-num">02</span><h2>When it posts</h2></div><p class="sec-sub">Day-of-week and hour patterns (local time, America/Toronto). Use these to find — or fix — a consistent slot aligned to when the audience is actually scrolling.</p><div class="grid2"><div class="card"><h3>By day of week</h3><p class="cap">// busiest day: Sat</p><div class="bars"><div class="bar"><div class="n">2</div><div class="col cy" style="height:18.2%"></div><div class="x">Mon</div></div><div class="bar"><div class="n">5</div><div class="col cy" style="height:45.5%"></div><div class="x">Tue</div></div><div class="bar"><div class="n">4</div><div class="col cy" style="height:36.4%"></div><div class="x">Wed</div></div><div class="bar"><div class="n">1</div><div class="col cy" style="height:9.1%"></div><div class="x">Thu</div></div><div class="bar"><div class="n">4</div><div class="col cy" style="height:36.4%"></div><div class="x">Fri</div></div><div class="bar"><div class="n">11</div><div class="col cy" style="height:100.0%"></div><div class="x">Sat</div></div><div class="bar"><div class="n">7</div><div class="col cy" style="height:63.6%"></div><div class="x">Sun</div></div></div></div><div class="card"><h3>By hour (local)</h3><p class="cap">// hour-of-day distribution</p><div class="bars"><div class="bar"><div class="n"></div><div class="col cy" style="height:0.0%"></div><div class="x">0</div></div><div class="bar"><div class="n"></div><div class="col cy" style="height:0.0%"></div><div class="x">1</div></div><div class="bar"><div class="n">7</div><div class="col cy" style="height:100.0%"></div><div class="x">2</div></div><div class="bar"><div class="n"></div><div class="col cy" style="height:0.0%"></div><div class="x">3</div></div><div class="bar"><div class="n"></div><div class="col cy" style="height:0.0%"></div><div class="x">4</div></div><div class="bar"><div class="n"></div><div class="col cy" style="height:0.0%"></div><div class="x">5</div></div><div class="bar"><div class="n"></div><div class="col cy" style="height:0.0%"></div><div class="x">6</div></div><div class="bar"><div class="n"></div><div class="col cy" style="height:0.0%"></div><div class="x">7</div></div><div class="bar"><div class="n">1</div><div class="col cy" style="height:14.3%"></div><div class="x">8</div></div><div class="bar"><div class="n">2</div><div class="col cy" style="height:28.6%"></div><div class="x">9</div></div><div class="bar"><div class="n">2</div><div class="col cy" style="height:28.6%"></div><div class="x">10</div></div><div class="bar"><div class="n"></div><div class="col cy" style="height:0.0%"></div><div class="x">11</div></div><div class="bar"><div class="n">6</div><div class="col cy" style="height:85.7%"></div><div class="x">12</div></div><div class="bar"><div class="n">2</div><div class="col cy" style="height:28.6%"></div><div class="x">13</div></div><div class="bar"><div class="n">5</div><div class="col cy" style="height:71.4%"></div><div class="x">14</div></div><div class="bar"><div class="n"></div><div class="col cy" style="height:0.0%"></div><div class="x">15</div></div><div class="bar"><div class="n">1</div><div class="col cy" style="height:14.3%"></div><div class="x">16</div></div><div class="bar"><div class="n"></div><div class="col cy" style="height:0.0%"></div><div class="x">17</div></div><div class="bar"><div class="n">2</div><div class="col cy" style="height:28.6%"></div><div class="x">18</div></div><div class="bar"><div class="n">4</div><div class="col cy" style="height:57.1%"></div><div class="x">19</div></div><div class="bar"><div class="n"></div><div class="col cy" style="height:0.0%"></div><div class="x">20</div></div><div class="bar"><div class="n">1</div><div class="col cy" style="height:14.3%"></div><div class="x">21</div></div><div class="bar"><div class="n">1</div><div class="col cy" style="height:14.3%"></div><div class="x">22</div></div><div class="bar"><div class="n"></div><div class="col cy" style="height:0.0%"></div><div class="x">23</div></div></div></div></div></section><section><div class="sec-head"><span class="sec-num">03</span><h2>Fundamentals report</h2></div><p class="sec-sub">Each row is a growth fundamental scored against the export. The failing rows are exactly what the strategy fixes.</p><div class="tests"><div class="test pass"><span class="badge pass">PASS</span><div><div class="tname">Niche, named-entity humor</div><div class="why">Marmoset, named profs, WaterlooWorks, course codes — hyper-relatable to the target audience.</div></div><div class="metric">★ strong</div></div><div class="test pass"><span class="badge pass">PASS</span><div><div class="tname">Meme-format range</div><div class="why">Drake, Two-Buttons, Philosoraptor, reaction crops — fluent in the visual language.</div></div><div class="metric">9+ formats</div></div><div class="test partial"><span class="badge partial">PARTIAL</span><div><div class="tname">Posting timing</div><div class="why">Good weekend instinct, but no fixed slot and missed weeknight deadline windows.</div></div><div class="metric">~half right</div></div><div class="test fail"><span class="badge fail">FAIL</span><div><div class="tname">Consistent cadence</div><div class="why">A launch sprint is not a schedule; concentration in one month means no posting habit.</div></div><div class="metric">79% / 1 mo</div></div><div class="test fail"><span class="badge fail">FAIL</span><div><div class="tname">Sustained presence</div><div class="why">The algorithm forgets dormant accounts; long silence resets reach.</div></div><div class="metric">2751 days</div></div><div class="test fail"><span class="badge fail">FAIL</span><div><div class="tname">Native video / Reels</div><div class="why">Reels are the discovery engine; static-only output barely reaches non-followers.</div></div><div class="metric">3% video</div></div><div class="test fail"><span class="badge fail">FAIL</span><div><div class="tname">Captions with hooks</div><div class="why">No caption = no hook, no CTA, no keyword context for ranking.</div></div><div class="metric">6% captioned</div></div><div class="test fail"><span class="badge fail">FAIL</span><div><div class="tname">Hashtag discoverability</div><div class="why">Zero/low hashtags means little non-follower reach via tags.</div></div><div class="metric">0% tagged</div></div><div class="test fail"><span class="badge fail">FAIL</span><div><div class="tname">Native-first publishing</div><div class="why">Recycled cross-posts are suppressed and miss native features.</div></div><div class="metric">100% crosspost</div></div></div></section><section><div class="sec-head"><span class="sec-num">04</span><h2>What it posts</h2></div><p class="sec-sub">Themed from the 34 recovered meme images. The humor clusters into four pillars — the most ownable ones are the most insider.</p><div class="card"><div class="pillar"><div class="pillar-top"><span><b>Courses &amp; profs</b></span><span class="pct">37%</span></div><div class="track"><div class="fill" style="width:37%;background:#6D3BE6"></div></div><div class="ex">VHDL · John Thistle · Paul Ward · Harmsworth&#x27;s calc final · discrete math · Lin Alg</div></div><div class="pillar"><div class="pillar-top"><span><b>General student relatable</b></span><span class="pct">29%</span></div><div class="track"><div class="fill" style="width:29%;background:#A79ED0"></div></div><div class="ex">all-nighters · #motivation · &quot;scratch and get a star&quot; · 11:11 wishes</div></div><div class="pillar"><div class="pillar-top"><span><b>Marmoset &amp; test cases</b><span class=sig>◆ signature</span></span><span class="pct">25%</span></div><div class="track"><div class="fill" style="width:25%;background:#4f9d7c"></div></div><div class="ex">&quot;submit to marmoset at 9:59&quot; · &quot;5/6 → 6/6 release cases&quot; · &quot;0/10 fixed code&quot;</div></div><div class="pillar"><div class="pillar-top"><span><b>Co-op / WaterlooWorks</b></span><span class="pct">9%</span></div><div class="track"><div class="fill" style="width:9%;background:#9B5FD6"></div></div><div class="ex">&quot;nOt SeLeCtEd&quot; · &quot;can&#x27;t fail 2A if you don&#x27;t pass 1B&quot; · co-op streams</div></div></div></section><section><div class="sec-head"><span class="sec-num">05</span><h2>Strategy, wired to the data</h2></div><p class="sec-sub">Every recommendation traces to a finding above. Left = what the export proved. Right = the fix.</p><div class="links"><div class="lk"><div class="from "><p class="tag">Data · §01</p><h4>79% of posts in month 1, then a cliff</h4><p>A burst with no system. Motivation ran out before a habit formed.</p></div><div class="arrow">→</div><div class="to"><p class="tag">Fix</p><h4>Ship 3×/week, batch monthly</h4><p>Make <b>one batch day per month</b> producing ~12 memes, scheduled out. A queue survives bad weeks; inspiration doesn't.</p></div></div><div class="lk"><div class="from "><p class="tag">Data · §03</p><h4>97% static images · 0 Reels</h4><p>The 2018 playbook. Static memes barely reach beyond existing followers now.</p></div><div class="arrow">→</div><div class="to"><p class="tag">Fix</p><h4>Make 60% of output Reels</h4><p>Animate the same formats: text-build reveals, trending audio over a punchline, 5-sec "POV: marmoset at 9:59." Same jokes, the format the algorithm pushes.</p></div></div><div class="lk"><div class="from "><p class="tag">Data · §03</p><h4>0 hashtags · 6% captioned</h4><p>No discoverability layer at all — invisible to anyone who doesn't already follow.</p></div><div class="arrow">→</div><div class="to"><p class="tag">Fix</p><h4>Caption hook + 5–8 niche tags</h4><p>Every post gets a one-line hook and a CTA (<b>"tag the friend still on 5/6"</b>) plus #uwaterloo #waterlooengineering #ecememes + course tags. Geotag UW.</p></div></div><div class="lk"><div class="from "><p class="tag">Data · §03</p><h4>100% auto-crossposted from Facebook</h4><p>IG suppresses recycled FB posts and you lose native features.</p></div><div class="arrow">→</div><div class="to"><p class="tag">Fix</p><h4>Post native-first, then repurpose</h4><p>Create in IG (Reels, captions, tags), then push the winners out to TikTok and the FB page — not the other way around.</p></div></div><div class="lk"><div class="from "><p class="tag">Data · §02</p><h4>Weekend-heavy, no fixed slot</h4><p>Right instinct, wrong precision — and it skipped peak ECE stress hours.</p></div><div class="arrow">→</div><div class="to"><p class="tag">Fix</p><h4>Lock slots to the grind</h4><p>Post <b>Sun–Thu, 9–11pm</b> (assignment crunch + procrastination scroll) and on known deadline eves. Keep a lighter Sat/Sun slot for casual hits.</p></div></div><div class="lk"><div class="from passdrv"><p class="tag">Strength · §04</p><h4>Marmoset / prof / co-op humor lands</h4><p>The most insider pillars are the most ownable — and the most shareable inside the niche.</p></div><div class="arrow">→</div><div class="to"><p class="tag">Amplify</p><h4>Turn pillars into recurring series</h4><p><b>"Marmoset Mondays,"</b> a prof-of-the-week, and a <b>Co-op Season</b> arc each Sept/Jan. Series give followers a reason to come back.</p></div></div><div class="lk"><div class="from "><p class="tag">Data · KPI</p><h4>Stuck at 21 followers</h4><p>Great content with no distribution never compounds.</p></div><div class="arrow">→</div><div class="to"><p class="tag">Fix</p><h4>Seed where ECE already gathers</h4><p>Drop posts into class group chats, cohort Discords, and <b>r/uwaterloo</b>; tag the engineering society. Insider memes spread by being shared.</p></div></div></div></section><section><div class="sec-head"><span class="sec-num">06</span><h2>A normal week</h2></div><p class="sec-sub">A sustainable cadence, slotted to the timing data — what the launch sprint should have become.</p><div class="cal"><div class="d"><div class="dh"><span>Mon</span><span class="t">9pm</span></div><div class="slot reel"><span class=k>Reel · Series</span>Marmoset Mondays</div></div><div class="d"><div class="dh"><span>Tue</span><span class="t">—</span></div><div class="slot story"><span class=k>Story</span>Poll: &quot;5/6 or 6/6?&quot;</div><div class="slot rest">// engage in comments</div></div><div class="d"><div class="dh"><span>Wed</span><span class="t">10pm</span></div><div class="slot reel"><span class=k>Reel · Courses</span>Prof-of-the-week</div></div><div class="d"><div class="dh"><span>Thu</span><span class="t">9pm</span></div><div class="slot reel"><span class=k>Reel · Relatable</span>Deadline-eve POV</div></div><div class="d"><div class="dh"><span>Fri</span><span class="t">—</span></div><div class="slot story"><span class=k>Story</span>Repost best-DM meme</div><div class="slot rest">// reshare to FB/TikTok</div></div><div class="d"><div class="dh"><span>Sat</span><span class="t">1pm</span></div><div class="slot reel"><span class=k>Static · Light</span>Weekend casual meme</div></div><div class="d"><div class="dh"><span>Sun</span><span class="t">—</span></div><div class="slot rest">REST + monthly batch day (~12)</div></div></div><div class="note"><b>How this maps back:</b> 3 Reels (§03 fix) · niche series from the strong pillars (§04) · Sun–Thu 9–11pm slots (§02) · one batch day so cadence survives bad weeks (§01). <b>Consistent beats heavy.</b></div></section><div class="note" style="margin-top:28px"><b>About this data.</b> Built from a personal Instagram data export (34 posts, 2017-11-04 → 2018-12-12). A personal export contains <b>post content, timestamps, and format</b> — but <b>not</b> likes, comments, reach, or impressions (those need an Insights export). So this measures posting behaviour and content, which is where the fixable problems live.</div><div class="ffoot"><span>./audit @uw_ece_meme — generated Jun 24 2026</span><span>report: 6 failed · 2 passed · 1 partial</span></div></div>
```

---

*End of build guide. If anything in the environment differs (different connector ids, different skills available), adapt the marked customization points but keep the platform constraints in Section 3 intact — they are what make the control center work inside Cowork.*
