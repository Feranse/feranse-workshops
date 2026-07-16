# My Business Command Center — Master Prompt (Cowork version)

Fill in every [BLANK], then paste the whole prompt into **Claude Cowork** (the desktop app) with a project folder selected. This version lives as real files in your folder, so every report you generate is saved and linkable.

---

**Role:** You are an expert web designer and small-business operations assistant. You build clean, branded, file-based dashboards.

**Instructions:** In my project folder, build a "Business Command Center" home page as `index.html`, branded for:

- Business name: [YOUR BUSINESS NAME]
- What I do: [ONE LINE — e.g. "hand-poured soy candles, sold online and at markets"]
- Brand colors: [2–3 COLORS, e.g. "blush pink, cream, deep wine"]
- Vibe: [e.g. "warm, feminine, professional — not corporate"]

**Steps:**
1. Create `index.html`: a header with my business name and today's date, then three tool cards:
   - **Morning Briefing** — links to `briefings/latest.html`, with a "how to run" hint: "Ask Claude: run my morning briefing and save it to the briefings folder."
   - **Cash Flow** — links to `reports/cashflow-dashboard.html`, hint: "Drop a bank statement PDF in this folder and ask Claude to analyze it."
   - **Instagram Audit** — links to `reports/instagram-dashboard.html`, hint: "Drop your Instagram export zip here and ask Claude to audit it."
2. Create the empty `briefings/` and `reports/` folders with a small placeholder page in each that says "Nothing here yet — run me from Cowork."
3. Style everything with my brand colors; single shared look across pages.

**End goal:** A home page I can open every morning from my own folder. When I run the matching skills in Cowork (Morning Briefing, Bank Statement Analyzer, Instagram Dashboard), their output files land in these folders and the cards light up with real, saved reports I can reopen anytime.

**Narrowing:** Plain HTML/CSS only, no build tools, no external images. Never invent data — placeholder pages stay honest until a skill writes a real report. Keep file names exactly as specified so the links always work.

---

## After it builds

1. Install the three **Cowork skills** (from the workshop downloads): in Cowork, add them to your skills so Claude can run them in this folder.
2. Try it end to end: "Run my morning briefing and save it to the briefings folder." Then open `index.html` and click the card.
3. Iterate out loud: "add a fourth card for invoices," "make the cards bigger," "add my logo." Keep prompting — that's the whole skill.
