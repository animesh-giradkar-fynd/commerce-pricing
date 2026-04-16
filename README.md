# commerce-pricing

Fynd Commerce pricing page — standalone, single-file HTML application. Mobile-responsive from 360px through 1480px.

**Live source:** [`index.html`](index.html)

---

## Quick start

```bash
git clone https://github.com/animesh-giradkar-fynd/commerce-pricing
cd commerce-pricing
open index.html          # macOS
# or
python3 -m http.server   # then visit http://localhost:8000
```

No build step. No dependencies. No Node required.

---

## What's in here

A pricing **playground**: merchants pick the products they need from a left-side category nav and see plan cards, a feature comparison table, and add-ons for that selection. Customer Platform is the default panel.

- **10 panels** — Customer Platform, Free Tools, Create a Bundle, Fynd for SMBs, Storefront, Store OS / POS, Konnect, Managed Logistics, AI Studio, OMS / WMS
- **Billing toggle** — Monthly / Annual, swaps all price figures via `data-a` / `data-m` attributes
- **Compare table** — Accordion by category. On mobile, scrolls horizontally with a sticky first column.
- **Add-on cards** — Toggleable, six services (POS, AI Studio, Logistics, Commerce Automation, Kaily, Konnect Marketplaces)

---

## Mobile behavior

| Width | Layout |
|---|---|
| ≥992px | Sticky 240px sidebar, 4-col Customer Platform / 3-col products |
| 768–991px | Hamburger drawer (off-canvas), 2-col cards |
| 480–767px | Drawer, 2-col cards, table scrolls horizontally with sticky first column |
| ≤479px | Drawer, 1-col cards, compact billing bar, smaller H1 |

Drawer closes on: nav-item tap, backdrop tap, **Esc** key, or window resize past 991px.

---

## File layout

```
commerce-pricing/
├── index.html       # Single-file app — HTML + CSS + vanilla JS, no dependencies
├── README.md
└── .gitignore
```

Future phases (see internal docs) will split this into `src/` + `styles/` modules and replace the inline data with API calls.

---

## Contributing

The CSS uses BEM-ish naming with a `c-` / `js-` split where applicable. All JS functions are global and exported on `window` for easy console testing — `show('platform')`, `setBilling('monthly')`, `toggleAddon('pos')`, `toggleDrawer()`.

Browser support: Chrome 100+, Safari 15+, Firefox 100+, Edge 100+.

---

## License

Proprietary — Fynd. All rights reserved.
