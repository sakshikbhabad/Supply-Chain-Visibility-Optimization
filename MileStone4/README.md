# Supply Chain Visibility & Warehouse Efficiency Analysis

This repository contains two Power BI dashboards built on `Fact_table`,`Dim_warehouse`:

- **Executive Dashboard (Supply Chain Visibility and Optimization)** — consolidates order fulfillment, delivery reliability, inventory health, and sales trend performance into one company-wide view.
- **Warehouse Efficiency Dashboard** — tracks utilization, capacity risk, and stock distribution across individual warehouses.

---

## 1. Order Fulfillment & Delivery Reliability Methodology

Order-level health is tracked through two paired metrics — fulfillment and late delivery — so the dashboard reflects both whether orders are completed and whether they arrive on time.

**Formula used:**
```
Late Orders = CALCULATE ( DISTINCTCOUNT ( Fact_table[order_id] ), Fact_table[delivery_status] = "Late delivery" )
Late Delivery % = DIVIDE ( [Late Orders], [Total Orders], 0 )
Late Delivery % Target = 0.2

Fulfillment Rate = DIVIDE ( [Total Orders] - [Canceled Orders], [Total Orders], 0 )
```

**Implementation notes:**
- Late Delivery % is measured against a fixed 20% target line on the gauge, so any reading above the target marker is an immediate visual flag rather than something the viewer has to calculate mentally.
- Fulfillment Rate is calculated as a share of total orders (not shipped orders), so cancellations are penalized directly in the headline KPI rather than being excluded from the denominator.

**Current results from the dashboard:**
- **Total Orders:** 7,104
- **Fulfillment Rate:** 95.97% — comfortably healthy against typical benchmarks.
- **Late Delivery %:** 51.22% — more than half of all orders are arriving late, well above the 20% target, despite the high fulfillment rate. This is the clearest sign that the fulfillment metric alone hides a serious downstream delivery problem.

---

## 2. Supplier Reliability Methodology

Supplier reliability is surfaced as a single averaged gauge metric so procurement and operations can see, at a glance, how dependable the supplier base is as a whole.

**Formula used:**
```
Avg. Reliability % = AVERAGE ( Dim_supplier[Reliability %] )
```
Gauge axis max set to 107.80 to accommodate the highest recorded supplier reliability value in the dataset.

**Current results from the dashboard:**
- **Avg. Reliability %:** 53.90 — roughly half of maximum observed reliability, and the lowest-performing of the four gauge KPIs on the Executive Dashboard.
- Read alongside the 51.22% late delivery rate above, supplier reliability and delivery lateness track closely together, suggesting upstream supplier performance is a meaningful driver of the delivery problem rather than a purely logistics-side issue.

---

## 3. Inventory Health & Turnover Methodology

Inventory health is measured through two complementary metrics: how much stock is effectively dead weight, and how efficiently the remaining inventory converts into sales.

**Formula used:**
```
Dead Stock Quantity =
SUMX (
    FILTER (
        VALUES ( Fact_table[product_name] ),
        CALCULATE ( MAX ( Fact_table[Stock Status] ) ) = "Dead Stock"
    ),
    CALCULATE ( AVERAGE ( Fact_table[stock_qty] ) )
)

Total Inventory Value =
SUMX ( VALUES ( Fact_table[product_name] ), CALCULATE ( AVERAGE ( Fact_table[inventory_value] ) ) )

Inventory Turnover Ratio = DIVIDE ( [Total Sales], [Total Inventory Value], 0 )
```

**Implementation notes:**
- Both measures use `SUMX` over `VALUES(Fact_table[product_name])` rather than a plain `SUM`, since stock quantity and inventory value are stored at the product grain and would otherwise be double-counted across repeated order-line rows for the same product.
- Inventory Turnover Ratio is formatted as `0.00x` so it reads as a rate (times inventory is sold through) rather than a raw number.

**Current results from the dashboard:**
- **Dead Stock Quantity:** 45,605 units — a large absolute figure relative to total order volume (7,104 orders), indicating a meaningful share of inventory is not moving.
- **Inventory Turnover Ratio:** 0.47x — inventory is turning over less than once per period, reinforcing the dead stock signal: stock is accumulating faster than it's being sold through.

---

## 4. Warehouse Utilization & Capacity Risk Methodology

Warehouse-level utilization is ranked and segmented so operations can identify both over- and under-utilized sites, and flag capacity risk before it becomes a bottleneck.

**Formula used:**
```
Avg Utilization % = AVERAGE ( Fact_table[utilization_%] )

Max Utilization % =
MAXX ( VALUES ( Fact_table[warehouse_name] ), CALCULATE ( AVERAGE ( Fact_table[utilization_%] ) ) )

Min Utilization % =
MINX ( VALUES ( Fact_table[warehouse_name] ), CALCULATE ( AVERAGE ( Fact_table[utilization_%] ) ) )

Warehouse Utilization Target = 80

Capacity Risk Flag =
VAR Util = [Avg Utilization %]
RETURN
SWITCH ( TRUE(),
    Util >= 90, "Critical",
    Util >= 75, "Watch",
    "OK"
)

Critical Warehouses =
CALCULATE ( DISTINCTCOUNT ( Fact_table[warehouse_name] ), Fact_table[utilization_%] >= 90 )
```

**Implementation notes:**
- Min/Max Utilization use `MAXX`/`MINX` over `VALUES(Fact_table[warehouse_name])` so each warehouse is first collapsed to its own average utilization before ranking — this prevents a single high-utilization order row from skewing the warehouse-level max/min.
- The Capacity Risk Flag uses a three-tier `SWITCH(TRUE())` pattern rather than nested `IF`s, making it straightforward to add further tiers later without restructuring the logic.

**Current results from the dashboard:**
- **Total Warehouses:** 10
- **Lowest Utilization %:** 15.93% (Warehouse L9)
- **Highest Utilization %:** 84.24% (Warehouse L4)
- **Average Utilization %:** 55.15%
- **Utilization ranking (high → low):** L4, L3, L7, L0, L2, L5, L6, L1, L8, L9 — L4 alone sits above the 80% target, meaning it is the only warehouse currently at "Watch"/"Critical" risk while the rest of the network runs below target.
- **Capacity share** is concentrated in a few sites: L2 (36.39%), L5 (19.36%), L9 (17.19%), L6 (13.56%), L0 (13.51%) account for the visualized capacity utilization split.

---

## 5. Sales Trend & Forecasting Methodology

Sales are tracked monthly with a forward-looking forecast layer so the dashboard supports both historical reporting and short-term planning.

**Formula used:**
```
Order Month = DATE ( YEAR ( Fact_table[order_date] ), MONTH ( Fact_table[order_date] ), 1 )
```
Line chart: X axis = `Order Month`, Y axis = `Total Sales`
Analytics pane: Trend line — On | Forecast — On (Units: Month, Length: 6) | Confidence shade — On

**Implementation notes:**
- `order_date` is duplicated and cleaned to a date-only type in Power Query before being used to build `Order Month`, ensuring the time-based line chart doesn't fragment by time-of-day values inherited from the source `date_orders` field.
- The 6-month forecast with confidence shading is used to flag directional risk (e.g., the widening confidence band toward 2018) rather than to be read as a precise prediction.

**Current results from the dashboard:**
- **Total Sales (2015–2018):** 1,989,984.85
- **Top region by sales:** Western Europe (307,453.95), followed by Central America (259,576.18) and Southeast Asia (154,828.20).
- Sales show a strong upward move into early 2018 before the trend line flattens and the forecast band widens — the increased uncertainty in the forecast coincides with the tail end of the available historical data, so the 6-month projection should be treated as directional rather than firm.

---

## 6. Key Insights & Business Recommendations

### Executive Dashboard (Fulfillment, Delivery, Inventory)

**Insights:**
- **Fulfillment (95.97%) and Late Delivery (51.22%) tell two different stories** — orders are being processed and shipped, but not arriving on time. A high fulfillment rate alone is masking a significant service-level problem.
- **Supplier reliability (53.90) is the weakest of the four gauge KPIs**, and its weakness lines up with the late-delivery problem — this points to the delay originating upstream (supplier lead time/consistency) rather than purely in last-mile logistics.
- **Dead stock (45,605 units) combined with a 0.47x turnover ratio** shows inventory is accumulating faster than it's selling — this ties up working capital and warehouse space simultaneously.

**Recommendations:**
- **Treat late delivery as the priority metric**, not fulfillment — investigate the gap between "order fulfilled" and "order arrived on time," starting with suppliers scoring below the 53.90 reliability average.
- **Cross-reference late deliveries against supplier reliability tier** (see Supplier dashboard) to confirm whether specific low-reliability suppliers are driving the bulk of late orders, rather than treating delay as a network-wide issue.
- **Launch a dead-stock clearance or markdown review** for the 45,605 flagged units, and reassess reorder points for products with the lowest turnover to prevent the ratio from declining further.

### Warehouse Efficiency Dashboard

**Insights:**
- **The network is under-utilized on average (55.15%)**, with only one warehouse (L4, 84.24%) above the 80% target — capacity risk is currently concentrated rather than widespread.
- **L9 stands out as significantly under-utilized (15.93%)** while still holding meaningful stock, based on its position in the Stock Level vs. Capacity Utilization scatter plot — this is a candidate for consolidation or reallocation.
- **Capacity share is concentrated in five warehouses** (L2, L5, L9, L6, L0 account for the large majority of the pie chart), meaning operational risk from a single-site disruption is not evenly spread across all 10 warehouses.

**Recommendations:**
- **Monitor L4 for Capacity Risk escalation** — it's the only warehouse near the 80% target line; a small increase in volume could push it into "Watch" or "Critical" per the Capacity Risk Flag logic.
- **Evaluate rebalancing volume from high-utilization to low-utilization warehouses** (e.g., toward L9, L8, L1) to smooth utilization across the network and reduce single-site risk.
- **Re-examine warehouse sizing/capacity assumptions** for consistently low-utilization sites — persistent under-utilization may indicate over-provisioned capacity that could be resized or repurposed.
