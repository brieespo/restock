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
| products | array of product objects (below) |
| settings | category list, store list, snooze defaults, theme |

### Product object

```js
{
  id: 1,
  name: "Daily Moisturizing Lotion",
  brand: "CeraVe",
  category: "bath_body",        // bath_body | makeup | household | food | health | pet | other (user-extendable)
  size: "19 oz",
  where_to_buy: [{store: "Costco", price: 11.49, url: ""}],  // multiple stores, price per store
  status: "stocked",            // 'stocked' | 'running_low' | 'out' | 'ordered'
  purchases: [                   // the heart of the app — every purchase logged
    {date: "2026-05-02", price: 11.49, store: "Costco", size: "19 oz"}
  ],
  interval_override_days: null, // manual interval if desired; otherwise computed
  substitutes: "Cetaphil in a pinch",  // acceptable backup when the real one is unavailable
  rating: null, notes: "",      // catalog job: why this one, shade names, dupes tried
  discontinued_watch: false     // stock up whenever seen (for products being phased out)
}
```

### The interval engine (computed, the app's requirements-engine equivalent)

- With 2+ purchases: predicted run-out = last purchase date + **median** gap between purchases. The app learns the real pace; no manual guessing.
- With 1 purchase (or by preference): `interval_override_days`.
- A product surfaces on the radar when *either* signal fires, and the UI distinguishes them: "you marked this low" vs. "you usually rebuy around now."
- **Feedback loop:** "still fine, snooze 2 weeks" stretches the interval estimate slightly; "ran out early" shrinks it. Accuracy improves with normal use, including lazy use.
- **"Bought it" is one tap** — today's date, last price/store/size prefilled, editable. Frictionless logging is non-negotiable; it is the lesson of every abandoned spreadsheet.

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

## Open items (confirm with Bri)

1. Rough product count to expect (dozens vs. hundreds — affects whether category filters suffice or search must lead).
2. Food scope: dinner planner owns weekly groceries. Suggested default — only non-weekly staples here (the specific olive oil, Sloane's snacks) to avoid double entry. Confirm.
3. Store list to seed (Costco, Target, Kroger, Amazon…?).
