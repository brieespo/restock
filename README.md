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
Phase 2 (done): "Bought it!" one-tap purchase logging (prefills today's date + last price/store, editable); interval engine (median gap between 2+ purchases, or a manual "rebuy every ~N days" override with 0-1 purchases); a 🔮 Predicted due soon radar panel — surfaces items nearing their predicted rebuy date independent of manual status; snooze ("Still fine, +2wk") gently widens the estimate, marking something Out earlier than predicted tightens it; purchase history is viewable/editable (add/delete individual entries) from each product's edit screen.
Phase 3 (done): 🛒 Shopping mode — needs (running low / out / due soon) grouped by store, checklist style with per-store est. totals, "Mark checked as bought" bulk-logs a purchase for every checked item at once; 🔮 Restock rhythm — an 8-week heat-strip previewing predicted run-outs, click a week to see what's in it; 📈 Price watch — optional per-purchase size unlocks price-per-unit tracking, flags an 8%+ increase between two same-unit purchases (needs 150+ days between them) as a chip on the card/edit screen and its own Home panel.
Phase 4 (done, partial): a rule-based rebuying-profile line at the top of Home ("You're loyal to CeraVe and Method, you rebuy ~2 products monthly, and Costco wins on price for 3 of them") that degrades gracefully with sparse data; 🚨 Discontinued watch is now its own standing Home panel (not just a card icon) with one-tap Bought it!/Stop watching. Hub theming is deferred — there's no hub site yet to theme for; the app already shares the sibling apps' CSS-variable theme system, so it's ready to slot in whenever the hub exists.

See `CLAUDE.md` for the full plan.
