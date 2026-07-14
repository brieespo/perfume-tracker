# Perfume Tracker — Planning Doc / CLAUDE.md

Planning document for a personal perfume tracking web app. Written to be dropped into a new repo as `CLAUDE.md` (or kept alongside it) for Claude Code to build from.

## Who this is for

Bri — perfume lover, currently tracking her collection in a 7-tab spreadsheet (`Fragrance.xlsx`). This app replaces that spreadsheet. It is a personal tool, single user, but built on the same account/sync pattern as her existing dinner planner so it could support others later.

## Why the spreadsheet failed (design motivation)

- Four separate tabs (Permanent Collection / Samples / Sold-Unloved / To Try) meant a perfume's lifecycle required copy-pasting rows between tabs.
- Perfume notes were free text in inconsistent formats, so analyzing "what notes do I love?" required manual tallying — attempted three times across three tabs, with duplicate/miscounted entries each time.
- Excel auto-converted ratings like "3.5/5" into dates, destroying the data. Only 3 of 143 samples ended up with usable scores.

Every one of those failures maps to a feature below.

## Tech stack — match the dinner planner exactly

Same conventions as github.com/brieespo/dinner-planner (see its CLAUDE.md):

- **Pure HTML + CSS + vanilla JavaScript, one file** (`perfume-tracker.html`), no frameworks, no npm, no build step.
- `index.html` is always a copy of `perfume-tracker.html`; after every change: `cp perfume-tracker.html index.html`, then push.
- **Supabase** for auth (email sign-in) and cloud storage. Bri already has a Supabase account — reuse the existing project with a new table, or create a project for this app (builder's choice; note which in the README).
- **GitHub Pages** hosting via the same GitHub Actions deploy workflow as dinner-planner (`actions/checkout@v5`, `configure-pages@v6`, `upload-pages-artifact@v5`, `deploy-pages@v5`).
- **No external JS libraries except the Supabase client.** Charts are built with CSS (horizontal bar charts are just divs with widths) — no Chart.js.
- Style: CSS variables for theming, card-based UI, mobile-friendly. This app should feel like a sibling of the dinner planner; it will eventually be linked from a personal hub site alongside it.
- Suggested repo name: `perfume-tracker` → live at brieespo.github.io/perfume-tracker

## Data model

### Supabase

Follow the dinner planner's pattern: one row per user in a `perfume_data` table with jsonb columns.

| column | type | contents |
|---|---|---|
| user_id | uuid (PK, references auth) | |
| perfumes | jsonb | array of perfume objects (below) |
| note_map | jsonb | user's note alias map + custom canonical notes |
| wear_log | jsonb | array of daily wear entries: `{id, perfume_id, date}` |
| settings | jsonb | theme, default chart view, etc. |

~240 records is tiny; jsonb blobs synced whole (like dinner planner recipes) are fine. No need for relational tables.

### Perfume object

```js
{
  id: 1,                      // integer, assigned like dinner planner's nextId
  name: "John Frume",
  house: "Aether Arts",       // brand
  status: "owned",            // 'owned' | 'sampled' | 'sold' | 'to_try' | 'finished'
  price: 58.00,               // number or null
  size_ml: 5.5,               // number or null
  acquired: "American Perfumer - 2019",  // free text, matches existing data
  rating: 3.5,                // 0.5–5 in half steps, or null. UI enforces — never free text
  notes_raw: "Lime & coconut water, guava, ...",  // as pasted, never modified
  notes: ["lime", "coconut", "guava"],            // canonical note ids, parsed
  notes_reviewed: false,      // true once user confirms the parse
  description: "...",         // marketing copy, optional
  thoughts: "Love. Cold but interesting.",        // Bri's own review, free text
  all_natural: true,          // or null
  history: [{status: "sampled", date: "2026-07-09"}]  // appended on status change
}
```

**Status is the core concept.** One list, five statuses, replacing four spreadsheet tabs. Changing status (to_try → sampled → owned → sold/finished) is one tap on the perfume card and appends to `history`. No data is ever re-entered.

`finished` means used up because she loved it — the strongest possible endorsement. It is distinct from `sold` (rejected). Owned and finished perfumes both count as "loved" everywhere; finished counts *more* in preference scoring.

### Wear log

A flat, append-only log of daily wear, separate from `history` (which tracks lifecycle status changes, not day-to-day wear): `{id, perfume_id, date}`. One tap ("Wearing today") on any card logs today's date and toggles off again on a second tap; a dedicated Wear Log screen lists the full history grouped by date and lets you log a specific past date. This is data-collection only for now — no frequency analysis yet. The flat one-row-per-wear-per-day shape is deliberately simple so a future frequency breakdown (most-worn perfumes, by season/day-night, alongside the existing preference-score charts) is just a group-and-count away.

### Canonical note vocabulary

Two-layer system so counting is reliable but nothing is lost:

- `notes_raw` — exactly what the user pasted; never touched.
- `notes` — array of canonical note ids, produced by the parser and user-confirmable.

The app ships with a built-in canonical note list (~150 notes) organized into families:

| family | example notes |
|---|---|
| white floral | tuberose, jasmine, orange blossom, gardenia, ylang-ylang, frangipani, magnolia |
| floral | rose, iris, violet, peony, lily, osmanthus, mimosa |
| citrus | orange, lime, lemon, grapefruit, bergamot, neroli |
| fruity | peach, apple, guava, mango, coconut, fig, berry |
| warm/amber | amber, vanilla, benzoin, tonka, labdanum, honey, beeswax |
| woody | sandalwood, cedar, vetiver, oud, "woods" (generic) |
| spicy | pepper, cardamom, cinnamon, coriander, cumin, clove |
| green/herbal | green notes, grass, tea, wormwood, herbs |
| musky/animalic | musk, ambergris, africa stone, castoreum |
| smoky/resinous | incense, tobacco, leather, smoke, resins |
| fresh/other | ozone, aquatic, aldehydes, powder |

(Builder: extend to a full list during implementation; families matter more than completeness — unknown notes can always be added as custom.)

An **alias map** normalizes variants to canonical ids: "Bourbon vanilla" → vanilla, "Indonesian vetiver" → vetiver, "juicy orange accord" → orange, "Rosa Gallica" → rose. Seed the alias map generously from the actual strings in `perfume-seed-data.csv` — that file contains the exact vocabulary this user's data uses.

## Features

### 1. Collection management (Phase 1)

- Card grid of perfumes, filterable by status (tab strip like dinner planner: Owned / Samples / Finished / Sold / To Try / All), searchable by name/house/note.
- Add/edit form: name, house, status, price, size, rating (star/half-star picker — enforced input, this is the anti-Excel-date feature), notes paste box, thoughts, description, all-natural toggle.
- One-tap status changes from the card (e.g., a "Bought it! → Owned" button on a sampled perfume).
- Price-per-ml computed and displayed, never stored.
- Export all data as JSON (same as dinner planner's export).

### 2. Note parsing & review (Phase 2)

When the user pastes notes text, the parser:

1. Strips structure words: "Top Notes:", "Middle Notes:", "Base Notes:", "accord", "notes of", trailing periods.
2. Splits on commas, "&", "and".
3. Lowercases and matches each fragment against canonical notes + alias map (substring match: "orange blossom accord" hits "orange blossom" before "orange").
4. Unmatched fragments surface as chips in a review strip: user taps to assign to an existing note, create a new custom note (picking its family), or discard. Assignments are saved to the user's alias map so the same string never asks twice.

A **review queue** screen lists perfumes with `notes_reviewed: false` so imported/backlog perfumes can be cleaned up gradually. Nothing blocks on review — unparsed perfumes just don't contribute to charts until reviewed (show a count badge: "12 perfumes not yet in your charts").

No LLM calls — the app is a static site and can't hold API keys. The alias map + review UI achieves the same end. (Stretch idea, not Phase 1–3: a Supabase Edge Function proxying an LLM for note parsing.)

### 3. Note charts (Phase 3)

A "My Notes" screen with a CSS horizontal bar chart of note frequency, with a **dropdown selecting the view**:

- **Owned + Finished** (default — this is the "what I actually love" view)
- Owned
- Finished
- Sampled
- Sold / Unloved
- To Try
- All
- **Preference score** — the smart view: each note scores `+3` per finished perfume, `+2` per owned perfume, `+1` per sampled perfume rated ≥ 4, `0` per unrated/mid sample, `−1` per sold perfume. Explicitly: plain sampled perfumes do NOT count toward preference (user requirement — sampling something is not loving it).

Secondary chart on the same screen: the same data rolled up by **note family** (white floral, warm/amber, etc.). Toggle between note-level and family-level.

### 4. Scent profile line (Phase 3)

A dynamically generated one-liner at the top of the My Notes screen — "This is what you look for in a perfume." Rule-based, not ML:

1. Compute family-level preference scores (from the Preference score view).
2. Take the top 2 families, plus the top individual note.
3. Render through sentence templates, e.g.:
   - "You're a **white floral** person with a weakness for **warm amber** bases — and you can't say no to **rose**."
   - If a family dominates sold perfumes, add a gentle negative: "…but sweet gourmands keep letting you down."
4. A few template variants chosen by data shape (one dominant family vs. two close ones vs. scattered) so it doesn't feel canned. Recompute live; it should shift as the collection changes.

Keep it charming and short. This is the feature that makes the app feel like it knows her.

### 5. Import (Phase 1, one-time)

`perfume-seed-data.csv` sits in the repo — 240 perfumes already consolidated from the spreadsheet with a `status` column (26 owned, 143 sampled, 59 sold, 12 to_try; 104 have raw notes text).

Build a hidden/one-time import: "Import from CSV" in a ⋯ More menu that fetches the CSV, creates perfume objects (`notes_reviewed: false`), and syncs. After import the review queue drives note cleanup at leisure.

CSV columns: `name, house, status, price, size_ml, acquired, rating, notes_raw, description, all_natural, thoughts`.

## Build phases

1. **Phase 1 — Working replacement for the spreadsheet:** auth, perfume CRUD, status tabs + one-tap status change, search, CSV import, JSON export. *Usable from day one.*
2. **Phase 2 — Structured notes:** canonical vocabulary + alias map, paste-parser, review chips, review queue.
3. **Phase 3 — The payoff:** note charts with view dropdown, family rollup, scent profile line.
4. **Phase 4 — Polish:** filters (by rating, house, all-natural), price-per-ml sort, "sample first?" nudges on to_try items, theming to match the future hub site.

## Explicit non-goals

- No Fragrantica/Parfumo scraping or API — none exists publicly; the paste-parser is the ingestion path.
- No multi-user community features in v1 (the dinner planner pattern leaves the door open).
- No LLM features in v1–v3.

## Open questions for the builder to confirm with Bri

1. Reuse the dinner-planner Supabase project (new table) or a fresh project?
2. Half-star rating scale 0.5–5 assumed; confirm.
3. During CSV import review, some rows in `status: sold` may really be `finished` — the import UI could ask, or Bri can fix statuses one-tap afterward.

## Design language (suite-wide rules)

- **No emoji in UI chrome.** Buttons, menus, headers, tab labels use inline SVG line icons (Lucide/Feather style, open-licensed, pasted as inline <svg> with stroke="currentColor" so they tint via CSS variables). No icon library or CDN.
- **Status markers are CSS dots/chips** in the theme palette, never colored emoji.
- **One identity mark**: a single logo glyph in the header is the only decorative one on screen.
- Emoji is allowed in user-defined content (tags, notes) — data, not chrome.
- Warmth via accent colors, rounded cards, micro-copy voice — not decoration.

## Model escalation

If a task appears to exceed your ability — a fix has failed twice, architectural uncertainty, or a risky data-model change — say so explicitly and recommend rerunning on a more capable model (/model fable) instead of continuing to attempt it.
