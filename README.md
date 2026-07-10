# 🧺 Restock

One place for the products you actually use and rebuy — a catalog of trusted brands ("what's the moisturizer I liked?") plus a reorder radar ("what's about to run out?"). Sibling app to [dinner-planner](https://github.com/brieespo/dinner-planner), [perfume-tracker](https://github.com/brieespo/perfume-tracker), and [sewing-tracker](https://github.com/brieespo/sewing-tracker): pure HTML + CSS + vanilla JS in a single file, Supabase for auth + sync, GitHub Pages for hosting. Mobile-first — marking things "running low" happens standing in the bathroom.

**Live at:** https://brieespo.github.io/restock

## Files

- `restock.html` — the entire app (source of truth)
- `index.html` — always an exact copy of `restock.html`. After every change: `cp restock.html index.html`, then push.
- `.github/workflows/deploy.yml` — GitHub Pages deploy on every push to `main`

## Supabase setup

Reuses the shared Supabase project (same URL and publishable key as the sibling apps, so one account works everywhere) with its own table, `restock_data`. Run once in the SQL editor:

```sql
create table if not exists restock_data (
  user_id uuid primary key references auth.users(id) on delete cascade,
  products jsonb not null default '[]'::jsonb,
  settings jsonb not null default '{}'::jsonb,
  updated_at timestamptz default now()
);

alter table restock_data enable row level security;

create policy "Users manage own restock data" on restock_data
  for all using (auth.uid() = user_id) with check (auth.uid() = user_id);
```

One row per user; products sync as a jsonb blob. Guest mode works without an account and stores data in localStorage only.

## Data model

```js
{
  id: 1,
  name: "Daily Moisturizing Lotion",
  brand: "CeraVe",
  category: "bath_body",        // bath_body | makeup | household | food | health | pet | other
  size: "19 oz",
  where_to_buy: [{store: "Costco", price: 11.49, url: ""}],
  status: "stocked",            // 'stocked' | 'running_low' | 'out' | 'ordered'
  purchases: [],                // Phase 2: every purchase logged, drives the interval engine
  interval_override_days: null,
  substitutes: "Cetaphil in a pinch",
  rating: null, notes: "",
  discontinued_watch: false     // flag: stock up when seen
}
```

## Roadmap

Phase 1 (done): auth, product CRUD, dashboard home (headline "N need attention" banner with computed counts + stacked color-coded panels for Out / Running low / Ordered), category tabs + search + status filter, one-tap status changes, paste-a-list importer, JSON export, themes.
Phase 2: purchase logging ("Bought it"), interval engine (median purchase gap), Reorder Radar with snooze.
Phase 3: shopping mode by store, price/unit history + shrinkflation flag, restock rhythm view.
Phase 4: polish — trusted-brands line, discontinued-watch surfacing, hub theming.

See `CLAUDE.md` for the full plan.
