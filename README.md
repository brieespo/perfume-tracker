# 🌸 Perfume Tracker

A personal perfume collection tracker — one list, five statuses (To Try → Sample → Owned → Finished / Sold), replacing a 7-tab spreadsheet. Sibling app to [dinner-planner](https://github.com/brieespo/dinner-planner): pure HTML + CSS + vanilla JS in a single file, Supabase for auth + sync, GitHub Pages for hosting.

**Live at:** https://brieespo.github.io/perfume-tracker

## Files

- `perfume-tracker.html` — the entire app (source of truth)
- `index.html` — always an exact copy of `perfume-tracker.html`. After every change: `cp perfume-tracker.html index.html`, then push.
- `perfume-seed-data.csv` — one-time import data consolidated from the old spreadsheet (⋯ More → Import from CSV)
- `.github/workflows/deploy.yml` — GitHub Pages deploy on every push to `main`

## Supabase setup

This app **reuses the dinner-planner Supabase project** (same URL and publishable key, so one account works for both apps) with its own table, `perfume_data`. Run this once in the Supabase SQL editor:

```sql
create table if not exists perfume_data (
  user_id uuid primary key references auth.users(id) on delete cascade,
  perfumes jsonb not null default '[]'::jsonb,
  note_map jsonb not null default '{}'::jsonb,
  wear_log jsonb not null default '[]'::jsonb,
  settings jsonb not null default '{}'::jsonb,
  updated_at timestamptz default now()
);

alter table perfume_data enable row level security;

create policy "Users manage own perfume data" on perfume_data
  for all using (auth.uid() = user_id) with check (auth.uid() = user_id);
```

If you already have this table from before the wear log, just add the new column:

```sql
alter table perfume_data add column if not exists wear_log jsonb not null default '[]'::jsonb;
```

One row per user; the whole collection syncs as jsonb blobs (~240 perfumes is tiny). Guest mode works without an account and stores data in localStorage only.

## Data model

Each perfume object:

```js
{
  id: 1,
  name: "John Frume",
  house: "Aether Arts",
  nose: "Amber Jobin",           // the perfumer, optional
  season: "fall",                // 'spring' | 'summer' | 'fall' | 'winter' | null
  daynight: "night",             // 'day' | 'night' | null
  status: "owned",            // 'owned' | 'sampled' | 'sold' | 'to_try' | 'finished'
  price: 58.00,               // number or null
  size_ml: 5.5,               // number or null
  acquired: "American Perfumer - 2019",
  rating: 3.5,                // 0.5–5 in half steps, or null — enforced by the star picker
  notes_raw: "Lime & coconut water, guava, …",   // as pasted, never modified
  notes: [],                  // canonical note ids (Phase 2)
  notes_reviewed: false,      // true once the parse is confirmed (Phase 2)
  description: "…",
  thoughts: "Love. Cold but interesting.",
  all_natural: true,          // true | false | null
  history: [{status: "sampled", date: "2026-07-09"}]  // appended on every status change
}
```

Wear log — a separate flat array, one entry per perfume worn per day:

```js
{ id: 1, perfume_id: 3, date: "2026-07-14" }
```

## Roadmap

Phase 1 (done): auth, CRUD, status tabs + one-tap status changes, search, star ratings, CSV import, JSON export, themes.
Phase 2 (done): canonical note vocabulary (~170 notes, 11 families) + alias map, paste-parser with review chips, review queue, FS-worthy flag on samples.
Phase 3 (done): Trends screen (Notes tab) — note/family frequency charts (pure CSS bars), preference-score view (finished ×3, owned ×2, loved samples ×1, sold −1), live scent-profile line.
Phase 4: filters, price-per-ml sort, polish.
Phase 5 (done): daily wear log — one-tap "Wearing today" toggle on each card, a Wear Log screen for the full history (grouped by date, with past-date entry and delete), synced like everything else.
Phase 6 (done): wear frequency tables, and a restructure of the old "My Notes" screen into a **Trends** nav tab with a sub-tab strip (Notes / Wear Frequency) — built so future stat views are just another tab, not a redesign.

See `CLAUDE.md` for the full plan.
