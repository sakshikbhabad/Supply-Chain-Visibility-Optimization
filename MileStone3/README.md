# Milestone03_PowerBI — Report Documentation

This report contains two dashboards built on `Dim_supplier` and `Fact_table`:

1. **Supplier Scorecard Generation**
2. **Transportation Analysis**

---

## 1. Supplier Scorecard Generation

### Purpose
Evaluate and rank suppliers based on quality, reliability, and lead time performance, helping procurement/operations identify top and at-risk suppliers.

### KPI Cards
| KPI | Measure | Description |
|---|---|---|
| Total Suppliers | `Total Suppliers` | Distinct count of suppliers in the model |
| Avg. Quality Score | `Avg Quality Score` | Average quality score across suppliers |
| Avg. Reliability % | `Avg Reliability %` | Average reliability percentage across suppliers |
| Avg. Lead Time (Days) | `Avg Lead Time (Days)` | Supplier-level average lead time, averaged across all suppliers |

### DAX Measures
```dax
Total Suppliers = DISTINCTCOUNT(Dim_supplier[supplier_name])

Avg Quality Score = AVERAGE(Dim_supplier[quality_score])

Avg Reliability % = AVERAGE(Dim_supplier[reliability_%])

Avg Lead Time (Days) =
AVERAGEX(
    VALUES(Dim_supplier[supplier_name]),
    CALCULATE(AVERAGE(Dim_supplier[lead_time_(days)]))
)

Products Supplied = DISTINCTCOUNT(Dim_supplier[product_card_id])

Orders Fulfilled (Proxy) =
CALCULATE(
    DISTINCTCOUNT(Fact_table[order_id]),
    Fact_table[order_status] IN { "COMPLETE", "CLOSED" }
)

Low Tier Count =
CALCULATE(DISTINCTCOUNT(Dim_supplier[supplier_id]), Dim_supplier[Reliability Tier] = "Low")

Medium Tier Count =
CALCULATE(DISTINCTCOUNT(Dim_supplier[supplier_id]), Dim_supplier[Reliability Tier] = "Medium")

High Tier Count =
CALCULATE(DISTINCTCOUNT(Dim_supplier[supplier_id]), Dim_supplier[Reliability Tier] = "High")
```

### Calculated Columns (on `Dim_supplier`)
```dax
Reliability Tier =
SWITCH(
    TRUE(),
    Dim_supplier[reliability_%] >= 80, "High",
    Dim_supplier[reliability_%] >= 50, "Medium",
    "Low"
)

Supplier Composite Score =
    ( Dim_supplier[quality_score] * 0.4 )
  + ( Dim_supplier[reliability_%] * 0.4 )
  + ( ( 30 - Dim_supplier[lead_time_(days)] ) / 30 * 100 * 0.2 )

Supplier Rank =
RANKX(
    ALL(Dim_supplier),
    Dim_supplier[Supplier Composite Score],
    ,
    DESC
)
```

**Composite score logic:** weighted blend of Quality (40%), Reliability (40%), and Lead Time efficiency (20%, inverted — shorter lead time scores higher, normalized against a 30-day baseline).

### Visuals
| Visual | Type | Fields |
|---|---|---|
| Avg. Lead Time / Total Suppliers / Avg. Quality Score / Avg. Reliability % | Card | KPI measures above |
| Product-level Reliability table | Table + data bars | `Product Name`, `Avg. Reliability %` |
| Supplier Tier Distribution | Donut chart | `Reliability Tier` (Low / Medium / High) |
| Lead Time vs Reliability by Quality Score | Scatter chart | X: `lead_time_(days)`, Y: `reliability_%`, Size/Color: `quality_score` |
| Lead Time by Supplier Tier | Clustered bar | `Reliability Tier` vs `Avg Lead Time (Days)` |
| Top Suppliers by Composite Score | Bar chart | `supplier_name` vs `Supplier Composite Score`, sorted by `Supplier Rank` |

### Color Standard (Reliability Tier — used across donut & scatter)
| Tier | Hex |
|---|---|
| High | `#3BA55D` (green) |
| Medium | `#F4A83D` (amber) |
| Low | `#E15759` (red) |

---

## 2. Transportation Analysis

### Purpose
Track shipping performance, discounting, and delivery reliability across shipping modes, markets, and regions to surface operational risk areas.

### KPI Cards
| KPI | Measure | Description |
|---|---|---|
| Avg. Profit per Order | `Avg Profit Per Order` | Average order-level profit |
| Avg. Discount Rate | `Avg Discount Rate` | Average order-item discount rate |
| Total Discount Given | `Total Discount Given` | Sum of order-item discounts |
| Same-Day Share % | `Same Day Share %` | Proportion of total orders shipped Same Day |

### DAX Measures
```dax
Avg Profit Per Order = AVERAGE(Fact_table[order_profit_per_order])

Total Discount Given = SUM(Fact_table[order_item_discount])

Avg Discount Rate = AVERAGE(Fact_table[order_item_discount_rate])

Same Day Share % =
DIVIDE(
    CALCULATE(
        DISTINCTCOUNT(Fact_table[order_id]),
        Fact_table[shipping_mode] = "Same Day"
    ),
    [Total Orders],
    0
)

Late Rate by Shipping Mode =
CALCULATE(
    [Late Delivery %],
    ALLEXCEPT(Fact_table, Fact_table[shipping_mode])
)
```





