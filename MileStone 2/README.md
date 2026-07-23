# Delivery Performance & Inventory Analysis

This repository contains two Power BI / Tableau-style dashboards built on order, shipment, and inventory data:

- **Delivery Performance Dashboard** — tracks on-time vs. late delivery behavior across shipping modes and regions.
- **Inventory Analysis Dashboard** — tracks stock health, turnover, and sales velocity across warehouses and product categories.

---

## 1. Inventory Turnover Calculation Approach

Inventory turnover measures how many times inventory is sold and replaced over a given period. It is the primary efficiency KPI on the Inventory dashboard (currently **38.56**).

**Formula used:**

```
Inventory_Turnover_Ratio = DIVIDE([Total_Sales], [Avg_Inventory_Value],0)

Avg_Inventory_Value = AVERAGE(Fact_Table[inventory_value])
```

**Implementation notes:**
- `Total_Sales` (from the *Sales by Category* visual) is used as the numerator proxy where COGS is not directly available.
- `Sum of stock_qty` (from the *Stock Quantity vs Reorder Level* visual) is used to derive average inventory on hand per product/warehouse.
- The ratio is calculated at the **overall** level for the headline KPI card, and can be sliced by `Warehouse Name`, `category_name`, or `product_name` for drill-down analysis.
- A higher ratio indicates inventory is selling and being replenished quickly; a lower ratio signals overstocking or weak demand.

**Interpretation guide:**

| Turnover Ratio | Interpretation |
|---|---|
| High (> 20–30) | Fast-moving, efficient inventory — but check for stockout risk |
| Moderate (5–20) | Healthy balance between supply and demand |
| Low (< 5) | Overstocked / at risk of becoming dead or slow-moving stock |

---

## 2. Slow-Moving and Fast-Moving Inventory Identification Logic

Stock is classified into three statuses, shown in the *Inventory Distribution by Stock Status* donut chart:

| Status | Logic |
|---|---|
| **Active (Fast-Moving)** | `stock_qty` turns over within a short lookback window (e.g., last 30–60 days) relative to average daily sales; `Sum of stock_qty` stays close to `Sum of reorder_level`, indicating regular replenishment. |
| **Slow-Moving** | Units have not sold within a defined threshold period (e.g., 90 days) but have moved at least once within a longer window (e.g., 180 days). Flagged when sales velocity is low relative to `stock_qty` on hand. |
| **Dead Stock** | No sales recorded within an extended window (e.g., 180+ days); `stock_qty` sits well above `reorder_level` with no depletion trend. |

**Classification rule (pseudo-logic):**

```
Stock Status = VAR DaysIdle = Fact_Table[Days_Since_Last_Sale]
VAR Qty = Fact_table[stock_qty] RETURN
SWITCH (    TRUE(),     Qty = 0, "Out of Stock",
    DaysIdle > 90 && Qty > 0, "Dead Stock",
    DaysIdle > 30 && DaysIdle <= 90, "Slow-Moving",    "Active")

```

**Current results from the dashboard:**
- **Dead Stock:** 96.58% of inventory (6M units) — the overwhelming majority of stock
- **Active:** 2.37%
- **Slow-Moving:** 1.05% (152K units)

The *Stock Quantity vs Reorder Level* chart compares `Sum of stock_qty` against `Sum of reorder_level` per SKU — products where stock on hand is far above the reorder point (e.g., Nike Men's Dri-FIT, Nike Men's CJ Elite) are the primary contributors to dead/slow-moving classification, since they are not depleting toward their reorder threshold.

---

## 3. Delivery Performance Analysis Methodology

The Delivery dashboard evaluates fulfillment reliability by comparing **scheduled vs. actual** shipping performance.

**Core metrics:**

| Metric | Formula |
|---|---|
| `On_Time_Delivery%` | DIVIDE(CALCULATE(DISTINCTCOUNT(Fact_Table[order_id]),Fact_Table[delivery_status]="Shipping on Time"),[Total_Orders],0) |
| `Late_Delivery%` | DIVIDE(CALCULATE(DISTINCTCOUNT(Fact_Table[order_id]),Fact_Table[delivery_status]="Late Delivery"),[Total_Orders],0) |
| `Advance_Shipping%` | DIVIDE(CALCULATE(DISTINCTCOUNT(Fact_Table[order_id]),Fact_Table[delivery_status]="Advance Shipping"),[Total_Orders],0) |
| `Avg_Days_For_Shipping (Real)` | AVERAGE(Fact_Table[days_for_shipping_(real)]) |
| `Avg_Days_For_Shipment (Scheduled)` | AVERAGE(Fact_Table[days_for_shipment_(scheduled)]) |
| `Late Delivery Risk %` | DIVIDE([Total_Risk_Flagged_Orders],[Total_Orders],0) |

**Current results from the dashboard:**
- **On-Time Delivery:** 17.1% (17.84% of order mix)
- **Late Delivery:** 51.2% (53.37% of order mix) — the majority outcome
- **Advance Shipping:** 27.6% (28.79% of order mix)
- **Late Delivery Risk:** 51.22%
- **Total Orders analyzed:** ~7K
- All shipping modes (Second Class, Standard Class, First Class) show actual shipping days consistently **exceeding** scheduled days, with Same Day showing the smallest promise-to-actual gap.
- Late delivery rate is elevated fairly uniformly across all top regions (Northern Europe through South of USA), suggesting a **systemic/carrier-level issue** rather than a region-specific one.

---

## 4. Key Insights & Business Recommendations

### Delivery Performance

**Insights:**
- Only **17.1%** of orders are delivered on time, while **51.2%** arrive late — late delivery is the *dominant* outcome, not the exception.
- Every shipping mode shows actual shipping days running longer than scheduled, meaning the SLA promise itself may be miscalibrated or the fulfillment process is under-resourced.
- Late delivery is spread broadly across regions rather than concentrated in one geography, pointing to a network-wide capacity or carrier problem.

**Recommendations:**
- **Re-baseline SLA commitments** using actual historical shipping times per mode, rather than optimistic targets that are consistently missed — this improves customer trust even before operational fixes land.
- **Audit carrier performance** for the shipping modes with the widest actual-vs-scheduled gap (Second Class and Standard Class appear widest) and consider renegotiating contracts or adding backup carriers.
- **Investigate root cause of the systemic late trend** (warehouse dispatch delays, carrier handoff times, customs/regional logistics) since the issue affects nearly all regions similarly.
- **Prioritize Same Day / First Class capacity** since these already track closer to promise — study what's working there and replicate it for slower tiers.

### Inventory

**Insights:**
- **96.58% of inventory is classified as dead stock** — this is the single most urgent issue in the dataset, representing significant tied-up capital and warehouse space.
- Inventory value is heavily concentrated in a few warehouses (L3 and L1 dominate the *Inventory Value by Warehouse* chart), creating both a capital-concentration risk and a potential single point of failure.
- Several top products (e.g., Nike Men's Dri-FIT, Nike Men's CJ Elite) show `stock_qty` far exceeding `reorder_level`, indicating over-ordering relative to actual demand.
- Despite the dead stock problem, the overall turnover ratio (38.56) looks healthy — suggesting turnover is being driven by a small subset of genuinely fast-moving SKUs while the bulk of inventory sits idle.

**Recommendations:**
- **Launch a dead-stock clearance initiative** (discounting, bundling, liquidation, or return-to-vendor) to free up capital and warehouse space, starting with the highest-value dead-stock SKUs.
- **Recalibrate reorder levels** for products where `stock_qty` consistently overshoots `reorder_level`, to prevent the current dead stock problem from recurring.
- **Rebalance warehouse allocation** — investigate why L3/L1 hold disproportionate inventory value and whether redistributing stock closer to demand (aligned with regions showing best delivery performance) would reduce both holding cost and delivery time.
- **Segment purchasing decisions by category performance** — the *Sales by Category* chart shows Cleats and Cardio Equipment leading sales; align future purchasing/replenishment more closely with this demand signal rather than uniform stocking across categories.
- **Set up an early-warning slow-moving flag** (e.g., no sale in 60 days) so items are caught and acted on before they age into dead stock.

---

## Data Sources & Fields Referenced
- `order_date`, `order_region`, `shipping_mode`, `Avg_Days_For_Shipping (Real/Scheduled)`, `Late_Delivery%`, `On_Time_Delivery%`, `Advance_Shipping%`
- `stock_qty`, `reorder_level`, `category_name`, `product_name`, `Warehouse Name`, `inventory_value`, `Dead_Stock_Quantity`, `Slow_Moving_Quantity`, `Inventory_Turnover_Ratio`

## Notes
- Percentages in the donut charts are shown two ways: as a % of the three-metric total (Late/On-Time/Advance) and as a % of overall records — both are reported above for clarity.
- Thresholds used for slow-moving/dead-stock classification (30/90/180-day windows) should be adjusted to match the specific business's typical sales cycle and replenishment lead times.
