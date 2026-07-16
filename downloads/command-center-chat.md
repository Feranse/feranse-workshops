# Business Control Center — Build Guide (Chat + Dashboard-Skills Edition)

**Purpose:** This document is a complete, self-contained instruction set. Attach it to a fresh **Claude chat** (claude.ai / Claude app — no Cowork required, no API keys, no MCP config) and Claude will reproduce the **Business Control Center**: a branded dashboard artifact whose Run buttons copy a premade prompt to the clipboard, which you paste into the chat; the chat runs a real skill, and that skill delivers its result straight into the artifact's own panel.

> **What this build is, in one paragraph.** There is no live bridge from an HTML artifact into this conversation — a plain file artifact can't send messages on its own. So the loop is: click **▶ Copy run** on a card → paste the copied prompt into this chat → the chat runs a **skill** built specifically for this board (a "-cc" skill) → that skill renders a small branded HTML fragment and **edits the dashboard artifact directly**, appending the fragment into a JSON channel embedded in the file → the chat re-shares the updated file → the panel shows the new run. Nothing calls the Anthropic API from inside the artifact, nothing needs MCP server URLs baked into the file, and no scheduled tasks are involved — every run happens live, in this chat, while you have it open.

**How to use this document (Claude, read this first):**
1. Work in a **normal claude.ai chat**. You will build one **HTML artifact** (the dashboard) and, for each skill it needs, **confirm that skill already exists — you do not create skills in this flow**; if one is missing, flag it to the user per Section 4, Step D (Appendix B is the guideline for whoever builds it later). Skills are the thing that do the real work (reading Calendar/Gmail, parsing a PDF/ZIP, writing a fragment); the artifact itself is just a branded launcher + results viewer.
2. **Do Section 1 (the interview) FIRST.** Collect brand info and the list of skills wanted on the board before building anything — ask ONE question at a time and wait for the answer before asking the next; don't bundle several questions into a single message.
3. Then follow Section 4 (Build Steps) **in order** — Step D (skills) matters as much as Step C (the artifact); a card whose skill doesn't exist will copy a prompt that goes nowhere.
4. Read Section 3 (Platform Constraints) carefully — these are the hard-won rules, several learned by trial and error in this exact build, that keep the artifact and its skills working together.
5. Appendix A is the **full reference artifact HTML** (the dashboard shell). Appendix B is the **dashboard skill template** — the exact shape (output contract + delivery step) a skill must have to plug into this board. Appendix C shows **example result fragments**. Reproduce from these, customizing only the parts called out in Step C/D.

---

## Section 1 — Collect this from the user first (the interview)

Do not build until you have all of this. **Ask ONE question at a time: pose a single question, wait for the answer, then ask the next.** Never paste the whole list at the user, never batch several questions into one message (even ones that feel related — brand name and tagline are two separate questions), and never re-ask something they've already answered. Offer clickable options where possible; keep each question short and in plain language — the user is a business owner, not a developer. Work through 1.1 → 1.2 → 1.3 → 1.4 → 1.5 in order, one item at a time within each, and only after ALL answers are collected give ONE summary confirmation (in Step A) before building.

### 1.1 Brand & identity
- **Company / dashboard name** (e.g. "arcAId") — used in the header.
- **One-line tagline** for the header subtitle (e.g. "Business Control Center").
- **Logo** — ask the user to attach it. **Prefer an SVG** (it inlines cleanly and scales); a PNG works too (embed as a `data:` URI). If they have neither, offer a gradient monogram.
- **Brand colors** — hex values from the user or derived from the logo: primary/heading, accent, background tint, border tint, body text, muted text. (Reference uses the arcAId indigo→orchid pair; Section 5 lists the exact variable slots.)
- **Font preference** — serif headings (Georgia fallback) + system sans body. NOTE: web fonts do **not** load in the artifact sandbox; use system stacks only.

> **⚠ The user's branding is the source of truth for the ENTIRE build.** Every brand value shipped in this document — the reference palette, the arcAId logo, the header name — is a **placeholder only**. Once the user provides their brand, build exclusively in it.
>
> **The specific trap:** the reference brand hexes appear **literally** inside each skill's fragment-rendering step and inside the Appendix C example fragments — skills run in their own turns and their output cannot read the artifact's CSS variables, so every color is hard-coded at render time. Re-theming only the artifact's `:root` leaves the shell on-brand while **every delivered result card still renders in the reference brand**. When setting the user's palette, propagate their hexes into **every skill's rendering step in the same pass**, and replace **every** reference hex — leaving none behind.

### 1.2 Company context (so skills produce accurate, relevant output)
- **What the business does** (industry, product/service, customers) — feeds skills like a morning brief with the right frame of reference.
- **Location / timezone** (for calendar/hour analytics and local-event lookups).
- Any **names/entities** that should be recognized (team, key clients) — optional.

### 1.3 Skills / modules & their layout
For each skill the user wants on the board, collect:
- **Skill name** (display) and a **short one-line description**.
- **Which section** it belongs to (top-level nav groups, e.g. Home / Finance / Marketing).
- **Does a "-cc" skill for this already exist** (ask, or check the environment's skill list)? If yes, reuse it. If not, that card will be **inert until a conforming skill exists** — building skills is NOT part of this guide's flow; Appendix B defines the shape any such skill must have (its output fragment contract and delivery step) so it integrates, and Step D covers flagging the gap to the user.
- **Does it need a file upload?** If yes: what file type (PDF, ZIP, CSV…) and a short button label (e.g. "PDF", "ZIP"). Uploads are **never** handled by the artifact — the skill's prompt tells the chat to ask the user to attach the file and wait, then the skill reads it the normal way (`/mnt/user-data/uploads/`).
- **What the result should look like** (KPIs, tables, charts). Charts must be **static** — inline SVG or plain divs with baked-in `style="height:H%"` — see Section 3.

The reference build ships three skills as a model (these already exist as saved skills; Appendix B defines the template shape they follow):
| Card | Section | Skill (package name) | Upload |
|---|---|---|---|
| Morning Routine · Dashboard | Home | `morning-briefing-cc` | none |
| Bank Statement Analysis · Dashboard | Finance | `bank-statement-analyzer-cc` | PDF |
| Instagram Dashboard · Dashboard | Marketing | `instagram-export-dashboard-cc` | ZIP |

**Note on naming:** this reference board ships *only* the "-cc" dashboard-delivery skills — no plain chat-only siblings. If the user already has ordinary chat skills for some of these tasks (e.g. a `morning-briefing` skill that just replies in chat), leave those alone; the "-cc" copy is a separate skill, not a replacement, and the board only ever needs the "-cc" one.

### 1.4 Connectors (authorization)
Tell the user which connectors each skill needs (e.g. Gmail + Google Calendar for a morning brief) and confirm they're authorized in claude.ai's connector settings before the corresponding card will work. There's no Drive requirement and nothing to wire into the artifact — connectors belong to the **skill**, not the dashboard, so this is a conversation with the user, not a build step.

### 1.5 Where to build
Nothing to provision. There's no Drive folder, no scheduled task, no server-side storage to set up before the first run — the dashboard creates its own run history the first time each skill delivers a result.

---

## Section 2 — What you're building (architecture)

- A single **HTML artifact** (`create_file` → a `.html` file) — the dashboard UI. Top bar (logo + name), left nav of **sections**, per-section **skill cards**, and per-card **result panels** underneath, titled "Dashboard results."
- Each card's **▶ Copy run** button copies a premade prompt to the clipboard (`navigator.clipboard`, with a hidden-textarea `execCommand` fallback for contexts where the Clipboard API isn't available). The user pastes it into this chat. **There is no other bridge** — a plain HTML file artifact cannot send messages into the conversation on its own; that capability (`sendPrompt`) only exists for a different kind of inline widget, not for downloadable file artifacts, so copy-paste is the deliberate, working design — not a workaround.
- Each prompt names a specific **skill** (a "-cc" skill) and, for upload cards, tells the chat to ask for the attachment first. The skill itself contains everything about *how* to do the analysis, *how* to render the result as a small branded fragment, and *how* to deliver it — the artifact's copied prompt is deliberately short because it doesn't need to repeat any of that.
- **Delivery is the skill editing the artifact.** After producing a fragment, a "-cc" skill's last step is: locate this HTML file, read the `<script id="cc-results" type="application/json">` block embedded in it, parse the JSON, prepend a new run under that card's key (`{"when", "ts", "b64"}`, base64-encoded fragment, newest first, capped at 10), write it back with `str_replace`, and re-share the file. The chat then confirms with one line — the actual result is only visible in the dashboard panel, not repeated in the conversation.
- **Two-layer run history.** The embedded `cc-results` JSON is the record any given copy of the file carries. On load, the artifact also copies every embedded run into `window.storage` (a per-user, persistent key-value store artifacts get in claude.ai) under `ccidx:<skillId>` / `ccrun:<skillId>:<ts>` keys. This means history isn't lost across future edits — a later delivery only has to know about *its own* run; the artifact merges embedded + stored history and shows the union in the run-history dropdown. If the artifact is opened as a bare local file (no `window.storage`), it falls back to `localStorage` so config still persists, though cross-edit run-merging needs `window.storage`.

Data flow per run:
```
[user clicks ▶ Copy run] → prompt copied to clipboard → user pastes into chat →
chat identifies the named "-cc" skill → (if upload skill) chat asks for the file,
user attaches it → skill does its real analysis → skill renders ONE scoped,
script-free HTML fragment → skill finds the dashboard artifact, str_replaces its
cc-results JSON (prepend run under the card's key, cap 10) → chat re-shares the file
→ [artifact, on load] merges cc-results + window.storage, renders the newest run
in that card's panel (+ run-history dropdown once 2+ runs exist)
```

---

## Section 3 — Platform constraints (the rules that make this work)

These are non-obvious and were each hit directly while building this reference. The included Appendix A/B already respect all of them.

1. **A file artifact cannot message the chat.** There is no `sendPrompt`-equivalent for downloadable HTML artifacts (that bridge exists only for a different, inline-widget surface). Therefore **Run buttons copy a prompt to the clipboard**, and the user is the one who pastes it into the chat. Don't try to build a "Run" button that calls an API or a bridge that isn't there — it will silently do nothing.
2. **`window.confirm()` / `alert()` / `prompt()` are blocked inside the artifact's sandboxed iframe** — they silently no-op. This is why an early version's "remove skill" (×) button appeared to do nothing: it was waiting on a native confirm dialog that could never resolve. Any destructive action needs an **in-DOM** confirm instead — the reference uses a two-click pattern (first click arms the button and shows "Sure? ×" for 3 seconds; a second click within that window confirms; anything else lets it time out and revert).
3. **Injected result HTML never runs `<script>`.** The dashboard renders a delivered fragment by setting `innerHTML`, which never executes scripts. So **every chart in a result must be static** — inline SVG (`<rect>` bars, `<circle>` donuts via `stroke-dasharray`) or divs with a **server-side-computed** `style="height:H%"` / `style="width:P%"` baked in before delivery. This is exactly why the reference skills replaced the *original* skills' JS-animated bar charts — those would render at zero height once dropped into another page's DOM.
4. **Sandbox CSP allows only a small CDN allowlist** (e.g. `cdnjs.cloudflare.com`) and blocks arbitrary external fonts/scripts/stylesheets. Inline all CSS in both the artifact and every delivered fragment; use system/serif font stacks only; embed images as `data:` URIs or inline SVG. Design for light mode.
5. **`localStorage` does not reliably work inside claude.ai artifacts** — use **`window.storage`** (`get`/`set`/`delete`/`list`, async, ~5 MB/key, no whitespace/slashes/quotes in keys) for anything that must survive a reload. Non-existent keys **throw** on `get` — always wrap in try/catch. Fall back to `localStorage` only for the case where the file is opened as a bare local file outside claude.ai (detect via `!window.storage`).
6. **A skill delivers by editing the artifact file directly, not by calling anything.** The delivery step is: `view` the artifact, regex/parse out the `<script id="cc-results">` JSON, prepend the new run, `str_replace` it back in, `present_files` again. Skills should **never** touch any other part of the file, and should cap each card's run array (10 is the reference default) so the file doesn't grow without bound.
7. **Every delivered fragment follows the same contract:** one scoped wrapper (`<div class="cc-<key>"> ... </div>`), exactly one `<style>` block with every rule scoped under that class, no `<html>/<head>/<body>`, no `<script>`, no external resources, and nothing that depends on JS running after injection. Skills that don't follow this will visually break the dashboard (leaking CSS) or silently render blank (relying on JS that never runs).
8. **Uploads never touch the artifact.** A card marked `needsFile` shows a small "📎 the chat will ask you to attach…" note instead of a file input — the actual attachment happens natively in this conversation, and the skill reads it from `/mnt/user-data/uploads/` the normal way. This sidesteps an entire category of problems (chunked browser uploads, base64 size limits, Drive round-trips) that earlier iterations of this exact board ran into.
9. **`window.storage` / config migration.** When new fields are added to a card's seed config, `migrate()` backfills them onto any previously-saved board so older saved state doesn't silently miss new behavior (e.g. a `dash` flag). The reference also actively **strips retired skill ids** out of saved state via a `REMOVED_IDS` list — useful any time a board's card lineup changes and old saved boards would otherwise keep showing a stale card.
10. **Never use `<form>` tags** in the artifact; use plain buttons/inputs with `onclick`/`onchange` handlers (the reference already does this throughout).

---

## Section 4 — Build steps (do these in order)

### Step A — Interview & confirm
Complete Section 1 **one question at a time** (single question → wait for the answer → next question; no batching, no re-asking). Only once every answer is collected, confirm the whole picture back in a **single summary message**: the sections, the list of cards (with sections, descriptions, uploads), the brand palette, the logo, and which skills already exist vs. need to be built. Get a yes before touching Step B.

### Step B — Check which skills already exist
For each card, check the environment's available skills for a matching "-cc" skill (or ask the user, or search — a skill's own `description` field is what makes it discoverable). Sort each card's skill into one of two buckets:
- **A dashboard-delivery ("-cc") skill exists** → reuse it untouched, just point the card at it (Step C).
- **It doesn't** (whether an ordinary chat-only skill exists or nothing does) → the card still gets built, but Step D covers telling the user it's inert until a skill conforming to Appendix B is saved to their profile. Do not build the skill as part of this flow.

### Step C — Build the artifact
1. Start from **Appendix A** (the full reference HTML). Customize only these parts:
   - **`:root` CSS variables** → the user's brand palette (Section 5). Also the `--grad` gradient, the **header name**, and the **`<title>`** — replace **ALL** reference hexes and brand strings; grep the file for each reference hex afterward to confirm none survived.
   - **The header logo** → replace the inline `<svg>` in `.brand .mark` (or a `data:` URI `<img>` for PNG). Keep it ~40–44px.
   - **`SECTIONS`** object → the user's section keys, titles, eyebrows, ledes.
   - **`SEED`** array → one entry per card: `{ id, section, name, desc, tag, dash:true, fav, needsFile?, uploadLabel? }`. `tag` is the skill name shown as `/tag` on the card and is purely cosmetic — the actual link to a skill is the **prompt**, not this field.
   - **`SKILL_PROMPTS`** → one short prompt per card, keyed by the card's `id` (see Step D for what belongs in each prompt — it should be a few lines, not a restatement of the skill's rules).
2. **Do not change** the machinery: the clipboard-copy run engine, `window.storage`/`localStorage` persistence + `migrate()`, the `cc-results` merge + run-history dropdown, the two-click delete confirm. It's all in Appendix A and already accounts for every constraint in Section 3.
3. **The board ships EMPTY.** The embedded results channel — `<script id="cc-results" type="application/json">` — must ship as exactly `{}`. Never bake a sample, mock, or placeholder result into it, and never deliver a demo run through a skill to "show the layout" — a mock run is indistinguishable from real output. Every panel must show its empty state ("No runs yet — ▶ Copy run…") until the user performs the first real run themselves (▶ Copy run → paste into the chat). The Appendix C fragments are reference models for **writing skill renderers only** — they are never loaded into the board.
4. Validate the JS parses before publishing (e.g. extract the inline `<script>` body and run it through a syntax check).
5. Publish as a single **HTML artifact**. That's the whole deliverable for this step — no bridge config, no MCP server list, no scheduled tasks.

### Step D — Confirm every card's skill exists (do NOT build skills here)
A card whose prompt names a skill that isn't saved to the user's profile will copy a prompt that goes nowhere. For each card, confirm a matching dashboard-delivery ("-cc") skill exists; if it does, move on. If it doesn't, **do not create one** — authoring skills is outside this guide's scope. Instead, tell the user plainly which cards are inert and why, and point them at **Appendix B**: it is the guideline defining the exact shape a skill must have (fragment contract + `cc-results` delivery step, rendered in the user's brand palette — not the reference one) for whoever builds it, whenever they choose to, as its own separate task. Once such a skill is saved to the user's profile (packaged `.skill` file → **Save skill** clicked), the card starts working with no changes to the board.

### Step E — Wire the cards to their skills
Back in the artifact (Step C), make sure each card's `SKILL_PROMPTS` entry:
- Names the exact skill by its package name (e.g. "using the morning-briefing-cc skill").
- For upload cards, opens with an instruction to ask for the attachment first and wait (e.g. "First ask me to attach my bank statement PDF to this chat and wait for my upload.").
- Tells the skill which dashboard key to deliver into (e.g. `under the "bank-analysis-dash" key`) — this only matters if it isn't already obvious from the skill's own SKILL.md, which typically hard-codes its expected key.
- Does **not** restate fragment CSS, palette hexes, or the delivery mechanics — all of that belongs in the skill, not the prompt. A short prompt is a sign the split is correct.

### Step F — Verify
- **Fresh load, before any run:** every panel shows its empty state ("No runs yet…"), `cc-results` is `{}`, and no sample content exists anywhere on the board.
- Reload the artifact. The top-bar pill should read **"Clipboard mode."**
- Click ▶ Copy run on a no-upload card, paste into this chat, confirm the named skill triggers, and check the reply is a short one-line confirmation (not the full result — that means delivery is misconfigured).
- Re-open (or re-fetch) the artifact and confirm the new run appears in that card's panel; run a second time and confirm the run-history dropdown appears.
- Test an upload card: click ▶ Copy run, paste, confirm the chat asks for the file before doing anything else, attach it, and confirm delivery.
- Click × on a card, confirm it turns into "Sure? ×", and confirm a second click actually removes it (this is the constraint-2 fix — if it silently does nothing, something reintroduced a native `confirm()`).
- Confirm the brand palette and logo render correctly **in the shell AND inside the first real delivered result card** — an off-brand result means reference hexes were missed in a skill's rendering step (see the branding callout in 1.1).

---

## Section 5 — Brand palette mapping

The artifact themes everything through CSS variables in `:root` (reference values shown; replace with the user's):

| Variable | Role | Reference value |
|---|---|---|
| `--cream` | page background tint | `#FAF9FE` |
| `--blush` | soft surface | `#F2EDFA` |
| `--blush-deep` | borders | `#E4DDF6` |
| `--rose` | accent (buttons, labels) | `#C789DC` |
| `--rose-soft` | light accent | `#DBB0E9` |
| `--wine` | headings / primary | `#4E39E0` |
| `--wine-soft` | muted text | `#77719F` |
| `--ink` | body text | `#221A4E` |
| `--brass` | small accent ticks | `#8A63E8` |
| `--paper` | card background | `#FFFFFF` |
| `--ok` | success/positive | `#4f9d7c` |
| `--grad` | logo gradient (buttons, mark) | `linear-gradient(135deg,#4E39E0,#C789DC)` |

**Delivered fragments** (Appendix C) use the same hexes literally — a skill has no access to the artifact's CSS variables, so its rendering script must hard-code the palette. When you set a custom palette, feed those hexes into every skill's rendering step too, or delivered fragments won't match the shell. Keep **green for positive/pass and red for negative/fail** regardless of brand — they read as status.

**Footer credit.** The shell carries a fixed footer line — `Training & Workshops by arcAId | Technology by Feranse` (the `.site-foot` element in Appendix A). Keep it in any rebuild.

Delivered fragments use scoped wrapper classes so their styles never leak: `.cc-morning`, `.cc-finance`, `.cc-ig` in the reference (name new ones `.cc-<card-id>`). Each is `<div class="cc-X"><style>/* rules all prefixed .cc-X */</style> … </div>` — no `<html>/<head>/<body>`, no `<script>`, no external resources.

---

## Section 6 — Chart recipes (static, no JS)

**Bar chart (cadence/day/hour, cash-flow-by-month):** each bar is a flex column; the bar height is baked in server-side as `style="height:H%"` where `H = round(value/max*100)` (a small minimum like 3% for nonzero, 0% for zero). Colors inline. This must be computed before delivery — never left for client-side JS to fill in, since the artifact's `innerHTML` injection never executes it.

**Donut (expenses by category):** one `<circle r=93>` per segment inside `<g transform="rotate(-90 110 110)">`, `stroke-width=34`, circumference ≈ `2π·93 = 584.34`. For a segment with fraction `f`: `stroke-dasharray = "(f*584.34) (584.34 − f*584.34)"` and `stroke-dashoffset = -(cumulativeFractionBefore * 584.34)`. Legend is plain HTML with colored dots.

See Appendix C for complete, working examples of both.

---
## Appendix A — Full reference artifact HTML (Chat + Dashboard-Skills Edition)

Reproduce this artifact, customizing only the parts called out in Step C (palette, logo, `SECTIONS`, `SEED`, `SKILL_PROMPTS`). Everything else — the clipboard-copy run engine, persistence + `migrate()`, the `cc-results` merge and run-history dropdown, the two-click delete confirm — should stay as-is; each piece exists because of a specific constraint in Section 3.

```html
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>arcAId · Business Control Center</title>
<!-- CLIPBOARD EDITION: JSZip removed — uploads happen in the chat that runs the skill,
     not in the artifact, so nothing is parsed client-side anymore. -->
<style>
  :root{
    color-scheme: light;
    /* arcAId brand — indigo + orchid, sampled from the logo (#4E39E0 / #C789DC) */
    --cream:#FAF9FE;        /* page background (indigo-tinted white) */
    --blush:#F2EDFA;        /* soft orchid surface */
    --blush-deep:#E4DDF6;   /* borders */
    --rose:#C789DC;         /* orchid accent (labels, chips) */
    --rose-soft:#DBB0E9;    /* light orchid */
    --wine:#4E39E0;         /* indigo (headings / primary) */
    --wine-soft:#77719F;    /* muted text */
    --ink:#221A4E;          /* body text */
    --brass:#8A63E8;        /* small accent ticks (indigo→orchid midpoint) */
    --paper:#FFFFFF;        /* card background */
    --ok:#4f9d7c;           /* success/positive */
    --grad:linear-gradient(135deg,#4E39E0,#C789DC);  /* the logo gradient */
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
  .icon-btn{background:var(--paper);border:1px solid var(--blush-deep);border-radius:10px;min-width:38px;height:38px;padding:0 .5rem;
    cursor:pointer;color:var(--wine-soft);font-size:1rem;display:grid;place-items:center;transition:border-color .2s,color .2s;white-space:nowrap}
  .icon-btn:hover{border-color:var(--rose);color:var(--rose)}
  .upload-btn{font-family:var(--mono);font-size:.72rem;letter-spacing:.08em;text-transform:uppercase;
    background:var(--blush);border:1px solid var(--blush-deep);border-radius:10px;padding:.7rem .8rem;cursor:pointer;
    color:var(--wine);display:inline-flex;align-items:center;white-space:nowrap;transition:border-color .2s,color .2s,background .2s}
  .upload-btn:hover{border-color:var(--rose);color:var(--rose);background:var(--paper)}
  /* CLIPBOARD EDITION: chip shown on skills whose chat run will request a file attachment */
  .file-note{font-family:var(--mono);font-size:.62rem;letter-spacing:.04em;color:var(--brass);
    background:var(--blush);border:1px dashed var(--blush-deep);border-radius:9px;padding:.45rem .6rem;line-height:1.4}
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
  /* REBRAND: brand credit footer */
  .site-foot{text-align:center;padding:1.1rem 1rem 1.4rem;font-family:var(--mono);font-size:.62rem;
    letter-spacing:.14em;text-transform:uppercase;color:var(--wine-soft);border-top:1px solid var(--blush-deep);
    background:var(--cream)}
  .site-foot b{color:var(--wine);font-weight:600}
  .site-foot .sep{color:var(--rose);margin:0 .5rem}
</style>
</head>
<body>

<div class="topbar">
  <div class="brand">
    <div class="mark"><svg viewBox="0 0 48 48" fill="none" xmlns="http://www.w3.org/2000/svg" role="img" aria-label="arcAId logo">
      <circle cx="24" cy="24" r="21.5" stroke="#4E39E0" stroke-width="3.4" fill="#fff"/>
      <path d="M11.5 32.5 v-8.5 a9.5 9.5 0 0 1 19 0 v8.5 h-5.4 v-8.5 a4.1 4.1 0 0 0-8.2 0 v8.5 z" fill="#C789DC"/>
      <path d="M16.9 32.5 v-8.5 a4.1 4.1 0 0 1 8.2 0 v8.5 h-2.9 v-8.5 a1.2 1.2 0 0 0-2.4 0 v8.5 z" fill="#4E39E0"/>
      <rect x="33.4" y="20.6" width="4.6" height="11.9" rx="2.1" fill="#4E39E0"/>
      <path d="M36.6 9.2 c.7 2.8 1.4 3.6 4.2 4.4 c-2.8 .8-3.5 1.6-4.2 4.4 c-.7-2.8-1.4-3.6-4.2-4.4 c2.8-.8 3.5-1.6 4.2-4.4 z" fill="#C789DC"/>
    </svg></div>
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

<!-- REBRAND: brand credit footer -->
<div class="site-foot">Training &amp; Workshops by <b>arcAId</b> <span class="sep">|</span> Technology by <b>Feranse</b></div>

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
      <!-- CLIPBOARD EDITION: was "Scheduled task ID" (Cowork) then an API run prompt -->
      <label>Run prompt</label>
      <textarea id="fTask" placeholder="What should the chat do when you paste this? e.g. Run the /friday-brief skill for the last 7 days."></textarea>
      <div class="hint">▶ Copy run puts this on your clipboard — paste it into your Claude chat and the chat runs it. If a file is needed, start with: "first ask me to attach my … and wait for my upload."</div>
    </div>
    <div class="modal-actions">
      <button class="btn-primary" id="saveSkill">Save skill</button>
      <button class="btn-ghost" id="cancelSkill">Cancel</button>
    </div>
  </div>
</div>

<div class="toast" id="toast"></div>

<!--
  RESULTS CHANNEL (restored for the DASHBOARD skill copies)
  The chat Claude delivers a dashboard run by editing this artifact: it appends an entry to
  the skill's "runs" array below (newest first). Each entry: { "when": "Jul 13 · 3:30pm",
  "ts": <epoch ms>, "b64": "<base64 of a self-contained HTML fragment>" }.
  On load the artifact merges these into window.storage (when available) so history
  survives artifact edits, then renders the newest run in the skill's panel.
-->
<script id="cc-results" type="application/json">{}</script>

<script>
(function(){
  "use strict";
  const LS_KEY = "feranse_cc_v1";

  // DASHBOARD RUNS: parse the embedded cc-results channel (restored — dashboard skill
  // copies get their results injected here by the chat Claude when it edits the artifact).
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
  // CLIPBOARD EDITION: no API config (mcp/webSearch/fileKind gone). needsFile skills keep a
  // flag only to show the "chat will ask you to attach…" chip; the artifact never touches files.
  // BOARD UPDATE: the three original chat-only skills (morning-routine, bank-analysis,
  // instagram) are REMOVED from this board — this dashboard build only makes sense paired
  // with the -cc skills' delivery step, so only those three ship here now.
  const SEED = [
    { id:"morning-routine-dash", section:"home", name:"Morning Routine · Dashboard",
      desc:"Same brief, but the result lands in the panel on this board instead of the chat.",
      tag:"morning-briefing-cc", dash:true, fav:true },
    { id:"bank-analysis-dash", section:"finance", name:"Bank Statement Analysis · Dashboard",
      desc:"Same analysis — the chat asks for your PDF, then drops the dashboard into the panel here.",
      tag:"bank-statement-analyzer-cc", needsFile:true, uploadLabel:"PDF", dash:true, fav:true },
    { id:"instagram-dash", section:"marketing", name:"Instagram Dashboard · Dashboard",
      desc:"Same audit — the chat asks for your export ZIP, then drops the report into the panel here.",
      tag:"instagram-export-dashboard-cc", needsFile:true, uploadLabel:"ZIP", dash:true, fav:false }
  ];
  // Skills to actively strip from any PREVIOUSLY SAVED state (returning users had the
  // three chat-only originals persisted before this board update removed them from SEED).
  const REMOVED_IDS = ["morning-routine", "bank-analysis", "instagram"];

  // CLIPBOARD EDITION: the run engine. Was: direct Anthropic API calls. Now: ▶ Run copies a
  // premade prompt to the clipboard; the user pastes it into their Claude chat, and the chat
  // runs the REAL skill (with connectors, file attachments, everything). Upload skills'
  // prompts tell the chat to ask for the attachment first and wait — no files in the artifact.
  // BOARD UPDATE: prompts for the three retired chat-only skills (morning-routine,
  // bank-analysis, instagram) are removed along with their cards — only the -cc
  // dashboard-delivery prompts remain below.
  const SKILL_PROMPTS = {
    // DASHBOARD-DELIVERY prompts: now just point at the real morning-briefing-cc /
    // bank-statement-analyzer-cc / instagram-export-dashboard-cc skills — those skill
    // files already contain the fragment-rendering rules AND the cc-results delivery
    // steps (find the control center artifact, str_replace its cc-results block under
    // the matching key, cap at 10 runs, reply with one line). No need to restate any
    // of that here; the prompt only needs to name the skill and the run key.
    "morning-routine-dash":
`Use the morning-briefing-cc skill to run my /morning-routine and deliver it to my Business Control Center dashboard artifact under the "morning-routine-dash" key.`,
    "bank-analysis-dash":
`I want to run /bank-analysis-dash with the bank-statement-analyzer-cc skill. First ask me to attach my bank statement PDF to this chat and wait for my upload — then analyze it and deliver the dashboard to my Business Control Center artifact under the "bank-analysis-dash" key.`,
    "instagram-dash":
`I want to run /instagram-dash with the instagram-export-dashboard-cc skill. First ask me to attach my Instagram personal data export (.zip) and wait for my upload — then run the audit and deliver it to my Business Control Center artifact under the "instagram-dash" key.`
  };

  // CLIPBOARD EDITION: persistence works in BOTH contexts — window.storage inside a claude.ai
  // artifact, localStorage when the file is opened directly in a normal browser (which the
  // clipboard edition supports, since Run only needs a clipboard). Falls back to memory.
  async function load(){
    try{
      if(window.storage){
        const r = await window.storage.get(LS_KEY);          // throws if the key doesn't exist yet
        if(r && r.value){ const d = JSON.parse(r.value); if(Array.isArray(d.skills)) return d; }
      } else {
        const raw = localStorage.getItem(LS_KEY);            // standalone-browser fallback
        if(raw){ const d = JSON.parse(raw); if(Array.isArray(d.skills)) return d; }
      }
    }catch(e){}
    return { skills: SEED.map(s=>({...s})), active:"home" };
  }
  function save(){
    try{
      if(window.storage) window.storage.set(LS_KEY, JSON.stringify(state)); // fire-and-forget async write
      else localStorage.setItem(LS_KEY, JSON.stringify(state));             // standalone-browser fallback
    }catch(e){}
  }

  let state = { skills: SEED.map(s=>({...s})), active:"home" };  // temporary state until boot() loads the saved one

  // Backfill seed-managed fields onto skills persisted before those fields existed.
  // CHAT EDITION: field list swapped to the new run config (mcp/webSearch/fileKind) —
  // otherwise state saved by the Cowork build would run without connectors or uploads.
  function migrate(){
    let changed = false;
    // BOARD UPDATE: drop the three retired chat-only originals from any saved state —
    // returning users had them persisted before this board removed them from SEED.
    const before = state.skills.length;
    state.skills = state.skills.filter(s => !REMOVED_IDS.includes(s.id));
    if(state.skills.length !== before) changed = true;
    for(const seed of SEED){
      const s = state.skills.find(x => x.id === seed.id);
      if(!s){ state.skills.push({...seed}); changed = true; continue; }   // NEW: add seed skills missing from saved state (the dashboard copies)
      ["needsFile","uploadLabel","tag","dash"].forEach(k=>{ if(seed[k] !== undefined && s[k] === undefined){ s[k] = seed[k]; changed = true; } }); // dash flag added
      // strip dead wiring from BOTH previous editions (Cowork tasks/Drive + API mcp/webSearch)
      ["taskId","driveTitle","driveFolderId","driveInput","mcp","webSearch","fileKind","accept"].forEach(k=>{ if(k in s){ delete s[k]; changed = true; } });
    }
    if(changed) save();
  }

  let editingId = null;

  const main   = document.getElementById("main");
  const nav    = document.getElementById("nav");
  const favRail= document.getElementById("favRail");
  const overlay= document.getElementById("overlay");
  const toastEl= document.getElementById("toast");

  function monogram(name){ return (name.trim()[0]||"•").toUpperCase(); }
  function esc(s){ return (s||"").replace(/[&<>"]/g,c=>({"&":"&amp;","<":"&lt;",">":"&gt;",'"':"&quot;"}[c])); }

  // CLIPBOARD EDITION: renderMarkdown() removed (only result panels used it).

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

    // results zone (restored): panels ONLY for the dashboard copies (dash:true) —
    // the original skills' results still render in the chat conversation.
    const rw = document.getElementById("resultsWrap");
    const dashSkills = skills.filter(s=>s.dash);
    if(dashSkills.length){
      rw.innerHTML = `<div class="eyebrow">Dashboard results</div>`;
      dashSkills.forEach(s=> rw.appendChild(resultCard(s)) );
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
      ${s.needsFile ? `<div class="file-note">📎 The chat will ask you to attach your ${esc(s.uploadLabel || "file")}</div>` : ""}
      <div class="status" data-status></div>
      <div class="card-actions">
        <button class="run-btn" title="Copies this skill's run prompt — paste it into your Claude chat">▶ Copy run</button>
        <button class="icon-btn" title="Edit" data-edit>✎</button>
        <button class="icon-btn" title="Remove" data-del>×</button>
      </div>
    `;
    el.querySelector(".star-btn").onclick = ()=>{ s.fav=!s.fav; save(); render(); };
    el.querySelector(".run-btn").onclick  = ()=> runSkill(s.id, el);
    // CLIPBOARD EDITION: upload input wiring removed — attachments happen in the chat.
    el.querySelector("[data-edit]").onclick = ()=> openModal(s.section, s.id);
    // FIX: window.confirm() is blocked (silently no-ops) inside the sandboxed artifact
    // iframe, which is why × previously did nothing. Replaced with an in-DOM two-click
    // confirm: first click arms it ("Sure? ×") for 3s, a second click within that window
    // deletes; clicking anything else, or letting it time out, reverts with no side effect.
    const delBtn = el.querySelector("[data-del]");
    let armed = false, armTimer = null;
    delBtn.onclick = ()=>{
      if(!armed){
        armed = true;
        delBtn.textContent = "Sure? ×";
        delBtn.title = "Click again to remove";
        clearTimeout(armTimer);
        armTimer = setTimeout(()=>{ armed = false; delBtn.textContent = "×"; delBtn.title = "Remove"; }, 3000);
        return;
      }
      clearTimeout(armTimer);
      state.skills = state.skills.filter(x=>x.id!==s.id); save(); render();
      toast(`Removed <b>${esc(s.name)}</b> from your control center. (The skill itself isn't deleted.)`);
    };
    return el;
  }

  // ----- dashboard result panels (restored) -----
  // A dashboard skill's runs come from TWO sources merged newest-first:
  //   (a) the embedded cc-results channel — what the chat Claude injected into this artifact,
  //   (b) window.storage — where boot() copies every cc-results run so history survives
  //       future artifact edits (each edit ships only its own cc-results snapshot).
  const RUNS = {};               // skillId -> [ {id, when, ts, html?} ] newest first (id = "ts")
  const CUR  = {};               // skillId -> selected run id
  const HTMLC = {};              // "skillId:ts" -> decoded html cache

  function ccRunsOf(id){
    const e = RESULTS[id];
    const arr = e && Array.isArray(e.runs) ? e.runs : (e && e.b64 ? [e] : []);  // tolerate old single-run form
    return arr.filter(r=>r && r.b64).map(r=>({ ts:r.ts||0, when:r.when||"run", b64:r.b64 }));
  }

  // Merge cc-results with storage-indexed runs, dedupe by ts, sort newest first, cap 10.
  async function listRuns(s){
    const merged = {};
    for(const r of ccRunsOf(s.id)) merged[r.ts] = { id:String(r.ts), ts:r.ts, when:r.when, b64:r.b64 };
    if(window.storage){
      try{
        const r = await window.storage.get("ccidx:" + s.id);
        for(const e of (r && r.value ? JSON.parse(r.value) : [])){ if(!merged[e.ts]) merged[e.ts] = { id:String(e.ts), ts:e.ts, when:e.when }; }
      }catch(e){}                                                        // missing index key throws → storage has none yet
    }
    return Object.values(merged).sort((a,b)=> b.ts - a.ts).slice(0,10);
  }

  async function loadRunHtml(s, run){
    const ck = s.id + ":" + run.ts;
    if(HTMLC[ck]) return HTMLC[ck];
    let html = run.b64 ? b64decode(run.b64) : null;                      // embedded runs decode directly
    if(!html && window.storage){                                         // storage-only runs (older than this artifact edit)
      try{ const r = await window.storage.get("ccrun:" + s.id + ":" + run.ts); html = r && r.value ? r.value : null; }catch(e){}
    }
    if(html) HTMLC[ck] = html;
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
  async function showRun(s, runId){
    const { canvas, whenEl } = panelEls(s);
    const run = (RUNS[s.id]||[]).find(r=>r.id===runId);
    if(!canvas || !run) return;
    CUR[s.id] = runId;
    if(whenEl) whenEl.textContent = run.when;
    const html = await loadRunHtml(s, run);
    if(CUR[s.id] !== runId) return;                                      // selection changed while loading — drop
    canvas.innerHTML = html || `<div class="result-empty">Couldn't load this run.</div>`;
  }
  async function loadSkillRuns(s){
    const { canvas, whenEl } = panelEls(s);
    const runs = await listRuns(s);
    RUNS[s.id] = runs;
    if(!runs.length){ fillRunSelect(s); if(canvas) canvas.innerHTML = `<div class="result-empty">No runs yet — <b>▶ Copy run</b>, paste into your Claude chat, and the result lands here.</div>`; if(whenEl) whenEl.textContent = "no runs yet"; return; }
    const chosen = (CUR[s.id] && runs.some(r=>r.id===CUR[s.id])) ? CUR[s.id] : runs[0].id;
    fillRunSelect(s);
    await showRun(s, chosen);
  }

  function resultCard(s){
    const wrap = document.createElement("div");
    wrap.className = "result-card";
    wrap.innerHTML = `
      <div class="rc-head">
        <span class="rc-name">${esc(s.name)}</span>
        <select class="run-select" title="Past runs" style="display:none"></select>
        <span class="rc-when">loading…</span>
      </div>
      <div class="result-body"><div class="result-canvas" id="result-${esc(s.id)}"></div></div>`;
    wrap.querySelector(".run-select").onchange = (e)=> showRun(s, e.target.value);
    setTimeout(()=> loadSkillRuns(s), 0);
    return wrap;
  }

  // On load, copy every embedded cc-results run into window.storage so history outlives
  // future artifact edits (which carry only their own cc-results snapshot).
  async function syncResultsToStorage(){
    if(!window.storage) return;
    for(const id of Object.keys(RESULTS)){
      const runs = ccRunsOf(id);
      if(!runs.length) continue;
      let idx = [];
      try{ const r = await window.storage.get("ccidx:" + id); idx = r && r.value ? JSON.parse(r.value) : []; }catch(e){}
      let changed = false;
      for(const run of runs){
        if(idx.some(e=>e.ts===run.ts)) continue;                         // already synced by a previous load
        try{
          await window.storage.set("ccrun:" + id + ":" + run.ts, b64decode(run.b64));
          idx.push({ ts:run.ts, when:run.when }); changed = true;
        }catch(e){}
      }
      if(changed){
        idx.sort((a,b)=> b.ts - a.ts);
        if(idx.length > 30){                                             // cap; delete overflow run keys
          for(const old of idx.slice(30)){ try{ await window.storage.delete("ccrun:" + id + ":" + old.ts); }catch(e){} }
          idx = idx.slice(0,30);
        }
        try{ await window.storage.set("ccidx:" + id, JSON.stringify(idx)); }catch(e){}
      }
    }
  }

  // ----- run a skill by copying its prompt to the clipboard (CLIPBOARD EDITION) -----
  // Was: direct Anthropic API calls with a continuation loop. A file artifact has no bridge
  // to post into the chat (sendPrompt is widget-only), so ▶ copies the premade prompt and
  // the user pastes it into their Claude chat — the chat then runs the REAL skill.
  function promptOf(s){ return s.prompt || SKILL_PROMPTS[s.id] || null; }   // custom skills carry their own prompt

  let CAN_RUN = false;
  function detectRunner(){
    CAN_RUN = !!(navigator.clipboard || document.execCommand);              // clipboard is the only requirement now
    const pill = document.getElementById("bridgePill");
    if(!pill) return;
    pill.className = "bridge-pill on";
    pill.textContent = "Clipboard mode";
    pill.title = "▶ copies a skill's run prompt; paste it into your Claude chat to run the real skill.";
    pill.onclick = ()=> toast("▶ copies the skill's premade prompt. Paste it into your Claude chat — the chat runs the actual skill (and will ask for any file attachments).");
  }

  // Copy with fallback: navigator.clipboard needs a secure context; the textarea+execCommand
  // path covers plain file:// opens and older webviews.
  async function copyText(t){
    try{ await navigator.clipboard.writeText(t); return true; }catch(e){}
    try{
      const ta = document.createElement("textarea");
      ta.value = t; ta.style.position = "fixed"; ta.style.opacity = "0";
      document.body.appendChild(ta); ta.select();
      const ok = document.execCommand("copy");
      document.body.removeChild(ta);
      return ok;
    }catch(e){ return false; }
  }

  function runSkill(id, cardEl){
    const s = state.skills.find(x=>x.id===id);
    if(!s) return;
    const statusEl = cardEl ? cardEl.querySelector("[data-status]") : null;
    const setStatus = (html)=>{ if(statusEl) statusEl.innerHTML = html; };
    const prompt = promptOf(s);

    if(!prompt){
      setStatus(`<span class="dot"></span> No prompt wired`);
      toast(`<b>${esc(s.name)}</b> has no run prompt yet — edit it and add one.`);
      return;
    }
    copyText(prompt).then((ok)=>{
      if(!ok) throw new Error("clipboard unavailable");
      s.lastRun = Date.now(); save();
      setStatus(`<span class="dot done"></span> Copied ✓ — paste into your Claude chat`);
      toast(`<b>${esc(s.name)}</b> prompt copied — paste it into the chat${s.needsFile ? `; it will ask you to attach your ${esc(s.uploadLabel||"file")}` : ""}.`);
    }).catch(()=>{
      setStatus(`<span class="dot"></span> Copy failed`);
      toast(`Couldn't reach the clipboard — select and copy the prompt from the Edit (✎) dialog instead.`);
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
    document.getElementById("fTask").value    = s ? (s.prompt||"") : "";   // was s.taskId — modal now edits the run prompt
    overlay.classList.add("show");
    setTimeout(()=>document.getElementById("fName").focus(), 50);
  }
  function closeModal(){ overlay.classList.remove("show"); editingId=null; }

  function saveSkill(){
    const name = document.getElementById("fName").value.trim();
    if(!name){ toast("Give your skill a name first."); return; }
    const desc = document.getElementById("fDesc").value.trim() || "Custom skill.";
    const section = document.getElementById("fSection").value;
    const prompt = document.getElementById("fTask").value.trim();          // was taskId — now the custom run prompt

    if(editingId){
      const s = state.skills.find(x=>x.id===editingId);
      Object.assign(s, { name, desc, section, prompt });
    } else {
      const baseId = name.toLowerCase().replace(/[^a-z0-9]+/g,"-").replace(/^-|-$/g,"") || ("skill-"+Date.now());
      let uid = baseId, n=2; while(state.skills.some(x=>x.id===uid)){ uid = baseId+"-"+(n++); }
      state.skills.push({ id:uid, section, name, desc, tag:uid, prompt, mcp:[], webSearch:false, fav:false }); // new fields default off
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

  // CLIPBOARD EDITION: input loading (PDF/ZIP) and the storage run-history machinery are
  // removed — files are attached in the chat that runs the skill, and results render there.

  // CLIPBOARD EDITION: async boot (load() now awaits window.storage before first render).
  (async function boot(){
    state = await load();
    migrate();
    detectRunner();
    await syncResultsToStorage();   // NEW: persist embedded runs into storage before first render
    render();
  })();
})();
</script>
</body>
</html>
```

---

## Appendix B — Dashboard skill template (the shape a skill needs to plug into this board)

Any skill can drive a card on this dashboard if it does three things: **do its work**, **render the result as a panel-safe fragment**, and **deliver that fragment into the artifact's `cc-results` channel**. This appendix defines exactly those three parts — it is a contract and a skeleton, not a tutorial. The reference board's three skills (`morning-briefing-cc`, `bank-statement-analyzer-cc`, `instagram-export-dashboard-cc`) all follow this shape.

### B.1 — SKILL.md skeleton

````markdown
---
name: <your-skill>-cc
description: <What it does> and deliver the result into a panel on the user's
  Business Control Center dashboard artifact, instead of replying with it in chat.
  Use this whenever the user runs the "<Card Name>" card on their control center,
  pastes its copied run prompt, or asks for <the task> to land on the
  dashboard/control center rather than in the conversation.
---

# <Skill Name> — Control Center Edition

## Workflow

### 1..n-2. Do the actual work
(Whatever the skill does: pull from connectors, read an uploaded file from
/mnt/user-data/uploads/, run scripts, compute. If the card is an upload card,
the copied prompt already told the chat to ask for the attachment first and
wait — this skill just reads it the normal way once it's there.)

### n-1. Render as a dashboard fragment
Render ONE self-contained HTML fragment (see B.2 — the fragment contract).

### n. Deliver to the dashboard
(See B.3 — the delivery step. Use this text nearly verbatim, with your card's key.)
````

The `description` frontmatter is what makes the card's copied prompt reliably trigger this skill — name the card and the "lands on the dashboard, not in chat" intent explicitly.

### B.2 — The fragment contract (what step n-1 must output)

Every fragment delivered to a panel must satisfy ALL of these — they exist because the dashboard injects fragments via `innerHTML` inside a sandboxed page (Section 3, rules 3–4 and 7):

- ONE wrapper div with a unique scoped class: `<div class="cc-<key>"> ... </div>`.
- Exactly ONE `<style>` block inside it, with **every rule prefixed** by `.cc-<key>` so nothing leaks into the shell.
- NO `<html>`, `<head>`, or `<body>` tags. NO `<script>` anywhere. NO external resources (fonts, images, stylesheets) — inline everything; system/serif font stacks only.
- Charts must be **static**: inline SVG, or divs whose `style="height:H%"` / `style="width:P%"` values are computed **before** delivery (server-side / in a script), never left for client JS to fill in — injected scripts never run, so a JS-drawn chart renders blank.
- Brand palette hard-coded (a fragment can't see the shell's CSS variables) — use the hexes from Section 5 **as customized to the user's brand; never the shipped reference placeholders** (re-theming the artifact's `:root` does nothing for skill output, so an off-brand skill renders off-brand results forever). Keep green `#4f9d7c` for positive/pass and red `#b3433f` for negative/fail regardless of brand.
- `width:100%; box-sizing:border-box` on the wrapper so it fills its panel. Keep it compact.

### B.3 — The delivery step (what step n must do)

> Find the Business Control Center HTML artifact for this conversation (check recent turns for its file path; ask the user rather than guessing).
> 1. `view` the file and locate the `<script id="cc-results" type="application/json">` block.
> 2. Parse its JSON. Under the key **`<card-key>`** (must match the card's `id` in the artifact's `SEED`, e.g. `bank-analysis-dash`), find or create an object shaped `{"runs": [...]}`.
> 3. Base64-encode the fragment and **prepend** a new entry: `{"when": "<short human label, e.g. 'Jul 13 · 8:02am'>", "ts": <current epoch ms>, "b64": "<the base64>"}`.
> 4. Cap the array at 10 entries (drop the oldest).
> 5. Write the updated JSON back into that `<script>` tag with `str_replace` — change **nothing else** in the file — then `present_files` the artifact again.
> 6. Reply in chat with only a **one-line confirmation** (optionally plus one headline number); the result lives in the panel, not the chat.

If the artifact can't be found or its `cc-results` block is missing/malformed, say so plainly and offer the result inline instead of guessing at a fix.

### B.4 — Integration checklist (per card ↔ skill pair)

- [ ] Skill saved to the user's profile (packaged `.skill` file → **Save skill** clicked).
- [ ] The card's `SKILL_PROMPTS` entry names the skill and (for upload cards) opens with "first ask me to attach … and wait for my upload."
- [ ] The skill's delivery key == the card's `id` in `SEED`.
- [ ] Fragment wrapper class is unique on the board (`.cc-<key>`).
- [ ] A test run lands in the panel and the chat reply is one line, not the full result.

---

## Appendix C — Example result fragments (what a skill should output)

**Reference only — NEVER build these into the dashboard.** These fragments exist purely to show the shape a delivered result should take (scoped div, one style block, no JS, static charts). Do **not** seed them into the artifact's `cc-results` channel, do not pre-populate any panel with them, and do not ship them as demo runs — the dashboard must be built **blank** (`cc-results` = `{}`), and every panel starts at its "No runs yet" empty state until a real skill run delivers a real result. Note also that these examples predate the current brand palette (they carry older hexes); when writing a new skill's renderer, take the *structure* from here but the *colors* from Section 5.

Each is a self-contained, scoped, no-JS HTML fragment. Use them as the visual/structure model; the skill fills in real data and matches the brand palette.

### C.1 — Morning brief (list/schedule/quote layout)
```html
<div class="cc-morning"><style>
.cc-morning{font-family:-apple-system,'Segoe UI',sans-serif;color:#2A1B45;background:#FAF8FE;border:1px solid #E7DEF5;border-radius:14px;padding:20px 22px;width:100%;box-sizing:border-box;line-height:1.5}
.cc-morning .top{display:flex;align-items:center;gap:12px;border-bottom:1px solid #E7DEF5;padding-bottom:13px;margin-bottom:6px}
.cc-morning h1{font-family:Georgia,'Times New Roman',serif;color:#6D28D9;font-size:23px;margin:0;font-weight:600;line-height:1}
.cc-morning .date{font-size:12px;color:#7A689B;margin-top:4px}
.cc-morning .lbl{font-family:ui-monospace,'SFMono-Regular',Menlo,monospace;font-size:10px;letter-spacing:.18em;text-transform:uppercase;color:#CB3A9D;display:flex;align-items:center;gap:8px;margin:18px 0 9px}
.cc-morning .lbl::before{content:"";width:18px;height:1px;background:#9333EA}
.cc-morning ul{list-style:none;margin:0;padding:0;display:flex;flex-direction:column;gap:9px}
.cc-morning li{padding-left:16px;position:relative}
.cc-morning li::before{content:"";position:absolute;left:0;top:8px;width:5px;height:5px;border-radius:50%;background:#CB3A9D}
.cc-morning b{color:#6D28D9;font-weight:600}
.cc-morning .sched{display:flex;gap:12px;padding:7px 0;border-bottom:1px solid #F3ECFB}
.cc-morning .sched:last-of-type{border-bottom:none}
.cc-morning .sched .t{font-family:ui-monospace,Menlo,monospace;font-size:13px;color:#6D28D9;min-width:66px;font-weight:600}
.cc-morning blockquote{margin:0;padding:13px 16px;background:#F3ECFB;border-left:3px solid #CB3A9D;border-radius:0 9px 9px 0;font-family:Georgia,serif;font-style:italic;color:#6D28D9;font-size:15px}
.cc-morning .foot{margin-top:18px;border-top:1px dashed #E7DEF5;padding-top:11px;font-size:11px;color:#7A689B;font-style:italic}
</style>
<div class="top">
<svg width="34" height="34" viewBox="0 0 24 24" fill="none" stroke="#CB3A9D" stroke-width="1.6" stroke-linecap="round"><circle cx="12" cy="12" r="4" fill="#F3ECFB"></circle><line x1="12" y1="2" x2="12" y2="4.5"></line><line x1="12" y1="19.5" x2="12" y2="22"></line><line x1="2" y1="12" x2="4.5" y2="12"></line><line x1="19.5" y1="12" x2="22" y2="12"></line><line x1="4.9" y1="4.9" x2="6.7" y2="6.7"></line><line x1="17.3" y1="17.3" x2="19.1" y2="19.1"></line><line x1="4.9" y1="19.1" x2="6.7" y2="17.3"></line><line x1="17.3" y1="6.7" x2="19.1" y2="4.9"></line></svg>
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
<div class="sched"><span class="t">8:30am</span><span>Feranse Daily SCRUM</span></div>
<div class="sched"><span class="t">8:45am</span><span>Daily Scrum, Exec Inner Circle · with Eric</span></div>
<div class="sched"><span class="t">9:30am</span><span>Feranse OST (Daily SCRUM) · with Ali</span></div>
<div class="sched"><span class="t">10:00am</span><span>VAL x Feranse · with Alessandro Labeque</span></div>
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
.cc-finance{font-family:-apple-system,'Segoe UI',sans-serif;color:#2A1B45;background:#FAF8FE;border:1px solid #E7DEF5;border-radius:14px;padding:20px 22px;width:100%;box-sizing:border-box;line-height:1.5}
.cc-finance .fh1{font-family:Georgia,'Times New Roman',serif;color:#6D28D9;font-size:21px;margin:0;font-weight:600}
.cc-finance .fsub{color:#7A689B;font-size:13px;margin-top:3px}
.cc-finance .kgrid{display:grid;grid-template-columns:repeat(auto-fit,minmax(140px,1fr));gap:12px;margin:18px 0}
.cc-finance .kpi{background:#fff;border:1px solid #E7DEF5;border-radius:12px;padding:14px}
.cc-finance .kl{color:#7A689B;font-family:ui-monospace,Menlo,monospace;font-size:10px;text-transform:uppercase;letter-spacing:.1em}
.cc-finance .kv{font-size:21px;font-weight:650;margin-top:5px;color:#2A1B45}
.cc-finance .pos{color:#4f9d7c} .cc-finance .neg{color:#b3433f}
.cc-finance .fcard{background:#fff;border:1px solid #E7DEF5;border-radius:12px;padding:18px;margin:14px 0}
.cc-finance .fcard h2{margin:0 0 12px;font-family:ui-monospace,Menlo,monospace;font-size:11px;text-transform:uppercase;letter-spacing:.12em;color:#CB3A9D}
.cc-finance .frow{display:grid;grid-template-columns:1.1fr 1fr;gap:16px}
@media(max-width:680px){.cc-finance .frow{grid-template-columns:1fr}}
.cc-finance .donut{display:flex;align-items:center;gap:16px;flex-wrap:wrap;justify-content:center}
.cc-finance .leg{flex:1;min-width:210px}
.cc-finance .lg{display:flex;align-items:center;gap:8px;font-size:12.5px;margin:5px 0}
.cc-finance .dot{width:11px;height:11px;border-radius:3px;flex:0 0 auto}
.cc-finance .ll{flex:1;min-width:110px} .cc-finance .lv{color:#7A689B;font-variant-numeric:tabular-nums}
.cc-finance .lv em{color:#A99FBC;font-style:normal;margin-left:4px}
.cc-finance table{width:100%;border-collapse:collapse;font-size:12.5px}
.cc-finance th,.cc-finance td{text-align:left;padding:7px 9px;border-bottom:1px solid #F3ECFB}
.cc-finance th{color:#7A689B;font-family:ui-monospace,Menlo,monospace;font-weight:600;font-size:10px;text-transform:uppercase;letter-spacing:.08em}
.cc-finance td.num,.cc-finance th.num{text-align:right;font-variant-numeric:tabular-nums}
.cc-finance .bars{display:flex;gap:16px;font-size:11.5px;color:#7A689B;margin-top:8px}
.cc-finance .bars span::before{content:"";display:inline-block;width:10px;height:10px;border-radius:2px;margin-right:5px;vertical-align:middle}
.cc-finance .bin::before{background:#4f9d7c} .cc-finance .bout::before{background:#6D28D9}
.cc-finance .muted{color:#A99FBC;text-align:center}
.cc-finance .ffoot{color:#A99FBC;font-size:11px;margin-top:16px;text-align:center;font-style:italic}
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
<svg viewBox="0 0 560 240" width="100%" height="240"><line x1="48" y1="212" x2="548" y2="212" stroke="#F3ECFB"/><text x="42" y="215" text-anchor="end" font-size="10" fill="#A99FBC">$0</text><line x1="48" y1="162" x2="548" y2="162" stroke="#F3ECFB"/><text x="42" y="165" text-anchor="end" font-size="10" fill="#A99FBC">$2,035</text><line x1="48" y1="112" x2="548" y2="112" stroke="#F3ECFB"/><text x="42" y="115" text-anchor="end" font-size="10" fill="#A99FBC">$4,069</text><line x1="48" y1="62" x2="548" y2="62" stroke="#F3ECFB"/><text x="42" y="65" text-anchor="end" font-size="10" fill="#A99FBC">$6,104</text><line x1="48" y1="12" x2="548" y2="12" stroke="#F3ECFB"/><text x="42" y="15" text-anchor="end" font-size="10" fill="#A99FBC">$8,138</text><rect x="270" y="208.8" width="26" height="3.2" fill="#4f9d7c" rx="2"/><rect x="300" y="12" width="26" height="200" fill="#6D28D9" rx="2"/><text x="298" y="230" text-anchor="middle" font-size="10" fill="#7A689B">2026-05</text></svg>
<div class="bars"><span class="bin">Money in</span><span class="bout">Money out</span></div>
</div>
<div class="frow">
  <div class="fcard"><h2>Expenses by category</h2>
  <div class="donut">
    <svg viewBox="0 0 220 220" width="200" height="200"><g transform="rotate(-90 110 110)"><circle cx="110" cy="110" r="93" fill="none" stroke="#6D28D9" stroke-width="34" stroke-dasharray="157.173 427.163" stroke-dashoffset="0"/><circle cx="110" cy="110" r="93" fill="none" stroke="#CB3A9D" stroke-width="34" stroke-dasharray="125.581 458.756" stroke-dashoffset="-157.173"/><circle cx="110" cy="110" r="93" fill="none" stroke="#9333EA" stroke-width="34" stroke-dasharray="93.034 491.303" stroke-dashoffset="-282.754"/><circle cx="110" cy="110" r="93" fill="none" stroke="#7A689B" stroke-width="34" stroke-dasharray="72.146 512.190" stroke-dashoffset="-375.788"/><circle cx="110" cy="110" r="93" fill="none" stroke="#E879C0" stroke-width="34" stroke-dasharray="46.671 537.665" stroke-dashoffset="-447.934"/><circle cx="110" cy="110" r="93" fill="none" stroke="#A855F7" stroke-width="34" stroke-dasharray="32.311 552.025" stroke-dashoffset="-494.605"/><circle cx="110" cy="110" r="93" fill="none" stroke="#C026D3" stroke-width="34" stroke-dasharray="24.190 560.146" stroke-dashoffset="-526.916"/><circle cx="110" cy="110" r="93" fill="none" stroke="#D8A7E8" stroke-width="34" stroke-dasharray="13.680 570.657" stroke-dashoffset="-551.106"/><circle cx="110" cy="110" r="93" fill="none" stroke="#8B5CF6" stroke-width="34" stroke-dasharray="12.363 571.973" stroke-dashoffset="-564.786"/><circle cx="110" cy="110" r="93" fill="none" stroke="#BE3FA0" stroke-width="34" stroke-dasharray="7.187 577.149" stroke-dashoffset="-577.149"/></g></svg>
    <div class="leg">
      <div class="lg"><span class="dot" style="background:#6D28D9"></span><span class="ll">Marketing &amp; advertising</span><span class="lv">$2,188.99 <em>27%</em></span></div>
      <div class="lg"><span class="dot" style="background:#CB3A9D"></span><span class="ll">Equipment &amp; hardware</span><span class="lv">$1,748.99 <em>21%</em></span></div>
      <div class="lg"><span class="dot" style="background:#9333EA"></span><span class="ll">Travel</span><span class="lv">$1,295.70 <em>16%</em></span></div>
      <div class="lg"><span class="dot" style="background:#7A689B"></span><span class="ll">Software &amp; SaaS</span><span class="lv">$1,004.80 <em>12%</em></span></div>
      <div class="lg"><span class="dot" style="background:#E879C0"></span><span class="ll">Office &amp; rent</span><span class="lv">$650.00 <em>8%</em></span></div>
      <div class="lg"><span class="dot" style="background:#A855F7"></span><span class="ll">Professional services</span><span class="lv">$450.00 <em>6%</em></span></div>
      <div class="lg"><span class="dot" style="background:#C026D3"></span><span class="ll">Meals &amp; entertainment</span><span class="lv">$336.90 <em>4%</em></span></div>
      <div class="lg"><span class="dot" style="background:#D8A7E8"></span><span class="ll">Supplies &amp; materials</span><span class="lv">$190.52 <em>2%</em></span></div>
      <div class="lg"><span class="dot" style="background:#8B5CF6"></span><span class="ll">Bank fees &amp; interest</span><span class="lv">$172.18 <em>2%</em></span></div>
      <div class="lg"><span class="dot" style="background:#BE3FA0"></span><span class="ll">Shipping &amp; postage</span><span class="lv">$100.10 <em>1%</em></span></div>
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
.cc-ig{font-family:-apple-system,'Segoe UI',sans-serif;color:#2A1B45;background:#FAF8FE;border:1px solid #E7DEF5;border-radius:14px;padding:22px 24px;width:100%;box-sizing:border-box;line-height:1.5}
.cc-ig .eyebrow{font-family:ui-monospace,Menlo,monospace;font-size:11px;letter-spacing:.16em;text-transform:uppercase;color:#CB3A9D;margin:0 0 10px}
.cc-ig h1{font-family:Georgia,'Times New Roman',serif;font-weight:600;font-size:26px;line-height:1.12;margin:0 0 12px;color:#6D28D9}
.cc-ig h1 .at{color:#9333EA;font-style:italic}
.cc-ig .lede{font-size:15px;color:#7A689B;max-width:64ch;margin:0}.cc-ig .lede b{color:#2A1B45;font-weight:600}
.cc-ig .kpis{display:grid;grid-template-columns:repeat(3,1fr);gap:10px;margin:20px 0 8px}
.cc-ig .kpi{background:#fff;border:1px solid #E7DEF5;border-radius:11px;padding:14px}
.cc-ig .kpi .v{font-family:ui-monospace,Menlo,monospace;font-weight:700;font-size:21px;color:#2A1B45}
.cc-ig .kpi .v.fail{color:#b3433f}.cc-ig .kpi .v.gold{color:#9333EA}
.cc-ig .kpi .l{font-size:11px;color:#7A689B;margin-top:3px;line-height:1.3}
.cc-ig section{margin-top:30px}
.cc-ig .sec-head{display:flex;align-items:baseline;gap:12px;margin-bottom:4px}
.cc-ig .sec-num{font-family:ui-monospace,Menlo,monospace;font-size:12px;color:#9333EA;font-weight:600}
.cc-ig h2{font-family:Georgia,serif;font-size:20px;font-weight:600;margin:0;color:#6D28D9}
.cc-ig .sec-sub{color:#7A689B;font-size:13.5px;margin:4px 0 16px;max-width:72ch}
.cc-ig .card{background:#fff;border:1px solid #E7DEF5;border-radius:12px;padding:18px 20px}
.cc-ig .grid2{display:grid;grid-template-columns:1fr 1fr;gap:14px}
.cc-ig .card h3{margin:0 0 3px;font-size:14px;font-family:ui-monospace,Menlo,monospace;font-weight:600;color:#6D28D9}
.cc-ig .cap{color:#A99FBC;font-size:11.5px;margin:0 0 16px;font-family:ui-monospace,Menlo,monospace}
.cc-ig .bars{display:flex;align-items:flex-end;gap:5px;height:160px}.cc-ig .bars.tall{height:190px}
.cc-ig .bar{flex:1;display:flex;flex-direction:column;justify-content:flex-end;align-items:center;gap:5px;min-width:0}
.cc-ig .bar .col{width:100%;background:#F3ECFB;border-radius:3px 3px 0 0;position:relative}
.cc-ig .bar .col.hi{background:linear-gradient(#9333EA,#5B21B6)}.cc-ig .bar .col.cy{background:linear-gradient(#CB3A9D,#9333EA)}
.cc-ig .bar .n{font-family:ui-monospace,Menlo,monospace;font-size:10px;color:#7A689B;height:12px}
.cc-ig .bar .x{font-family:ui-monospace,Menlo,monospace;font-size:9px;color:#A99FBC;white-space:nowrap}
.cc-ig .annot{position:absolute;top:-18px;left:50%;transform:translateX(-50%);background:#b3433f;color:#fff;font-family:ui-monospace,Menlo,monospace;font-weight:700;font-size:9px;padding:2px 6px;border-radius:5px}
.cc-ig .tests{display:flex;flex-direction:column;gap:9px}
.cc-ig .test{display:grid;grid-template-columns:74px 1fr auto;align-items:center;gap:14px;background:#fff;border:1px solid #E7DEF5;border-left-width:3px;border-radius:9px;padding:12px 15px}
.cc-ig .test.pass{border-left-color:#4f9d7c}.cc-ig .test.fail{border-left-color:#b3433f}.cc-ig .test.partial{border-left-color:#cda05c}
.cc-ig .badge{font-family:ui-monospace,Menlo,monospace;font-weight:700;font-size:10px;letter-spacing:.06em;text-align:center;padding:5px 0;border-radius:6px}
.cc-ig .badge.pass{background:rgba(79,157,124,.16);color:#3c7d62}.cc-ig .badge.fail{background:rgba(179,67,63,.14);color:#b3433f}.cc-ig .badge.partial{background:rgba(205,160,92,.18);color:#9a7330}
.cc-ig .tname{font-weight:600;font-size:14px;color:#2A1B45}.cc-ig .why{color:#7A689B;font-size:12px;margin-top:2px}
.cc-ig .metric{font-family:ui-monospace,Menlo,monospace;font-size:14px;font-weight:700;text-align:right}
.cc-ig .test.pass .metric{color:#3c7d62}.cc-ig .test.fail .metric{color:#b3433f}.cc-ig .test.partial .metric{color:#9a7330}
.cc-ig .pillar{margin-bottom:14px}.cc-ig .pillar-top{display:flex;justify-content:space-between;font-size:13px;margin-bottom:6px}
.cc-ig .pillar-top .pct{font-family:ui-monospace,Menlo,monospace;color:#7A689B}.cc-ig .sig{color:#9333EA;font-family:ui-monospace,Menlo,monospace;font-size:10px}
.cc-ig .track{height:9px;background:#F3ECFB;border-radius:6px;overflow:hidden}.cc-ig .fill{height:100%;border-radius:6px}
.cc-ig .ex{color:#A99FBC;font-size:11px;font-family:ui-monospace,Menlo,monospace;margin-top:5px}
.cc-ig .links{display:flex;flex-direction:column;gap:12px}
.cc-ig .lk{display:grid;grid-template-columns:1fr 40px 1fr;align-items:stretch;border:1px solid #E7DEF5;border-radius:12px;overflow:hidden}
.cc-ig .lk .from{background:#fff;padding:15px 17px;border-left:3px solid #b3433f}
.cc-ig .lk .from.passdrv{border-left-color:#4f9d7c}
.cc-ig .lk .arrow{display:flex;align-items:center;justify-content:center;background:#F3ECFB;color:#9333EA;font-family:ui-monospace,Menlo,monospace;font-size:17px}
.cc-ig .lk .to{background:#FBF7FE;padding:15px 17px;border-left:3px solid #9333EA}
.cc-ig .lk .tag{font-family:ui-monospace,Menlo,monospace;font-size:10px;letter-spacing:.1em;text-transform:uppercase;margin:0 0 6px}
.cc-ig .lk .from .tag{color:#b3433f}.cc-ig .lk .from.passdrv .tag{color:#4f9d7c}.cc-ig .lk .to .tag{color:#9333EA}
.cc-ig .lk h4{margin:0 0 5px;font-size:14px;font-weight:600;color:#2A1B45}.cc-ig .lk p{margin:0;font-size:12.5px;color:#7A689B}.cc-ig .lk p b{color:#2A1B45}
.cc-ig .cal{display:grid;grid-template-columns:repeat(7,1fr);gap:1px;background:#E7DEF5;border:1px solid #E7DEF5;border-radius:11px;overflow:hidden;margin-top:6px}
.cc-ig .cal .d{background:#fff;padding:11px 10px;min-height:108px}
.cc-ig .cal .dh{font-family:ui-monospace,Menlo,monospace;font-size:11px;color:#7A689B;margin-bottom:8px;display:flex;justify-content:space-between}
.cc-ig .cal .dh .t{color:#A99FBC}
.cc-ig .slot{font-size:11px;border-radius:6px;padding:6px 8px;margin-bottom:5px;line-height:1.3}
.cc-ig .slot.reel{background:rgba(147,51,234,.15);border:1px solid rgba(147,51,234,.35)}
.cc-ig .slot.story{background:rgba(203,58,157,.12);border:1px solid rgba(203,58,157,.3)}
.cc-ig .slot.rest{color:#A99FBC;font-family:ui-monospace,Menlo,monospace;font-size:10px}
.cc-ig .slot .k{font-family:ui-monospace,Menlo,monospace;font-size:9px;text-transform:uppercase;letter-spacing:.08em;display:block;opacity:.9}
.cc-ig .slot.reel .k{color:#5B21B6}.cc-ig .slot.story .k{color:#9333EA}
.cc-ig .note{background:#fff;border:1px solid #E7DEF5;border-radius:11px;padding:16px 18px;font-size:12.5px;color:#7A689B;margin-top:20px}.cc-ig .note b{color:#2A1B45}
.cc-ig .ffoot{margin-top:24px;border-top:1px dashed #E7DEF5;padding-top:16px;color:#A99FBC;font-family:ui-monospace,Menlo,monospace;font-size:11px;display:flex;justify-content:space-between;flex-wrap:wrap;gap:10px}
@media(max-width:680px){.cc-ig .kpis{grid-template-columns:repeat(2,1fr)}.cc-ig .grid2{grid-template-columns:1fr}.cc-ig .lk{grid-template-columns:1fr}.cc-ig .cal{grid-template-columns:1fr}}
</style><p class="eyebrow">Account Audit · Instagram Data Export · Jun 24 2026</p><h1>The memes were right.<br>The <span class="at">posting</span> failed the test cases.</h1><p class="lede">A University of Waterloo ECE meme account with genuinely strong content DNA — niche, named, share-ready humor — that never reached escape velocity. The export shows why: a launch-week sprint, then near-total silence. This is the autograder report, and the strategy to make it pass.</p><div class="kpis"><div class="kpi"><div class="v ">34</div><div class="l">posts ever published</div></div><div class="kpi"><div class="v fail">21</div><div class="l">followers reached</div></div><div class="kpi"><div class="v gold">79%</div><div class="l">of posts in peak month</div></div><div class="kpi"><div class="v fail">7.5 yrs</div><div class="l">since last post</div></div><div class="kpi"><div class="v ">97%</div><div class="l">static images · 3% video</div></div><div class="kpi"><div class="v fail">0%</div><div class="l">of posts use hashtags</div></div></div><section><div class="sec-head"><span class="sec-num">01</span><h2>The cadence</h2></div><p class="sec-sub">Posting volume over the account's life. The shape of this chart is usually the single most important fact about an account.</p><div class="card"><h3>Posts per period</h3><p class="cap">// 79% of posts in 2017-11; max gap 192d</p><div class="bars tall"><div class="bar"><div class="n">27</div><div class="col hi" style="height:100.0%"><div class="annot">peak</div></div><div class="x">Nov&#x27;17</div></div><div class="bar"><div class="n"></div><div class="col cy" style="height:0.0%"></div><div class="x">Dec</div></div><div class="bar"><div class="n"></div><div class="col cy" style="height:0.0%"></div><div class="x">Jan&#x27;18</div></div><div class="bar"><div class="n"></div><div class="col cy" style="height:0.0%"></div><div class="x">Feb</div></div><div class="bar"><div class="n"></div><div class="col cy" style="height:0.0%"></div><div class="x">Mar</div></div><div class="bar"><div class="n"></div><div class="col cy" style="height:0.0%"></div><div class="x">Apr</div></div><div class="bar"><div class="n">4</div><div class="col cy" style="height:14.8%"></div><div class="x">May</div></div><div class="bar"><div class="n">2</div><div class="col cy" style="height:7.4%"></div><div class="x">Jun</div></div><div class="bar"><div class="n"></div><div class="col cy" style="height:0.0%"></div><div class="x">Jul</div></div><div class="bar"><div class="n"></div><div class="col cy" style="height:0.0%"></div><div class="x">Aug</div></div><div class="bar"><div class="n"></div><div class="col cy" style="height:0.0%"></div><div class="x">Sep</div></div><div class="bar"><div class="n"></div><div class="col cy" style="height:0.0%"></div><div class="x">Oct</div></div><div class="bar"><div class="n"></div><div class="col cy" style="height:0.0%"></div><div class="x">Nov</div></div><div class="bar"><div class="n">1</div><div class="col cy" style="height:3.7%"></div><div class="x">Dec</div></div></div></div></section><section><div class="sec-head"><span class="sec-num">02</span><h2>When it posts</h2></div><p class="sec-sub">Day-of-week and hour patterns (local time, America/Toronto). Use these to find — or fix — a consistent slot aligned to when the audience is actually scrolling.</p><div class="grid2"><div class="card"><h3>By day of week</h3><p class="cap">// busiest day: Sat</p><div class="bars"><div class="bar"><div class="n">2</div><div class="col cy" style="height:18.2%"></div><div class="x">Mon</div></div><div class="bar"><div class="n">5</div><div class="col cy" style="height:45.5%"></div><div class="x">Tue</div></div><div class="bar"><div class="n">4</div><div class="col cy" style="height:36.4%"></div><div class="x">Wed</div></div><div class="bar"><div class="n">1</div><div class="col cy" style="height:9.1%"></div><div class="x">Thu</div></div><div class="bar"><div class="n">4</div><div class="col cy" style="height:36.4%"></div><div class="x">Fri</div></div><div class="bar"><div class="n">11</div><div class="col cy" style="height:100.0%"></div><div class="x">Sat</div></div><div class="bar"><div class="n">7</div><div class="col cy" style="height:63.6%"></div><div class="x">Sun</div></div></div></div><div class="card"><h3>By hour (local)</h3><p class="cap">// hour-of-day distribution</p><div class="bars"><div class="bar"><div class="n"></div><div class="col cy" style="height:0.0%"></div><div class="x">0</div></div><div class="bar"><div class="n"></div><div class="col cy" style="height:0.0%"></div><div class="x">1</div></div><div class="bar"><div class="n">7</div><div class="col cy" style="height:100.0%"></div><div class="x">2</div></div><div class="bar"><div class="n"></div><div class="col cy" style="height:0.0%"></div><div class="x">3</div></div><div class="bar"><div class="n"></div><div class="col cy" style="height:0.0%"></div><div class="x">4</div></div><div class="bar"><div class="n"></div><div class="col cy" style="height:0.0%"></div><div class="x">5</div></div><div class="bar"><div class="n"></div><div class="col cy" style="height:0.0%"></div><div class="x">6</div></div><div class="bar"><div class="n"></div><div class="col cy" style="height:0.0%"></div><div class="x">7</div></div><div class="bar"><div class="n">1</div><div class="col cy" style="height:14.3%"></div><div class="x">8</div></div><div class="bar"><div class="n">2</div><div class="col cy" style="height:28.6%"></div><div class="x">9</div></div><div class="bar"><div class="n">2</div><div class="col cy" style="height:28.6%"></div><div class="x">10</div></div><div class="bar"><div class="n"></div><div class="col cy" style="height:0.0%"></div><div class="x">11</div></div><div class="bar"><div class="n">6</div><div class="col cy" style="height:85.7%"></div><div class="x">12</div></div><div class="bar"><div class="n">2</div><div class="col cy" style="height:28.6%"></div><div class="x">13</div></div><div class="bar"><div class="n">5</div><div class="col cy" style="height:71.4%"></div><div class="x">14</div></div><div class="bar"><div class="n"></div><div class="col cy" style="height:0.0%"></div><div class="x">15</div></div><div class="bar"><div class="n">1</div><div class="col cy" style="height:14.3%"></div><div class="x">16</div></div><div class="bar"><div class="n"></div><div class="col cy" style="height:0.0%"></div><div class="x">17</div></div><div class="bar"><div class="n">2</div><div class="col cy" style="height:28.6%"></div><div class="x">18</div></div><div class="bar"><div class="n">4</div><div class="col cy" style="height:57.1%"></div><div class="x">19</div></div><div class="bar"><div class="n"></div><div class="col cy" style="height:0.0%"></div><div class="x">20</div></div><div class="bar"><div class="n">1</div><div class="col cy" style="height:14.3%"></div><div class="x">21</div></div><div class="bar"><div class="n">1</div><div class="col cy" style="height:14.3%"></div><div class="x">22</div></div><div class="bar"><div class="n"></div><div class="col cy" style="height:0.0%"></div><div class="x">23</div></div></div></div></div></section><section><div class="sec-head"><span class="sec-num">03</span><h2>Fundamentals report</h2></div><p class="sec-sub">Each row is a growth fundamental scored against the export. The failing rows are exactly what the strategy fixes.</p><div class="tests"><div class="test pass"><span class="badge pass">PASS</span><div><div class="tname">Niche, named-entity humor</div><div class="why">Marmoset, named profs, WaterlooWorks, course codes — hyper-relatable to the target audience.</div></div><div class="metric">★ strong</div></div><div class="test pass"><span class="badge pass">PASS</span><div><div class="tname">Meme-format range</div><div class="why">Drake, Two-Buttons, Philosoraptor, reaction crops — fluent in the visual language.</div></div><div class="metric">9+ formats</div></div><div class="test partial"><span class="badge partial">PARTIAL</span><div><div class="tname">Posting timing</div><div class="why">Good weekend instinct, but no fixed slot and missed weeknight deadline windows.</div></div><div class="metric">~half right</div></div><div class="test fail"><span class="badge fail">FAIL</span><div><div class="tname">Consistent cadence</div><div class="why">A launch sprint is not a schedule; concentration in one month means no posting habit.</div></div><div class="metric">79% / 1 mo</div></div><div class="test fail"><span class="badge fail">FAIL</span><div><div class="tname">Sustained presence</div><div class="why">The algorithm forgets dormant accounts; long silence resets reach.</div></div><div class="metric">2751 days</div></div><div class="test fail"><span class="badge fail">FAIL</span><div><div class="tname">Native video / Reels</div><div class="why">Reels are the discovery engine; static-only output barely reaches non-followers.</div></div><div class="metric">3% video</div></div><div class="test fail"><span class="badge fail">FAIL</span><div><div class="tname">Captions with hooks</div><div class="why">No caption = no hook, no CTA, no keyword context for ranking.</div></div><div class="metric">6% captioned</div></div><div class="test fail"><span class="badge fail">FAIL</span><div><div class="tname">Hashtag discoverability</div><div class="why">Zero/low hashtags means little non-follower reach via tags.</div></div><div class="metric">0% tagged</div></div><div class="test fail"><span class="badge fail">FAIL</span><div><div class="tname">Native-first publishing</div><div class="why">Recycled cross-posts are suppressed and miss native features.</div></div><div class="metric">100% crosspost</div></div></div></section><section><div class="sec-head"><span class="sec-num">04</span><h2>What it posts</h2></div><p class="sec-sub">Themed from the 34 recovered meme images. The humor clusters into four pillars — the most ownable ones are the most insider.</p><div class="card"><div class="pillar"><div class="pillar-top"><span><b>Courses &amp; profs</b></span><span class="pct">37%</span></div><div class="track"><div class="fill" style="width:37%;background:#9333EA"></div></div><div class="ex">VHDL · John Thistle · Paul Ward · Harmsworth&#x27;s calc final · discrete math · Lin Alg</div></div><div class="pillar"><div class="pillar-top"><span><b>General student relatable</b></span><span class="pct">29%</span></div><div class="track"><div class="fill" style="width:29%;background:#A99FBC"></div></div><div class="ex">all-nighters · #motivation · &quot;scratch and get a star&quot; · 11:11 wishes</div></div><div class="pillar"><div class="pillar-top"><span><b>Marmoset &amp; test cases</b><span class=sig>◆ signature</span></span><span class="pct">25%</span></div><div class="track"><div class="fill" style="width:25%;background:#4f9d7c"></div></div><div class="ex">&quot;submit to marmoset at 9:59&quot; · &quot;5/6 → 6/6 release cases&quot; · &quot;0/10 fixed code&quot;</div></div><div class="pillar"><div class="pillar-top"><span><b>Co-op / WaterlooWorks</b></span><span class="pct">9%</span></div><div class="track"><div class="fill" style="width:9%;background:#CB3A9D"></div></div><div class="ex">&quot;nOt SeLeCtEd&quot; · &quot;can&#x27;t fail 2A if you don&#x27;t pass 1B&quot; · co-op streams</div></div></div></section><section><div class="sec-head"><span class="sec-num">05</span><h2>Strategy, wired to the data</h2></div><p class="sec-sub">Every recommendation traces to a finding above. Left = what the export proved. Right = the fix.</p><div class="links"><div class="lk"><div class="from "><p class="tag">Data · §01</p><h4>79% of posts in month 1, then a cliff</h4><p>A burst with no system. Motivation ran out before a habit formed.</p></div><div class="arrow">→</div><div class="to"><p class="tag">Fix</p><h4>Ship 3×/week, batch monthly</h4><p>Make <b>one batch day per month</b> producing ~12 memes, scheduled out. A queue survives bad weeks; inspiration doesn't.</p></div></div><div class="lk"><div class="from "><p class="tag">Data · §03</p><h4>97% static images · 0 Reels</h4><p>The 2018 playbook. Static memes barely reach beyond existing followers now.</p></div><div class="arrow">→</div><div class="to"><p class="tag">Fix</p><h4>Make 60% of output Reels</h4><p>Animate the same formats: text-build reveals, trending audio over a punchline, 5-sec "POV: marmoset at 9:59." Same jokes, the format the algorithm pushes.</p></div></div><div class="lk"><div class="from "><p class="tag">Data · §03</p><h4>0 hashtags · 6% captioned</h4><p>No discoverability layer at all — invisible to anyone who doesn't already follow.</p></div><div class="arrow">→</div><div class="to"><p class="tag">Fix</p><h4>Caption hook + 5–8 niche tags</h4><p>Every post gets a one-line hook and a CTA (<b>"tag the friend still on 5/6"</b>) plus #uwaterloo #waterlooengineering #ecememes + course tags. Geotag UW.</p></div></div><div class="lk"><div class="from "><p class="tag">Data · §03</p><h4>100% auto-crossposted from Facebook</h4><p>IG suppresses recycled FB posts and you lose native features.</p></div><div class="arrow">→</div><div class="to"><p class="tag">Fix</p><h4>Post native-first, then repurpose</h4><p>Create in IG (Reels, captions, tags), then push the winners out to TikTok and the FB page — not the other way around.</p></div></div><div class="lk"><div class="from "><p class="tag">Data · §02</p><h4>Weekend-heavy, no fixed slot</h4><p>Right instinct, wrong precision — and it skipped peak ECE stress hours.</p></div><div class="arrow">→</div><div class="to"><p class="tag">Fix</p><h4>Lock slots to the grind</h4><p>Post <b>Sun–Thu, 9–11pm</b> (assignment crunch + procrastination scroll) and on known deadline eves. Keep a lighter Sat/Sun slot for casual hits.</p></div></div><div class="lk"><div class="from passdrv"><p class="tag">Strength · §04</p><h4>Marmoset / prof / co-op humor lands</h4><p>The most insider pillars are the most ownable — and the most shareable inside the niche.</p></div><div class="arrow">→</div><div class="to"><p class="tag">Amplify</p><h4>Turn pillars into recurring series</h4><p><b>"Marmoset Mondays,"</b> a prof-of-the-week, and a <b>Co-op Season</b> arc each Sept/Jan. Series give followers a reason to come back.</p></div></div><div class="lk"><div class="from "><p class="tag">Data · KPI</p><h4>Stuck at 21 followers</h4><p>Great content with no distribution never compounds.</p></div><div class="arrow">→</div><div class="to"><p class="tag">Fix</p><h4>Seed where ECE already gathers</h4><p>Drop posts into class group chats, cohort Discords, and <b>r/uwaterloo</b>; tag the engineering society. Insider memes spread by being shared.</p></div></div></div></section><section><div class="sec-head"><span class="sec-num">06</span><h2>A normal week</h2></div><p class="sec-sub">A sustainable cadence, slotted to the timing data — what the launch sprint should have become.</p><div class="cal"><div class="d"><div class="dh"><span>Mon</span><span class="t">9pm</span></div><div class="slot reel"><span class=k>Reel · Series</span>Marmoset Mondays</div></div><div class="d"><div class="dh"><span>Tue</span><span class="t">—</span></div><div class="slot story"><span class=k>Story</span>Poll: &quot;5/6 or 6/6?&quot;</div><div class="slot rest">// engage in comments</div></div><div class="d"><div class="dh"><span>Wed</span><span class="t">10pm</span></div><div class="slot reel"><span class=k>Reel · Courses</span>Prof-of-the-week</div></div><div class="d"><div class="dh"><span>Thu</span><span class="t">9pm</span></div><div class="slot reel"><span class=k>Reel · Relatable</span>Deadline-eve POV</div></div><div class="d"><div class="dh"><span>Fri</span><span class="t">—</span></div><div class="slot story"><span class=k>Story</span>Repost best-DM meme</div><div class="slot rest">// reshare to FB/TikTok</div></div><div class="d"><div class="dh"><span>Sat</span><span class="t">1pm</span></div><div class="slot reel"><span class=k>Static · Light</span>Weekend casual meme</div></div><div class="d"><div class="dh"><span>Sun</span><span class="t">—</span></div><div class="slot rest">REST + monthly batch day (~12)</div></div></div><div class="note"><b>How this maps back:</b> 3 Reels (§03 fix) · niche series from the strong pillars (§04) · Sun–Thu 9–11pm slots (§02) · one batch day so cadence survives bad weeks (§01). <b>Consistent beats heavy.</b></div></section><div class="note" style="margin-top:28px"><b>About this data.</b> Built from a personal Instagram data export (34 posts, 2017-11-04 → 2018-12-12). A personal export contains <b>post content, timestamps, and format</b> — but <b>not</b> likes, comments, reach, or impressions (those need an Insights export). So this measures posting behaviour and content, which is where the fixable problems live.</div><div class="ffoot"><span>./audit @uw_ece_meme — generated Jun 24 2026</span><span>report: 6 failed · 2 passed · 1 partial</span></div></div>
```

---

---

---

---

---

*End of build guide (Chat + Dashboard-Skills Edition). If the environment's connectors, available skills, or the user's brand differ from this reference, adapt the marked customization points — but keep the platform constraints in Section 3 and the skill template in Appendix B intact; those are what make the control center and its skills work together reliably.*
