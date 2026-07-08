# Milestone 1 — Data Modelling
### Supply Chain Visibility & Optimization (Power BI)

## Objective
Build the data foundation for a supply chain analytics solution by importing,
cleaning, and modelling the DataCo Smart Supply Chain dataset in Power BI, then
defining a star schema and a set of reusable DAX measures that later milestones
(dashboards, KPIs, forecasting visuals) will build on.

## Dataset Source
**DataCo Smart Supply Chain for Big Data Analysis**
Kaggle: https://www.kaggle.com/datasets/shashwatwork/dataco-smart-supply-chain-for-big-data-analysis

- 180,519 rows × 53 columns
- One row per order line item, covering customer, product, shipping, and
  order/location details from Jan 2015 – Jan 2018
- A local copy used for this milestone is included at `data/SupplyChain.csv`

## Data Cleaning and Transformation Steps
All transformations were performed in Power Query Editor (PQE). See
`PowerQuery_M_Scripts.md` in this folder for the exact M code used for each query.

1. **Import**: Loaded the CSV via Get Data → Text/CSV, promoted headers, and
   renamed the base query to `Fact_table`.
2. **Removed columns** with no analytical value or privacy concerns:
   `Customer Email`, `Customer Password`, `Order Zipcode`, `Product Description`
   (100% null), `Product Image` (URL only).
3. **Corrected data types**: `order date (DateOrders)` and
   `shipping date (DateOrders)` → Date/Time; numeric ID and metric columns →
   Whole Number / Decimal Number as appropriate.
4. **Removed duplicate rows** at the source level.
5. **Built dimension tables** from `Fact_table` (via Reference, not Duplicate,
   so every dimension stays in sync with the same cleaned source):
   - **Dim_Customer** — `Customer Id`, name, city, state, street, zipcode,
     country, segment. Deduplicated on `Customer Id`.
   - **Dim_Product** — `Product Card Id`, `Product Category Id`, name, price,
     status, `Category Id`, `Department Id`. Deduplicated on `Product Card Id`.
   - **Dim_Category** — `Category Id`, `Category Name`. Deduplicated on
     `Category Id`.
   - **Dim_Department** — `Department Id`, `Department Name`. Deduplicated on
     `Department Id`.
   - **Dim_Shipping** — shipping-related attributes (days for shipping/shipment,
     delivery status, late delivery risk, shipping date, shipping mode) that
     don't have a natural single-column key, so a surrogate `Shipping_Id` was
     generated via an index column, then merged back into `Fact_table`.
   - **Dim_Location** — market, order city/country/region/state, again keyed by
     a generated surrogate `Location_Id` merged back into `Fact_table`.
   - **Dim_Date** — a calculated date table spanning the min order date to the
     max shipping date, with Year, Month, Month Number, Quarter, Week, Day, and
     Day Name columns (DAX in `DAX_Measures.md`).
6. **Removed redundant descriptive columns from `Fact_table`** once each
   dimension was built and merged, keeping only foreign keys and measures
   (Customer/Category/Department/Product/Shipping/Location IDs plus Sales,
   Profit, Quantity, Discount, and the two date columns).
7. Applied all changes (Close & Apply) and saved.

## Data Model Overview
Star schema with `Fact_table` at the center and six dimension tables:
