# Calculator Logic — Pricing Math Spec

This doc describes the exact math the calculator playground runs, mirroring the semantics of
[tokyo/app/services/pricing.service.js](https://github.com/fynd-commerce/fyndconsole/blob/master/services/tokyo/app/services/pricing.service.js)
in the console.

## Inputs (calculator state)

```
state = {
  planId:    "plan_starter" | "plan_growth" | "plan_scale" | "plan_enterprise",
  billing:   "monthly" | "annual",
  country:   "IN" | "US",
  addons:    Set<productId>,           // addon products turned on
  usage:     { metricSlug → number }   // slider values
}
```

## Compute quote

For all tiers except Enterprise (which short-circuits to "Book a demo"):

```
plan = catalog.plans[state.planId]
priceRow = plan.prices.find(p =>
              p.country === state.country &&
              p.billingInterval === state.billing)

quote.baseMonthly =
  (state.billing === "annual")
    ? priceRow.unitPriceAmount / 12
    : priceRow.unitPriceAmount

quote.addonsMonthly = Σ for each addonId in state.addons:
    productPricing[addonId].prices.find(matching country & "month").unitPriceAmount

quote.metricOverageMonthly = Σ for each metric where unitSellPrice is set
                              AND state.usage[metric.slug] > planMetric.value:
    (state.usage[metric.slug] - planMetric.value) * metric.unitSellPrice
  // e.g. POS devices: beyond 1 included, ₹2,500/mo each

quote.subtotalMonthly = baseMonthly + addonsMonthly + metricOverageMonthly

if (state.billing === "annual") {
  quote.annualSubtotal = quote.subtotalMonthly * 12
  quote.annualDiscountPct = catalog.discounts.annual.amount   // 20
  quote.annualTotal = quote.annualSubtotal * (1 - 20/100)
  quote.effectiveMonthly = quote.annualTotal / 12
  quote.annualSavings = quote.annualSubtotal - quote.annualTotal
}
```

## Transaction fee (informational)

Not part of subtotal — shown as a separate estimate because actual cost depends on merchant GMV:

```
txEstimateMonthly = state.usage["commerce_b2c_orders"]
                  × metric.assumedAvgOrderValueINR   // 1000 INR by default
                  × plan.transactionFee.b2c / 100
```

Example: 1,000 orders/mo · ₹1,000 AOV · 2.0% (Growth) = ₹20,000/mo in transaction fees.
This is intentionally big and visible so merchants understand volume economics.

## Range hero band

Computed once on catalog load:

```
range.min = min(
  plan.prices.find(country, "month").unitPriceAmount
  for plan in plans where type === "standard" && isActive
)
// Cheapest paid plan, no add-ons, no overage

range.max = scalePlan.prices.find(country, "month").unitPriceAmount
          + Σ productPricing[p].prices.find(country, "month").unitPriceAmount
            for p in productPricing where commonlyPairedWith.includes("plan_scale")
          + Σ metric.unitSellPrice * metric.typicalHighEnd
            for metric in metrics where unitSellPrice exists
// Scale tier + commonly-paired add-ons + typical high-end usage.
// Explicitly NOT theoretical max — we want honest numbers.
```

The dot's position:

```
position = (log(quote.subtotalMonthly) - log(range.min))
         / (log(range.max) - log(range.min))       // 0..1
```

Log scale so a small configuration doesn't collapse against the left edge when max is 20× min.

## Enterprise tier

```
if (state.planId === "plan_enterprise") {
  show: "Custom pricing · Talk to us"
  hide:  monthly/yearly number, overage line items
  CTA:   same Book-a-demo handler as the top-right nav button
}
```

## Shareable URL

State → base64-encoded URL hash:

```
encoded = btoa(JSON.stringify({
  p: planId.replace("plan_", ""),   // "starter" etc.
  b: billing === "annual" ? "a" : "m",
  c: country,
  a: Array.from(addons).map(strip "prod_"),
  u: Object.fromEntries(Object.entries(usage).map([k, v] => [short(k), v]))
}))
location.hash = `calc=${encoded}`
```

On load, decode and hydrate state.

## Divergences from console's pricing.service.js (v1)

- No FX conversion — only currencies explicitly in the JSON work.
- No discount codes / promo validation.
- No "round to ending 9" price cosmetics.
- No max-discount cap (console caps at 25%; we only apply the static annual 20%).

These can be added later without schema changes.
