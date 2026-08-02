# Supplier Performance & Transportation Analysis

This repository contains two Power BI dashboards built on `Dim_supplier` and `Fact_table`:

- **Supplier Scorecard Generation** — ranks and segments suppliers by quality, reliability, and lead-time performance.
- **Transportation Analysis** — tracks discounting, profitability, and delivery reliability across shipping modes, markets, and regions.

---

## 1. Supplier Composite Scoring Approach

Supplier performance is consolidated into a single ranking metric — the **Supplier Composite Score** — so suppliers can be compared on one scale instead of three separate, sometimes conflicting, KPIs (quality, reliability, speed).

**Formula used:**
```
Supplier Composite Score =
    ( quality_score * 0.4 )
  + ( reliability_% * 0.4 )
  + ( ( 30 - lead_time_(days) ) / 30 * 100 * 0.2 )

Supplier Rank = RANKX(ALL(Dim_supplier), [Supplier Composite Score], , DESC)
```

**Implementation notes:**
- Quality and Reliability are weighted equally (40% each) since both directly affect downstream order fulfillment quality.
- Lead Time is inverted and normalized against a 30-day baseline before being weighted at 20% — a supplier with 0-day lead time scores the full 20 points on this component, a 30-day (or slower) supplier scores 0. This keeps a fast, mediocre-quality supplier from disproportionately jumping the rankings on speed alone.
- Rank is computed with `ALL(Dim_supplier)` so it stays stable regardless of report-level slicers (Supplier Name / Product Name filters don't change each supplier's underlying rank position).

**Current results from the dashboard:**
- Top 5 suppliers by composite score: **ABC Industries, John Deere, Global Supplies, Weidmüller, SABIC** — these lead specifically because they combine above-average reliability and quality with shorter lead times, not because they excel on any single dimension alone.
- Overall average lead time across all 118 suppliers: **17.31 days**.
- Overall average quality score: **75.71**, average reliability sits noticeably lower at the aggregate level — suggesting reliability is the weaker leg of the three-factor score for most suppliers.

---

## 2. Reliability Tier Classification Logic

Each supplier is bucketed into a tier so procurement can quickly triage risk without reading raw percentages.

```
Reliability Tier =
SWITCH(
    TRUE(),
    reliability_% >= 80, "High",
    reliability_% >= 50, "Medium",
    "Low"
)
```

| Tier | Threshold | Interpretation |
|---|---|---|
| **High** | ≥ 80% | Dependable — safe to concentrate volume/orders here |
| **Medium** | 50–79% | Acceptable but should be monitored, not over-relied on for critical orders |
| **Low** | < 50% | At-risk — consider dual-sourcing or supplier improvement plans |

**Current results from the dashboard (Supplier Tier Distribution):**
- **Low:** 46.61% of suppliers — the single largest segment
- **Medium:** 30.51%
- **High:** 22.88%

This is the most important finding on the Supplier dashboard: **nearly half of all suppliers fall into the "Low" reliability tier**, and High-reliability suppliers are the smallest group. Cross-referencing with the *Lead Time by Supplier Tier* chart, tier-level lead times cluster fairly close together (15–18 days across tiers), suggesting lead time alone does not explain why a supplier lands in the Low reliability bucket — the driver is more likely inconsistency/variability in delivery rather than raw speed.

---

## 3. Delivery & Discounting Methodology (Transportation Analysis)

**Core metrics:**

| Metric | Formula |
|---|---|
| `Avg Profit Per Order` | `AVERAGE(Fact_table[order_profit_per_order])` |
| `Total Discount Given` | `SUM(Fact_table[order_item_discount])` |
| `Avg Discount Rate` | `AVERAGE(Fact_table[order_item_discount_rate])` |
| `Same Day Share %` | `DIVIDE(CALCULATE(DISTINCTCOUNT(order_id), shipping_mode="Same Day"), [Total Orders], 0)` |
| `Late Rate by Shipping Mode` | `CALCULATE([Late Delivery %], ALLEXCEPT(Fact_table, Fact_table[shipping_mode]))` |

**Implementation notes:**
- `Late Rate by Shipping Mode` uses `ALLEXCEPT` rather than a plain `CALCULATE`, so the late-rate calculation for each shipping mode ignores other filters in the report (e.g., date slicer, market slicer cross-filtering from other visuals) while still respecting the shipping-mode context itself — this keeps the bar chart accurate even when cross-filtered from other visuals on the page.
- `Same Day Share %` is deliberately measured as a share of *total orders*, not a share within shipping modes only, so it reflects genuine adoption of Same Day as a fulfillment option across the whole business, not just relative popularity among shipping choices.

**Current results from the dashboard:**
- **Avg. Profit per Order:** 21.20
- **Avg. Discount Rate:** 10.34%
- **Total Discount Given:** 206.32K
- **Same-Day Share:** 1.39% of total orders — Same Day is a minor share of fulfillment volume despite appearing as its own category across every shipping-mode visual.
- **Late Delivery Rate by Shipping Mode:** First Class shows the highest late rate, followed by Second Class, then Same Day, with Standard Class showing the lowest late rate.
- **Order volume mix:** Standard Class dominates at ~71.34% of total orders, Second Class ~17.16%, First Class ~10.11%, Same Day ~1.39%.

---

## 4. Key Insights & Business Recommendations

### Supplier Performance

**Insights:**
- **46.61% of suppliers sit in the "Low" reliability tier** — this is the dominant segment, not the exception, meaning supply risk is broad-based rather than isolated to a few outlier vendors.
- High-reliability suppliers are the *smallest* group (22.88%), so the business currently has limited "safe" capacity to shift volume toward if a Low-tier supplier fails.
- The top-ranked suppliers by composite score (ABC Industries, John Deere, Global Supplies, Weidmüller, SABIC) succeed by being well-rounded across quality, reliability, and speed — not by winning on one dimension alone, which validates the weighted composite-score approach over ranking by a single raw metric.
- Average lead time (17.31 days) is moderate and fairly consistent across tiers, which suggests low reliability is driven more by inconsistency/failure rate than by raw delivery speed — worth confirming with variance/on-time-rate data at the order level, not just averages.

**Recommendations:**
- **Prioritize a reliability-improvement or dual-sourcing plan for Low-tier suppliers**, since they represent close to half of the supplier base — this is a portfolio-level risk, not a handful of vendors to swap out individually.
- **Use the Composite Score, not single metrics, for sourcing decisions** — a supplier that looks attractive on lead time alone may be masking reliability risk that only shows up after the order is placed.
- **Study what the top 5 composite-score suppliers are doing differently** (contract terms, communication cadence, geographic proximity) and use it as a template when renegotiating with Medium/Low tier suppliers.
- **Set a minimum reliability threshold for new supplier onboarding** going forward, so the Low-tier segment doesn't keep growing as the supplier base expands.

### Transportation & Delivery

**Insights:**
- **First Class has the highest late-delivery rate** despite typically being marketed as a premium/faster service — this is a credibility risk if customers are paying more for First Class and receiving worse on-time performance than Standard Class.
- **Standard Class**, while the slowest-promised tier, shows the *most* consistent on-time performance and carries **71% of total order volume** — the business's delivery reputation is being shaped mostly by this one shipping mode, so any improvement effort here has outsized impact.
- **Same Day fulfillment is used in only 1.39% of orders** — either demand for it is genuinely low, or it isn't being actively offered/promoted at checkout; worth investigating which, since it's currently too small a sample to draw firm reliability conclusions from.
- An average discount rate of 10.34% combined with 206K in total discounts given suggests discounting is a routine, broad-based practice rather than a targeted promotional tool — worth checking whether discounts are contributing to or eroding the 21.20 average profit per order.

**Recommendations:**
- **Investigate First Class fulfillment specifically** — audit carrier assignment, warehouse pick priority, and SLA promises for this tier, since it's underperforming relative to both its price positioning and slower tiers.
- **Protect Standard Class performance as the priority lever** — because it carries the majority of volume, even a small late-rate improvement here moves the overall delivery metric more than fixing any other single tier.
- **Test increased Same Day visibility/promotion** in a controlled way and monitor whether reliability holds up as volume grows, before assuming it's a viable premium offering to scale.
- **Review discount policy against profit-per-order trends** — determine whether the current ~10% average discount rate is targeted (clearing slow stock, loyalty) or blanket, and whether tightening it in low-margin segments would lift average profit per order without hurting order volume.


