# Nexa Steel — Product Intelligence Cockpit

A self-contained, single-file analytics cockpit that turns three years of steel mill sales into an executive-ready read: realised net/gross/MRP price, SKU momentum, margin structure, competitor mill rates, market share, dealer commissions, regional growth and price elasticity.

## The problem it solves
A steel business unit (rebar/MS rod) generates granular order-line data, but leadership had no fast way to see *what price is doing, where margin is leaking, which SKUs are gaining or losing momentum, and how we stack up against the market* without waiting on analyst reports. Decisions on pricing, promotions and share defence were being made on stale, fragmented numbers.

## The solution approach
One portable HTML file (no build, no server, no libraries) that embeds the entire dataset as a JS object and hand-rolls every chart as inline SVG. A tabbed narrative walks a reader from the overview down to per-SKU elasticity, with a "what this answers / how to read it" guide box on every section.

## Key features
- Overview, Price, GP/NP/Net margin, All-SKU, SKU month-on-month, Declared-price grid, Elasticity, Competitor, Market-share, Dealer, Area, Growth and Direction·Decision·Action tabs.
- Net vs gross vs MRP price per SKU across a clean 23-month window, with declared (mill-rate) vs realised gap analysis.
- Own-price elasticity computed live via log-log OLS (Δln volume on Δln price) with per-SKU β and R².
- Competitor mill-rate survey ranking and multi-mill trend comparison.
- Market-share progression, fastest/slowest SKU momentum, and revenue-bridge views.
- Sortable, searchable tables with click-to-drill and a shared hover tooltip on every chart.
- A built-in "sources, method & coverage" section documenting caveats (partial survey windows, derived volumes, mixed-unit exclusions).

## Tech stack
- Vanilla JavaScript (ES5-style), zero runtime dependencies.
- Hand-authored SVG for all charts/axes; CSS grid + custom properties for layout and theming.
- Data embedded as a single `var D = {...}` object literal; derived series computed at load.

## Demo data
All figures are synthetic placeholders. Revenue, price, volume, competitor rates, market-share and elasticity values have been randomized from the original patterns while preserving structure, record counts and order of magnitude so every chart and KPI still renders. Company, competitor and ERP identifiers are fictional.

## How to run locally
- Open `index.html` directly in a browser, **or**
- Serve statically, e.g. `npx serve .` then visit the printed URL.

## Deploying to Vercel
1. Push this folder to a Git repo.
2. In Vercel, **Import repo** → framework preset: **Other**.
3. Build command: leave blank; Output directory: leave as root (or `public`).
4. Deploy — the static `index.html` is served as-is.
