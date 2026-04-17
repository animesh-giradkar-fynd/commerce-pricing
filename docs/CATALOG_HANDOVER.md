# Catalog Handover — `data/catalog.json`

## TL;DR

1. **`data/catalog.json` is mockup data.** It's tagged `meta.source: "sheet-mockup-v1"`.
2. **Eventual source of truth = Fynd Console endpoints** (tokyo + charmander). The sheet snapshot goes away.
3. The JSON schema is a deliberate mirror of console's wire format, so the swap is drop-in with no UI refactor.

## Where the numbers came from (v1)

`catalog.json` was derived from `data/pricing-sheet.json`, which is a 94-row machine-readable
extract of `Proposed Commercial for Commerce.xlsx` (the internal pricing master owned by
Fynd's Pricing Ops).

Confirmed sheet values used today:

| | Free | Starter | Growth | Scale | Enterprise |
|---|---|---|---|---|---|
| Subscription (annual INR) | 0 | 11,111 | 22,222 | 33,333 | Custom |
| B2C Transaction Fee % | 0% | 2.5% | 2.0% | 1.5% | 0.25% |
| Fulfilment Locations included | — | 10 | 10 | 10 | Custom |
| POS Per Device Subscription | — | ₹2,500/mo · $30 | same | same | same |

## Why the sheet isn't the long-term source

- It lives in Google Drive, owned by Pricing Ops — not engineering. It drifts the moment
  console plan/price rows change and there's no enforcement loop.
- Console (tokyo + charmander) is where plans are actually provisioned, metered, and invoiced.
  The pricing page showing different numbers than what a merchant is billed is a credibility bug.
- A live fetch (or nightly export) means marketing pricing updates ship without a pricing-page
  code push.

## Future: switching to console as source of truth

Two candidate paths, either works — the UI doesn't care:

### Option A — Public CORS-safe endpoint
When the platform team exposes a CORS-friendly, unauthenticated (or server-token-auth) version of:
- `GET tokyo /v1/plans?include=prices,metrics` → drives `plans[]`, `plans[].prices[]`, `plans[].planMetrics[]`
- `GET charmander /v1/products?include=metrics` → drives `products[]` and `metrics[]`

Then the pricing page swaps `fetch('./data/catalog.json')` for these endpoints directly and
a thin client-side adapter normalises the response. Flip `meta.source` to `"console-live"`.

### Option B — Nightly export job
Easier politically: a scheduled job (GitHub Action / Vercel cron / small Cloud Function)
hits the authenticated console APIs with a service token, runs the same adapter, and commits
the resulting `catalog.json` back to this repo. Vercel redeploys automatically.

## Field-level mapping (for the future swap)

Every `catalog.json` field has a direct console source so the adapter is mechanical:

| `catalog.json` path | Console source | File reference |
|---|---|---|
| `plans[]._id`, `.name`, `.description`, `.type` | `tokyo.plans._id / name / description / type` | [tokyo/app/server/models/v2/plans.model.js](https://github.com/fynd-commerce/fyndconsole/blob/master/services/tokyo/app/server/models/v2/plans.model.js) |
| `plans[].prices[]` | `tokyo.prices` (filter by `planId`) | [tokyo/app/server/models/v2/prices.model.js](https://github.com/fynd-commerce/fyndconsole/blob/master/services/tokyo/app/server/models/v2/prices.model.js) |
| `plans[].planMetrics[]` | `charmander.planMetrics JOIN productMetrics` | [charmander/app/server/repositories/planMetrics.ts](https://github.com/fynd-commerce/fyndconsole/blob/master/services/charmander/app/server/repositories/planMetrics.ts) |
| `products[]` | `charmander.products` | [charmander/app/server/repositories/products.ts](https://github.com/fynd-commerce/fyndconsole/blob/master/services/charmander/app/server/repositories/products.ts) |
| `metrics[]` | `charmander.productMetrics` | [charmander/app/server/repositories/productMetric.ts](https://github.com/fynd-commerce/fyndconsole/blob/master/services/charmander/app/server/repositories/productMetric.ts) |
| `metrics[].unitSellPrice` | `productMetrics.unitSellPrice` | same file, `unitSellPrice` column |
| `discounts.annual.amount` | `tokyo/app/services/pricing.service.js` → `PRICING_YEARLY_DISCOUNT_PCT` | hardcoded 20 in console today |

## What to update in `catalog.json` today (while in mockup mode)

When Pricing Ops updates a number in the xlsx:

1. Re-run the extraction: open the xlsx, export `/tmp/commerce_pricing_sheet.json` via the same
   `openpyxl` script used to seed `data/pricing-sheet.json`.
2. Hand-edit the affected fields in `data/catalog.json` (keeping the schema stable).
3. Commit. Vercel auto-deploys.

This process becomes obsolete the moment Option A or B above lands.

## Fields we added that don't exist in console

Two UI-only fields not backed by a console column — both are v1 conveniences that Pricing Ops
controls via the JSON, kept here for honesty:

- `productPricing[].commonlyPairedWith: [planId…]` — drives the "realistic ceiling" for the
  range hero. If the console export omits this, default to `["plan_scale"]` for all add-ons.
- `metrics[].typicalHighEnd` — the p90-ish slider value used in the same ceiling calc.
  If omitted, default to `maximumValue / 3`.
