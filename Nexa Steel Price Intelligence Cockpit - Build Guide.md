# SBU Price Intelligence Cockpit — Build Guide (Skill)

A reusable runbook for building a **"Price & Volume History Cockpit"** — a self-contained HTML report that shows, per SKU: price & volume history, price-direction breadth, own-price elasticity, and price changes — pulled from the live ERP.

---

## 1. What this report answers

1. **Where has price gone** — first → last realised price per SKU, with % change.
2. **Is price rising or falling across the portfolio** — "breadth" (share of SKUs up vs down, month-on-month).
3. **How volume responds to price** — own-price elasticity (elastic / inelastic / anomalous).
4. **Which SKUs drive revenue** — sortable revenue / volume / price table.
5. **What changed when** — per-SKU month-on-month drill-in (volume, revenue, realised price, MRP).

---

## 2. Data sources (the ERP)

> All queries scoped to one Business Unit via `intBusinessUnitId = <business-unit-id>` (the gateway also auto-scopes).

| Metric | Table | Key columns |
|---|---|---|
| Volume | `sales.order_line` | `numOrderQuantity` |
| Net revenue | `sales.order_line` | `numNetValue` |
| Gross value | `sales.order_line` | `numOrderValue` |
| **Realised price** | derived | `numNetValue ÷ numOrderQuantity` |
| **Gross price** | derived | `numOrderValue ÷ numOrderQuantity` |
| **MRP / list price** | `sales.order_line` | `numItemPrice` |
| Declared DP / TP / MRP | `pricing.retail_price` + `pricing.retail_price_log` | `numDPRate`, `numTPRate`, `numMrpRate` |
| **Declared Trade / mill rate** | `catalog.price_setup` | `intConditionReffId` (channel), `numPrice`, `dteStartDate`, `dteEndDate` |
| SKU master / UOM | `catalog.item` | `strItemName`, `strItemCategoryName`, `strItemTypeName` |
| Sales date | `sales.order_header` | `dteSalesOrderDate` |

**Join pattern (avoid BU-column ambiguity):**
```sql
FROM sales.order_line r
JOIN (SELECT intSalesOrderId, dteSalesOrderDate dt FROM sales.order_header) h
  ON h.intSalesOrderId = r.intSalesOrderId
```
Put the header in a derived table that projects only `intSalesOrderId` + date — this keeps the gateway's injected `intBusinessUnitId IN (...)` unambiguous.

---

## 3. The SQL pulls (5)

**3a. Monthly revenue & volume per SKU (pivoted, one row per month):**
```sql
SELECT YEAR(h.dt) yr, MONTH(h.dt) mo,
  SUM(CASE WHEN r.strItemName='<SKU>' THEN r.numNetValue ELSE 0 END) rev,
  SUM(CASE WHEN r.strItemName='<SKU>' THEN r.numOrderQuantity ELSE 0 END) qty
FROM sales.order_line r
JOIN (SELECT intSalesOrderId, dteSalesOrderDate dt FROM sales.order_header) h ON h.intSalesOrderId=r.intSalesOrderId
WHERE r.intBusinessUnitId=<business-unit-id> AND YEAR(h.dt)>=<start>
GROUP BY YEAR(h.dt), MONTH(h.dt) ORDER BY yr, mo
```

**3b. Monthly net price per SKU:**
```sql
SELECT YEAR(h.dt) yr, MONTH(h.dt) mo,
  CAST(SUM(r.numNetValue)/NULLIF(SUM(r.numOrderQuantity),0) AS INT) price
FROM ... GROUP BY YEAR(h.dt), MONTH(h.dt)
```

**3c. Monthly gross price per SKU:** same as 3b but `numOrderValue`.

**3d. Monthly MRP per SKU:**
```sql
SELECT YEAR(h.dt) yr, MONTH(h.dt) mo, CAST(AVG(r.numItemPrice) AS INT) mrp
FROM ... GROUP BY YEAR(h.dt), MONTH(h.dt)
```

**3e. Declared DP/TP/MRP (only if the SBU uses retail-price config):**
```sql
SELECT strItemName, strDistributionChannelName, numItemRate, numTPRate, numDPRate, numMrpRate
FROM pricing.retail_price WHERE intBusinessUnitId=<business-unit-id>
```
Log of historical changes: `pricing.retail_price_log` (`dteLogEntryDate`).

**3f. Declared Trade / mill rate (for mills — e.g. rebar/steel):**
```sql
SELECT dc.strDistributionChannelName ch, CAST(ps.dteStartDate AS DATE) d, MIN(ps.numPrice) p
FROM catalog.price_setup ps
JOIN catalog.item i ON i.intItemId = ps.intItemId
LEFT JOIN sales.distribution_channel dc ON dc.intDistributionChannelId = ps.intConditionReffId
WHERE ps.intBusinessUnitId=<business-unit-id> AND i.strItemCategoryName='<category>'
GROUP BY dc.strDistributionChannelName, CAST(ps.dteStartDate AS DATE)
ORDER BY d
```
`catalog.price_setup` is a **daily-updated** rate history — collapse it to change-events (price differs from the previous row) for the "declared changes" tracker, and to month-end for the trend chart.

---

## 4. Computation

- **Realised price** = net value ÷ quantity (per selling unit).
- **Volume** = net revenue ÷ realised price (keeps unit consistent), or use `numOrderQuantity`.
- **MoM price change** = `p[i]/p[i-1] − 1`, compared to the previous month *with sales* for that SKU.
- **Breadth** per month = count of SKUs with price up vs down.
- **Elasticity** = slope of `ln(volume)` on `ln(price)` (log-log OLS):
  ```
  β = (n·Σxy − Σx·Σy) / (n·Σx² − (Σx)²),  where x=ln(P), y=ln(Q)
  ```
  Report only when ≥18 valid months and |β| ≤ 8. β < −1 elastic, −1…0 inelastic, >0 anomalous.
- **R²** = correlation² of ln(P), ln(Q).
- **Realised ÷ MRP** — outside 0.35–1.6 flags a unit mismatch (booked unit ≠ declared pack).

---

## 5. File structure (tabs)

Two tiers of tabs are combined in the full cockpit:

- **Price & elasticity core** (follows the Nexa Essentials reference): `Overview` (KPIs → portfolio → breadth → insights → journey) · `All SKUs` (sortable/searchable table) · `SKU month-on-month` (picker + dual-axis chart + table) · `Price change tracker` (realised moves + declared changes) · `Declared price grid` (declared rate trend + realised-vs-declared gap + grid) · `Elasticity` (KPIs + bar + scatter + table) · `Method & coverage`.
- **Business analysis sections** (the SBU-unique add-ons): `Price` (net/gross/MRP by SKU) · `GP/NP/Net` (waterfall + raw-material proxy) · `Competitor` (survey) · `Market Share` · `Dealer` (commission) · `Area` (delivery zones) · `Growth` (yearly revenue/volume).

Every section opens with a `.guide` box ("What this answers" / "How to read it") so a first-time reader can navigate. All charts are hand-rolled SVG (no library) with a shared hover tooltip; tables are sortable/searchable with click-to-drill.

---

## 6. To adapt for another SBU

1. Set `BU` = the ERP business-unit id (e.g. the mill BU, the cement BU, the essentials BU).
2. Set the finished-product filter (`strItemTypeName = 'Finished Products'`, or the relevant item names / category).
3. Re-run §3 pulls, paste results into the HTML `D`/`P`/`MRP`/`REV` objects.
4. Update the header, sub, badge, and the coverage table.
5. Verify the **clean price window** (see coverage).

---

## 7. Coverage / gotchas

- **DP / TP / MRP** — the retail DP/TP/MRP ladder (`pricing.retail_price`) is only populated for consumer BUs (each business unit has its own numeric id). **Mills use a Trade/mill rate instead** — look in `catalog.price_setup` (for the mill: the Trade and Sister-concern channels). Rebar has no separate distributor (DP) tier, so the "ladder" collapses to **Trade → MRP**, and MRP sits almost on top of Trade.
- **COGS / production cost** — often not posted monthly (empty cost fields, empty cost sheet, GL has only revenue). Use the item avg-cost field as a point-in-time proxy, or scrap purchase cost as a monthly raw-material proxy.
- **Mixed-unit quantity** — check per-ton price stability; early years often have kg/MT mix (e.g. Nexa Steel 2023–24). Start the "clean window" from when per-ton price is consistent.
- **500-row gateway cap** — aggregate/pivot in SQL; never pull raw rows.
