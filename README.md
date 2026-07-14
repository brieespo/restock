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
  staples jsonb not null default '[]'::jsonb,
  products jsonb not null default '[]'::jsonb,
  settings jsonb not null default '{}'::jsonb,
  updated_at timestamptz default now()
);

alter table restock_data enable row level security;

create policy "Users manage own restock data" on restock_data
  for all using (auth.uid() = user_id) with check (auth.uid() = user_id);
```

If you already have this table from before the staples/products split, just add the new column:

```sql
alter table restock_data add column if not exists staples jsonb not null default '[]'::jsonb;
```

One row per user; staples and products each sync as a jsonb blob. Guest mode works without an account and stores data in localStorage only. Old flat-product data auto-migrates to the two-level shape the first time the app loads it — one staple gets created per legacy product, and everything else carries forward untouched.

### Sync safety net

Signed-in saves now write a local cache (`rs-synccache-<user_id>` in localStorage) on every attempt, not just in guest mode. If a save to Supabase fails, a sticky banner ("Couldn't sync to the cloud — your changes are saved on this device and will keep retrying") appears and the save retries with exponential backoff (8s up to 60s) until it succeeds. On load, if the local cache has more items than what's reachable on the server (or the server can't be reached at all), the app restores from the cache instead of showing an empty/stale catalog, shows a notice explaining what happened, and re-pushes the cache to the server. This closes the gap where a schema mismatch or any other silent save failure could wipe a session's unsaved work on refresh with zero warning and zero recovery path — the exact failure mode that lost a set of live product entries on 2026-07-14.

## Data model

Two levels: a **staple** is the general need ("Face wash"); a **product** is a brand/size-specific variant of it, linked via `staple_id`. Status, snooze, and the predicted-runout engine live on the staple (pooled across all its variants); brand, price, rating, notes, on-hand count, and purchase history live on the product. A staple with one variant looks and edits exactly like a plain product — the split only becomes visible once a second brand is added.

```js
// staple
{
  id: 1,
  name: "Face wash",
  category: "skincare",          // bath_body | makeup | skincare | household | food | health | pet | other
  status: "stocked",             // 'stocked' | 'running_low' | 'out' | 'ordered'
  substitutes: "Cetaphil in a pinch",
  interval_override_days: null,
  interval_adjust: 1,
  snooze_until: null
}
// product (variant)
{
  id: 10,
  staple_id: 1,
  name: "Hydrating Facial Cleanser",
  brand: "CeraVe",
  size: "16 oz",
  where_to_buy: [{store: "Costco", price: 11.49, url: ""}],
  sale_patterns: [{store: "Ulta", when: "July", notes: "usually 20% off"}],
  rating: null, notes: "",
  discontinued_watch: false,
  on_hand: 3,                    // units on hand — powers the stock-aware radar
  purchases: [                   // every purchase logged, drives the interval engine
    {date: "2026-05-02", price: 11.49, store: "Costco", size: "16 oz", on_sale: true}
  ],
  finished_events: [             // one entry per "Finished one" tap — the usage-rate signal
    {date: "2026-06-01"}
  ],
  subscription: {                // optional — a fixed-cadence auto-ship for this variant
    active: true,
    vendor: "Trade Coffee",
    price: 34.99,                 // cost per shipment
    qty_per_shipment: 3,          // units delivered each time
    interval_days: 14,            // delivery cadence
    started: "2026-05-15",
    shipments: [                  // logged via "Log shipment" — also pushes a normal
      {date: "2026-05-15", on_hand_before: 0}   // `purchases` entry, so price history/
    ]                                            // shrinkflation tracking see it too
  }
}
```

## Roadmap

Phase 1 (done): auth, product CRUD, dashboard home (headline "N need attention" banner with computed counts + stacked color-coded panels for Out / Running low / Ordered), category tabs + search + status filter, one-tap status changes, paste-a-list importer, JSON export, themes.
Phase 2 (done): "Bought it!" one-tap purchase logging (prefills today's date + last price/store, editable); interval engine (median gap between 2+ purchases, or a manual "rebuy every ~N days" override with 0-1 purchases); a 🔮 Predicted due soon radar panel — surfaces items nearing their predicted rebuy date independent of manual status; snooze ("Still fine, +2wk") gently widens the estimate, marking something Out earlier than predicted tightens it; purchase history is viewable/editable (add/delete individual entries) from each product's edit screen.
Phase 3 (done): 🛒 Shopping mode — needs (running low / out / due soon) grouped by store, checklist style with per-store est. totals, "Mark checked as bought" bulk-logs a purchase for every checked item at once; 🔮 Restock rhythm — an 8-week heat-strip previewing predicted run-outs, click a week to see what's in it; 📈 Price watch — optional per-purchase size unlocks price-per-unit tracking, flags an 8%+ increase between two same-unit purchases (needs 150+ days between them) as a chip on the card/edit screen and its own Home panel.
Phase 4 (done, partial): a rule-based rebuying-profile line at the top of Home ("You're loyal to CeraVe and Method, you rebuy ~2 products monthly, and Costco wins on price for 3 of them") that degrades gracefully with sparse data; 🚨 Discontinued watch is now its own standing Home panel (not just a card icon) with one-tap Bought it!/Stop watching. Hub theming is deferred — there's no hub site yet to theme for; the app already shares the sibling apps' CSS-variable theme system, so it's ready to slot in whenever the hub exists.
Phase 5 (done): restructured to a two-level staples/products model — brand comparison (unit price, rating, avg. purchase gap) once a staple has 2+ variants; added Skincare category, an on-sale checkmark in purchase history, and manually-noted sale patterns (Phase 4.5, folded in here); added inventory depth — on-hand counts, one-tap "Finished one," and a stock-aware radar ("covered until ~date") that only surfaces a staple as it nears zero, falling back to the plain rebuy-pace prediction when on-hand isn't in use yet. The staple's own name field is now always visible and editable (not just once a 2nd variant exists), so a single-brand item can still be labeled by its general type ("Shampoo") rather than just its specific product name.
Phase 6 (done): paste-a-purchase — paste a purchase link and/or the copied product title + price into the Bought-it modal or a new variant's form; store comes from the link's domain, name/price/size are parsed locally from the pasted text. No scraping, no server, no API keys — just local text parsing, since a static site can't reliably fetch another site's page anyway.
Phase 7 (done): subscriptions — a variant can be marked as a fixed-cadence auto-ship (vendor, price, quantity per shipment, interval), with a one-tap "Log shipment" that adds the quantity to on-hand and *also* logs a regular purchase (so price history/shrinkflation see subscription price changes for free, no separate tracking needed). A balance check compares the subscription's incoming rate (qty/interval) against the actual usage rate (median gap between "Finished one" taps); once there's enough finish-event data it gives a real verdict — running short, building a surplus, or well matched — falling back to an on-hand trend across logged shipments when finish data is sparse. Surfaced on its own Home dashboard panel and inline on the product card/edit screen.
Phase 8 (done): sync safety net — see "Sync safety net" above. Local cache for every signed-in save, a visible failure banner with retrying backoff instead of a silent `console.error`, and load-time reconciliation that prefers a richer local cache over a leaner or unreachable remote copy.

See `CLAUDE.md` for the full plan.
