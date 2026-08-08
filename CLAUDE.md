# Restock Command Center — Planning Doc / CLAUDE.md

Planning document for a personal favorite-products + repurchase tracking web app. Drop into a new repo as `CLAUDE.md` for Claude Code to build from. Sibling app to the dinner planner, perfume tracker, law school app, and sewing tracker; will eventually link from a personal hub site.

**Visual direction (important):** Bri prefers the layout style of the Law School Command Center — a dashboard-first app: a rich home screen with a headline status banner, stacked color-coded panels, and prominent computed numbers, rather than a plain card grid with tabs. Match that app's visual language (same theming approach, same panel/banner patterns) so the two feel like siblings. Reference its CLAUDE.md and built screens when styling this one.

## Who this is for

Bri — wants one place for the products she actually uses and rebuys: bath & body, makeup, household goods, certain staple foods. She's tried to get this from Amazon and other tools and nothing stuck — subscription models fail because they impose rigid schedules on irregular reality. The design principle throughout: **observed behavior over declared schedules.** The app watches when she actually rebuys and predicts from that.

## The three jobs of this app

1. **Reorder radar** — what's running low or statistically due to run out, surfaced before it becomes an emergency errand.
2. **Trusted-products catalog** — the brands and exact products she's vetted ("which moisturizer was the good one?"), with where to buy and at what price.
3. **Purchase intelligence** — price history, unit-price drift (shrinkflation), and rebuy rhythms she'd never notice without data.

## Tech stack — match the dinner planner exactly

Same conventions as github.com/brieespo/dinner-planner (see its CLAUDE.md):

- Pure HTML + CSS + vanilla JS, one file (`restock.html`), `index.html` always a copy. No frameworks, no build step.
- Supabase for auth + storage (Bri has an account).
- GitHub Pages + same Actions deploy workflow.
- No external JS libraries except the Supabase client. Charts/heat-strips are CSS-built.
- **This is the most phone-centric app of the suite** — "running low" gets marked standing in the bathroom; shopping mode happens in-store. Every interaction must work one-handed on a phone. Dashboard panels stack vertically on mobile.
- Suggested repo: `restock`.

## Data model (Supabase)

Dinner-planner pattern: one row per user in `restock_data` with jsonb columns:

| column | contents |
|---|---|
| staples | array of staple objects (below) — the general need |
| products | array of product objects (below) — brand/size-specific variants of a staple |
| settings | category list, store list, snooze defaults, theme |

**Two-level model.** A staple is the thing you need — "Face wash," "Paper towels," "Coffee beans." A product is a specific brand/size you've actually bought for it — CeraVe vs. Cetaphil face wash — linked back via `staple_id`. Status, the radar, predicted run-out, and snooze all live on the **staple**, because "do I need to buy face wash" doesn't care which brand; the interval engine pools purchases (and finish events) across every variant of a staple. Brand, price, rating, notes, on-hand count, and purchase history stay on the **product**, because "which one is actually good" is a per-variant question.

A staple with exactly one variant is displayed and edited exactly like a single flat product — the two-level structure is invisible until a second brand gets added (via "Track another brand for this" on the edit screen), at which point the staple's own name becomes an editable field and a **brand comparison** table appears.

### Staple object

```js
{
  id: 1,
  name: "Face wash",             // the general need; for a single-variant staple this mirrors the product's name
  category: "skincare",          // bath_body | makeup | skincare | household | food | health | pet | other (user-extendable)
  status: "stocked",             // 'stocked' | 'running_low' | 'out' | 'ordered'
  substitutes: "Cetaphil in a pinch",  // acceptable backup when nothing you track is available
  interval_override_days: null,  // manual interval if desired; otherwise computed from pooled purchases
  interval_adjust: 1,            // snooze-feedback multiplier on the computed interval
  snooze_until: null             // date string or null
}
```

### Product object (a variant of a staple)

```js
{
  id: 10,
  staple_id: 1,
  name: "Hydrating Facial Cleanser",
  brand: "CeraVe",
  size: "16 oz",
  where_to_buy: [{store: "Costco", price: 11.49, url: ""}],   // multiple stores, price per store
  sale_patterns: [{store: "Ulta", when: "July", notes: "20% off"}],  // routinely-on-sale notes
  rating: null, notes: "",        // catalog job: why this one, shade names, dupes tried
  discontinued_watch: false,      // stock up whenever seen (this specific formula being phased out)
  on_hand: 3,                     // units currently on hand
  purchases: [                    // every purchase logged, on_sale flags an on-sale buy
    {date: "2026-05-02", price: 11.49, store: "Costco", size: "16 oz", on_sale: false}
  ],
  finished_events: [              // one entry per "Finished one" tap — the usage-rate signal
    {date: "2026-06-01"}
  ],
  subscription: {                 // optional — a fixed-cadence auto-ship for this variant
    active: true,
    vendor: "Trade Coffee", price: 34.99,       // cost per shipment
    qty_per_shipment: 3, interval_days: 14,     // e.g. 3 bags every 2 weeks
    started: "2026-05-15",
    shipments: [{date: "2026-05-15", on_hand_before: 0}]  // "Log shipment" also pushes
  }                                                        // a normal purchases entry
}
```

### Subscriptions (fixed-cadence auto-ships)

A variant can be flagged as a subscription instead of (or alongside) one-off purchases: vendor, price per shipment, quantity per shipment, and interval. "Log shipment" is one tap — it adds the quantity to `on_hand` *and* pushes a normal `purchases` entry, so price history and shrinkflation detection see subscription price changes without any separate code path. The balance check compares the subscription's incoming rate (`qty_per_shipment / interval_days`) against the actual usage rate (median gap between `finished_events`) — >15% faster usage than incoming supply reads as "running short," >15% slower reads as "building a surplus," otherwise "well matched." With too few finish-events to compute a usage rate, it falls back to comparing `on_hand_before` across the last two logged shipments (rising = surplus, both zero = short) rather than showing nothing.

### The interval engine (computed, the app's requirements-engine equivalent)

- **Purchase-gap pace (fallback signal).** With 2+ purchases pooled across all of a staple's variants: predicted run-out = last purchase date + **median** gap between purchases. With fewer: `interval_override_days`. This is the same signal as before the two-level split, just pooled across brands.
- **Stock-aware depletion (primary signal once on-hand is in use).** Once any variant has `on_hand >= 1`, predicted run-out becomes *today + (usage rate × total units on hand)* — "covered until ~date" — instead of a pace guess. Usage rate is the median gap between **finished events** (pooled across variants), falling back to the purchase-gap pace when there isn't enough finish history yet. This means a staple with backups on the shelf goes quiet on the radar and only resurfaces as it actually nears zero; a staple that's never had on-hand tracked behaves exactly like the old date-only engine (no regression).
- A staple surfaces on the radar when *either* signal fires, and the UI distinguishes them: "you marked this low" vs. "covered until ~date" vs. "you usually rebuy around now."
- **Feedback loop:** "still fine, snooze 2 weeks" stretches the interval estimate slightly; "ran out early" shrinks it. Accuracy improves with normal use, including lazy use.
- **"Bought it" is one tap** — asks which variant if a staple has more than one (defaulting to whichever was purchased most recently), logs today's date and a quantity (default 1, editable) that adds to that variant's `on_hand`, with price/store/size prefilled from its last purchase. Frictionless logging is non-negotiable; it is the lesson of every abandoned spreadsheet.
- **Paste-a-purchase.** No scraping — a static site can't fetch another site's page (CORS), and Amazon in particular blocks the server-side proxies that would otherwise get around that. Instead: paste a purchase link and/or the product title + price copied off the page into the Bought-it modal (or a new variant's form); the store name comes from the link's own domain (instant, no network call), and name/price/size are parsed locally from whatever text came with it — the same local-parsing approach as the rest of the app's paste features.
- **"Finished one" is the inventory counterpart** — one tap on a specific variant decrements `on_hand` (floor zero) and logs a `finished_events` entry, feeding the usage-rate calculation above.
- **Brand comparison** (shown once a staple has 2+ variants): unit price, rating, and average purchase gap (longevity) computed per variant, so "which one's actually winning" has an answer.

## Screens

1. **Dashboard (home)** — modeled directly on the law school app's Today/Agenda screen:
   - **Headline banner:** the computed status line — "3 items need attention · next predicted run-out: dish soap, ~Aug 20." Equivalent of the law school app's "on pace to graduate" headline.
   - **Reorder Radar panel:** merged list of marked-low/out products + statistically-due products, grouped by urgency (Out now / This week / Coming up), color-coded by category the way the agenda color-codes calendars. Each row: one-tap **Bought it** and **Snooze**.
   - **Restock rhythm panel:** an 8-week CSS heat-strip of predicted run-outs — scattered errand anxiety becomes "one Costco run around Aug 20."
   - **Trusted-brands line** (the perfume-profile idea, restock edition): "Loyal to CeraVe and Method · ~9 rebuys/month · Costco wins on price for 6 products."
2. **Catalog** — the full vetted-products reference: searchable, filterable by category and store, status dot per product. Cards link to a detail view.
3. **Product detail** — purchase-history timeline, price-history line (CSS), unit-price drift callout ("+12%/oz since last year"), substitutes, notes, where-to-buy with per-store prices.
4. **Shopping mode** — active needs grouped by store, checklist style, per-store totals. Same proven pattern as the dinner planner's grocery list; built one-handed-in-a-store first.
5. **Add product / importer** — minimal required fields (name, brand, category). Day-one bulk importer: paste a list, one product per line ("brand – product – store"), parsed best-effort, editable after.

## Ideas worth considering (novel features)

- **Shrinkflation flag:** size is logged per purchase, so unit price is computable over time — the detail view calls out drift ("this got 12% more expensive per oz this year"), and the dashboard can surface the worst offender.
- **Discontinued watch:** flagged products show a persistent "stock up when seen" chip — for the shade/formula being phased out.
- **Snooze-informed learning** (described in the engine): the radar gets quieter and more accurate the more it's corrected.
- **Hub synergy (later, hub-level):** the dinner planner owns weekly groceries; the future hub dashboard merges "reorder soon" with the week's grocery list into a single shopping trip.

## Build phases

1. **Phase 1:** auth, product CRUD, catalog with categories/search, status changes, paste-list importer.
2. **Phase 2:** purchase logging ("Bought it"), interval engine with snooze feedback, dashboard with headline banner + Reorder Radar panel.
3. **Phase 3:** shopping mode by store, price/unit history + shrinkflation callouts, restock rhythm heat-strip.
4. **Phase 4:** polish — trusted-brands line, discontinued watch, hub theming.
5. **Phase 8 (sync safety net, done):** every signed-in save writes a local cache alongside the Supabase upsert; a failed save shows a sticky banner and retries with exponential backoff instead of failing silently; on load, a richer/newer local cache wins over a leaner or unreachable remote copy, with a visible notice either way and an immediate re-push. Added after a schema mismatch (`staples` column missing on the live table) caused silent save failures with no local fallback, losing a session's unsaved product entries on refresh (2026-07-14). See README's "Sync safety net" section.

## Open items (confirm with Bri)

1. Rough product count to expect (dozens vs. hundreds — affects whether category filters suffice or search must lead).
2. Food scope: dinner planner owns weekly groceries. Suggested default — only non-weekly staples here (the specific olive oil, Sloane's snacks) to avoid double entry. Confirm.
3. Store list to seed (Costco, Target, Kroger, Amazon…?).

## Sync invariant (do not regress)

The local cache is a safety net for writes that never reached the server. It is
**not** a second source of truth, and which copy wins a load is a question about
time, not size.

This originally compared item counts — if the cache held more items than the
server, an earlier save was assumed lost, so the cache won and was pushed back
up. That destroyed data: a day-old cache legitimately outnumbers a server that
another device has since pruned, so the stale copy won the load and overwrote
the newer one. In the agenda app (2026-08-06) a laptop restored the previous day
and deleted tasks a phone had added that morning; the restored copy also
predated that day's check-offs, so rollover moved completed tasks back to today.

Rules to preserve:

- The cache stores `updated_at` (stamp of the write it holds) and `synced`
  (whether the server confirmed it). Never compare item counts.
- Prefer the cache only when the server is unreachable, or when it is unsynced
  **and** newer than the server's row.
- When the server is unreachable, do **not** push the cache up. Pushing blind is
  the step that overwrites the other device.
- Flush pending debounced saves on `pagehide` and on backgrounding — a phone is
  suspended the moment it is locked, often inside the debounce window.
- **Backgrounding must re-send over the network, not just settle the debounce.**
  Flushing only guaranteed the localStorage write; the upsert itself was an
  ordinary `fetch` that died with the frozen page, so an edit reached the cache
  and never the server. The leaving path posts directly to PostgREST with
  `keepalive: true`, which outlives the page. Bodies over ~60KB exceed the
  browser cap, so they decline and fall back to the unsynced-cache net rather
  than truncating.
- "A save is pending" means queued **or in the air**, not just queued.
  `savePending()` exists here even though nothing consults it yet — this app has
  no refetch or Realtime path (see the gap below), and the guard is in place so
  that adding one later cannot reintroduce the window that bit the other two.
- **Load-time normalization must not write to the server either.** The `_u` rule
  below is about the item stamp, but the *row's* `updated_at` is what actually
  decides a load. `applyPaceRecalcMarks` used to save from `afterLoad`, so merely
  opening the app republished the row and let a passive device outrank one
  holding real unsynced work — the bug that destroyed a phone's edits in agenda
  on 2026-08-08. It no longer saves. That is safe *specifically* because its
  result is idempotent from the server's state: the adjustment and the deletion
  of `pace_recalc_from` both stay in memory, so the next load recomputes the
  identical value from the same untouched base rather than compounding 0.85 each
  time. Verified by loading five times and getting 0.85 every time; compounding
  would have hit the 0.5 floor. Do not copy this omission to a load-time fixup
  whose result is *not* idempotent — see agenda's `syncRestockOutTasks`, which
  keeps its save because its ledger is real state.
- Never let a refetch or Realtime update replace memory while a save is pending.

## Cross-device merge — where it stands

Resolution is still *wholesale* last-write-wins at the row level: if two devices
both edited during a network gap, one side's changes are lost entirely. The fix
is phased.

**Phase A — SHIPPED 2026-08-06, all three apps.** Metadata only: written
everywhere, read nowhere.

- `newId()` — clock-seeded plus a random tail, replacing `max(id)+1`. The old
  scheme only made ids unique *within one device's row*, so two devices offline
  at once minted the same id for different items. Numeric on purpose: ids are
  interpolated into inline handlers and are foreign keys across apps, so
  changing their type would be a coordinated multi-repo migration.
- `nextId` is **deleted**, not kept alongside. A minting site missed in a future
  edit is then a loud `ReferenceError` rather than silent sequential collisions.
  This caught a real self-shadowing bug in restock the day it shipped.
- `_u` — timestamp of the last user-initiated content change. **Never set by
  load-time normalization** (rollover, dedupe, migrations, backfills). If a
  plain reload restamped everything, the last device to open the app would win
  every future merge, which is the original data-loss bug in a new costume.
- `settings._tomb` — deletion tombstones. Absence is never a delete without one.
  Lives in `settings` because every app version round-trips that column
  untouched; a new column would come back NULL from an older build.
- `settings._sync = {v:2, w:deviceId, at:stamp}` — protocol marker. A future
  merge must refuse to merge any row whose `_sync.v` is missing or < 2. That
  gate is what makes mixed versions safe while one device runs older code.
- `settings.timerAt` (time-tracker's row) — stamped at every timer decision. The
  timer is a mutex, not a collection, and **must never be merged**: that stamp
  is the only way a stopped timer can beat a running one, since absence cannot
  win on recency by itself.

**Phase B — not started.** Write `mergeRow()` as a pure function, wire it to a
dry run that logs the diff it *would* have produced, and discard the result.

**Phase C — not started.** Turn merge on. The first phase that can lose or
resurrect data. Do not start it until both devices have been running Phase A for
a while — a device on the old build deletes without leaving a tombstone, and a
merge that trusted absence would resurrect what it deleted.

**Open gap found during Phase A:** agenda's `ensureMonthlyRecurring` creates
this month's copy of a monthly task when one is missing. Two devices both
offline at the start of a month each create their own with different ids, and
Phase C's merge would keep both → duplicate monthly tasks. Harmless today.
Needs a deterministic identity (id derived from `seriesId + month`), handled in
Phase B while merge is still a dry run.

This sync block is copied between apps in the suite. Apps carrying it: agenda,
restock, time-tracker. Copy the corrected version, not an older sibling.
