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
  settings jsonb not null default '{}'::jsonb,
  updated_at timestamptz default now()
);

alter table perfume_data enable row level security;

create policy "Users manage own perfume data" on perfume_data
  for all using (auth.uid() = user_id) with check (auth.uid() = user_id);
```

One row per user; the whole collection syncs as jsonb blobs (~240 perfumes is tiny). Guest mode works without an account and stores data in localStorage only.

## Data model

Each perfume object:

```js
{
  id: 1,
  name: "John Frume",
  house: "Aether Arts",
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

## Roadmap

Phase 1 (done): auth, CRUD, status tabs + one-tap status changes, search, star ratings, CSV import, JSON export, themes.
Phase 2: canonical note vocabulary + alias map, paste-parser, review queue.
Phase 3: note frequency charts (CSS bars), family rollup, preference score, scent profile line.
Phase 4: filters, price-per-ml sort, polish.

See `CLAUDE.md` for the full plan.
