# Supply Chain Performance & Operational Risk Analysis - Project Documentation

## ETL & Engineering Notes

### 1. Sales & Orders

#### dim_customers

- Removed personal and contact fields, along with location and registration attributes that were not needed for the customer segmentation and purchasing behaviour analysis.

#### fact_orders

- **Data Preparation:** Removed the unused `shipping_address` field and standardized date and monetary data types. Added `expected_delivery_days` using the order date and expected delivery date.
- **Dimension Matching:** matched `order_status`, `payment_method`, and `payment_status` to `dim_orders_junk` using a composite-key merge.
- **Data Quality:** Kept the original order-level financial fields. Excluded `final_amount` from revenue calculations because its Inconsistencies could not be explained using the available data. Revenue calculations are based on order-item level.

#### fact_order_items

- **Data Preparation:** Standardized data types and prepared monetary fields for currency conversion.
- **Data Quality:** Reconstructed `line_total` using `(quantity * unit_price) - discount_amount + tax_amount` after identifying inconsistencies in the source values.
- **Net Sales:**
- **Feature Engineering:**
  - Added `discount_segment` to group items by discount tier.
  - Added `net_line_total` by subtracting `tax_amount` from the reconstructed `line_total`. This provides a tax-exclusive line for Net Sales.
- **Statistical Outlier Detection:** Applied the IQR method to `line_total`, identified 6,331 outliers (14.08%). Reviewed high-value samples and found that they were aligned with expensive products and high purchase quantity. No clear data errors were identified, so the records were retained.

#### dim_products

- **Data Preparation:** Merged category information from `dim_categories` into `dim_products`, removed the redundant `category_id`, and applied Trim to product and category names.
- **Column Pruning:** Removed attributes that were outside the project scope, redundant, or lacked reliable analytical definitions. Kept `stock_unit` and `product_status` for inventory analysis.
- **Data Quality Limitation:** Found repeated product names and inconsistencies in product units and prices. Kept the records without making unsupported manual corrections. The compatibility between product costs and quantity units remains unverified.

#### dim_orders_junk

- Extracted distinct combinations of `order_status`, `payment_method`, and `payment_status` from the orders data. Assigned `orders_junk_id` to each combination to use it in `fact_orders`.

---

### 2. Inventory & Warehouses

#### dim_suppliers

- **Column Pruning:** Removed `contact_person`, `email`, and `contact_number` because they were not needed for suppliers analysis.
- **Data Preparation & Validation:** Set `supplier_rating` as a decimal and `lead_time_days` as a whole number. Checked `supplier_id` for nulls and duplicates.
- **Statistical Outlier Detection:** Applied the IQR method to `lead_time_days`. No outliers were detected, all values fell within the calculated IQR limits.

#### dim_warehouses

- **Column Pruning:** Removed `warehouse_name`, `city` was used as the warehouse name in the analysis. Also removed `country` and `status` because each contained only one distinct value and provided no additional analytical or filtering value. Also removed manager and contact fields.
- **Data Quality Limitation:** Removed `storage_capacity` because its measurement unit was not documented, so it could not be used reliably for warehouse capacity analysis.

#### fact_inventory

- **Column Pruning:** Removed the original `inventory_status` field and the unused `last_restock_update` field.
- **Inventory Classification:** Created a new `Inventory_Status` field using `available_stock` and the reorder parameters for each product-warehouse position:
  - `Out of Stock`: `available_stock <= 0`
  - `Low Stock`: `available_stock > 0` and `available_stock <= reorder_level`
  - `Overstock`: `available_stock > ((reorder_level + reorder_quantity) * 1.5)`
  - `In Stock`: Any other cases.
- **Classification Limitation:** The 1.5 multiplier used for Overstock is a project-defined assumption, not a documented rule.
- **Data Quality Limitations:**
  - Products use different stock units, so stock quantities should not be aggregated across all products as if they shared one unit.
  - The source does not document how `available_stock` relates to `reserved_stock` and `damaged_stock`. These fields were retained separately and not used to adjust the inventory classification.

---

### 3. Logistics & Risk

#### fact_shipment

- **Data Preparation:** Standardized date and numerical data types.
- **Column Pruning:** Removed `customer_feedback` and kept `delivery_rating` instead.
- **Feature Engineering:** Calculated `expected_delivery_days` and `delay_days`. Negative delay values were set to zero for early deliveries. Pending shipments were kept as nulls.
- **Dimension Matching:** Replaced shipping attributes with `ship_junk_id` from `dim_ship_junk`, and tracking attributes with `ship_track_id` from `dim_shipment_tracking`.
- **Delay Reason Analysis:** Although 96% of shipments had no recorded `delay_reason`, the field was filled for 600 `Delayed` and 150 `Lost` shipments. All delay reasons had similar counts with no clear reason dominating. The field was retained.
- **Statistical Outlier Detection:** Applied the IQR method to `actual_delivery_days`. Identified 17 outliers, all exceeding the upper limit. These shipments had actual delivery durations of at least 14 days, with 7 delay days. The reviewed cases were consistent with the shipment date calculations.

#### fact_risk_events

- **Data Preparation:** Standardized date and numerical data types.
- **Feature Engineering:**
  - Calculated `days_to_resolve` from `event_date` and `resolved_date`.
  - Created`is_open_risk`. Set it to `0` for Resolved events and `1` for others.
- **Dimension Matching:** Replaced descriptive risk attributes with keys from `dim_risk_classify`, `dim_risk_detail`, and `dim_risk_resolution`.
- **Statistical Outlier Detection:** Applied the IQR method to `estimated_loss` and non-null `days_to_resolve` values. Identified 824 outliers (16.48%) in `estimated_loss`, but all aligned with High-Critical severity and were retained. No outliers were detected in `days_to_resolve`.

#### Supporting Dimensions

- `dim_shipment_tracking`: Contains distinct combinations of `shipment_priority`, `tracking_status`, and `is_on_time`.
- `dim_ship_junk`: Contains distinct combinations of `shipping_partner` and `shipment_status`.
- `dim_risk_classify`: Contains distinct combinations of `risk_category` and `severity`.
- `dim_risk_detail`: Contains distinct combinations of `risk_type`, `impact_area`, and `root_cause`.
- `dim_risk_resolution`: Contains distinct combinations of `status`, `is_open_risk`, and `resolution_action`.

Each dimension uses a generated key to connect its columns to the corresponding fact table.

#### Excluded Query

- `fact_shipping_risk`: Disabled Load and excluded from the model after identifying that it represents the same risk-event grain as `fact_risk_events`.
- `dim_categories`: Disabled Load and excluded from the model after merging its category information into `dim_products`.

---

### 4. Currency Normalization

- Converted monetary fields from INR to USD using a fixed average exchange rate of 88.2 INR per USD, calculated from monthly average exchange rates between July 2024 and July 2026.

**Source:** [X-Rates - Average Exchange Rates](https://www.x-rates.com/average/)

---

### 5. General Data Quality

- Checked key columns for nulls and duplicates and reviewed data types across the tables.
- Documented unresolved data-quality issues and IQR outlier findings rather than applying unsupported corrections.

## Data Modeling

| Fact Table         | Grain                                  |
| ------------------ | -------------------------------------- |
| `fact_orders`      | One row per customer order             |
| `fact_order_items` | One row per order line                 |
| `fact_inventory`   | One row per product-warehouse position |
| `fact_shipment`    | One row per shipment                   |
| `fact_risk_events` | One row per risk event                 |

![Power BI Data Model](images/data_model.png)

### 1. Sales & Orders

#### 1. Fact Table Grain

- `fact_orders`: Maintained at the order-header level, where each row represents one unique customer order.
- `fact_order_items`: Maintained at the order-line level, where each row represents one product line within an order. A single order can contain multiple items, so `order_id` is expected to repeat across rows, while `order_item_id` remains unique.

#### 2. Header-to-Line Item Relationship

- Established a One-to-Many (1:\*) relationship between `fact_orders` and `fact_order_items` using `order_id`.
- Applied single-direction filtering from `fact_orders` to `fact_order_items`, allowing order-level filters to apply to its order items.
- Maintained separate fact tables to preserve their different grains and prevent order-header values from being duplicated during item-level analysis.

#### 3. Dimension Relationships

- `dim_customers`: Connected to `fact_orders` through `customer_id` to support customer-level order analysis.
- `dim_orders_junk`: Connected to `fact_orders` through `orders_junk_id` to support filtering by order status, payment method, and payment status.
- `dim_date`: Connected to `fact_orders` through `order_date` to support time-based analysis.
- `dim_products`: Connected directly to `fact_order_items` through `product_id` to support product-level analysis using product attributes.

#### 4. Filter Flow

- Customer, order-status, payment, and date filters flow from their respective dimensions to `fact_orders` then pass to `fact_order_items` through the header-to-line relationship.
- Product filters directly affect `fact_order_items` but do not affect `fact_orders`.
- This filtering structure preserves the intended analytical grain of each table and avoids unnecessary bidirectional relationships.

#### 5. Modeling Notes

- Kept order-level and item-level calculations at their appropriate grains.
- Used distinct order counting when analyzing orders through `fact_order_items`, since the same `order_id` can appear across multiple item rows.
- Avoided creating an additional order dimension to connect the header and line-item tables, as the existing One-to-Many relationship provides a clear filtering path.

---

### 2. Inventory & Suppliers

- **fact_inventory Grain:** Kept `fact_inventory` at the product-warehouse level, where each record represents a product's inventory position in a specific warehouse. This allows inventory levels and reorder needs to be analyzed separately across warehouses.

- **Inventory Relationships:** Connected `fact_inventory` to `dim_products` and `dim_warehouses` through active one-to-many relationships with single-direction filtering. This allows product and warehouse selections to filter inventory records.

- **Supplier Modeling:** Kept `dim_suppliers` as a separate supplier table. Its 500 unique `supplier_id` values represent distinct supplier records, while the dataset contains only 50 distinct supplier names.

- **Supplier-to-Product Relationship:** Connected `dim_suppliers` to `dim_products` through `supplier_id` using an active one-to-many, single-direction relationship. This created a snowflake dimension path that allows supplier filters to flow through `dim_products`.

- **Supplier-Inventory Analysis:** Since `dim_products` is already connected to `fact_inventory` through `product_id`, supplier selections can filter the corresponding inventory records through the `dim_suppliers` → `dim_products` → `fact_inventory` path.

- **Filter Behavior:** Product and warehouse selections filter `fact_inventory` directly, while supplier selections filter inventory through `dim_products`.

---

### 3. Logistics & Risk

- **Fact Table Grain:** Maintained `fact_shipment` at the shipment level, with `shipment_id` validated as unique. Maintained `fact_risk_events` at the individual risk-event level, with `risk_event_id` as its unique identifier. Repeated `shipment_id` values in `fact_risk_events` are allowed because multiple risk events may reference the same shipment identifier.

- **Separate Fact Tables:** Kept `fact_shipment` and `fact_risk_events` as separate fact tables. Each retains its own grain and connects to the relevant dimensions, including shared shipment dimensions.

- **Shipment-to-Risk Attribute Matching:** Enriched `fact_risk_events` with `ship_junk_id` and `ship_track_id` from `fact_shipment` using `shipment_id` through a Power Query merge. This allows shipment-related attributes to be used when analyzing risk events associated with shipment impact areas.

- **Shipment Dimensions:** Connected `dim_ship_junk` and `dim_shipment_tracking` to both `fact_shipment` and `fact_risk_events` through active one-to-many, single-direction relationships. These dimensions support shipment analysis by shipping partner, shipment status, priority, and tracking status. The `delay_reason` field remains in `fact_shipment` for delay analysis.

- **Risk Dimensions:** Connected `dim_risk_classify`, `dim_risk_detail`, and `dim_risk_resolution` exclusively to `fact_risk_events` through active one-to-many, single-direction relationships. These dimensions support risk classification, severity assessment, and resolution monitoring.

- **Product and Warehouse Dimensions:** Connected `dim_products` and `dim_warehouses` to `fact_risk_events` through `product_id` and `warehouse_id`, respectively, to support product-level and warehouse-level risk analysis.

- **Shared Dimensions:** Retained `dim_date` and `dim_warehouses` across relevant fact tables, while linking shipment dimensions by foreign keys to support both shipment and risk-impact analysis.

- **Risk Fact Deduplication:** Kept `fact_risk_events` as the single source of truth for operational risk incidents. Disabled the load of `fact_shipping_risk` because it was identified as a redundant subset, preventing duplicated risk records and financial-loss calculations.

- **Analytical Limitation:** Carrier-level risk comparisons are restricted to records with `impact_area = "Shipment"`.

---

### 4. Relational Integrity Validation

- **Primary Key Uniqueness:** Verified that all dimension primary keys are unique and that the fact table grain keys contain no duplicates:
  - `fact_orders`: `order_id`
  - `fact_order_items`: `order_item_id`
  - `fact_inventory`: `product_id` + `warehouse_id`
  - `fact_shipment`: `shipment_id`
  - `fact_risk_events`: `risk_event_id`

- **Key Completeness:** Confirmed that the validated primary and grain keys contain no null or blank values.

- **Foreign Key Matching:** Used Power Query Merge operations to check foreign key matching across all relationships. No unmatched records were identified.

- **Date Relationships:** Reviewed date relationships through report filtering and confirmed that date selections behave as expected. `dim_date` table is a DAX calculated table.

- **Relationship Validation:** Verified that all relationships are active One-to-Many relationships with single-direction filtering. Dimension-to-Fact relationships follow the intended filter direction, while the `fact_orders`-to-`fact_order_items` relationship supports order-header-to-line-item filtering.

- **Data Type:** Confirmed compatible data types between the key columns used in the model.

---

### 5. Overall Modeling Architecture

- **Schema Architecture:** Designed a multi-fact (Galaxy) schema to separate business domains (Sales, Inventory, Logistics, and Risk) while supporting cross-functional analysis through shared dimensions.

- **Fact Table Separation:** Maintained fact tables at their appropriate grains, avoiding fact-to-fact relationships. The only direct relationship between facts is the 1-to-Many relationship between order tables.

- **Header-Line Item Logic:** Connected order headers to order items using a single-direction 1-to-Many relationship. This preserves order-level and item-level granularity and prevent any duplication.

- **Conformed and Snowflaked Dimensions:** Shared dimensions support cross-functional analysis across relevant fact tables. `dim_products` connects to `fact_order_items`, `fact_inventory`, and `fact_risk_events`, while `dim_suppliers` filters related products and inventory through `dim_products`.

- **Logistics and Risk Separation:** Kept `fact_shipment` and `fact_risk_events` independent, with no direct relationship. Shared shipment dimensions used to connect both facts.

- **Controlled Filter Flow:** Used single-direction filtering to maintain predictable filter paths and avoid unnecessary bidirectional relationships or ambiguous filtering paths.

- **Centralized Time Intelligence:** Used a centralized `dim_date` table to support time-based analysis across relevant fact tables, with date filtering following the intended date role of each table.

---

## DAX & Measure Design

DAX was used to create the centralized `dim_date` calculated table and develop analytical measures across the project's main business areas: Sales & Customer Behavior, Inventory & Suppliers, and Logistics & Risk.

The measures cover core business KPIs, financial calculations, ratios, time-based comparisons, and operational indicators. Calculations were designed to respect the grain of each fact table and the intended filter context, with helper measures used where needed for intermediate calculations.

Measures were validated against source records or independent calculations, and supporting measures were hidden from the interface.

A detailed reference of the DAX measures, including their business definitions, formulas, formats, filter behavior, and validation notes, is maintained separately in [DAX Measures](./DAX_Measures.md).

---

## Insights & Recommended Actions

### 1. A Quarter of Risk Events Accounted for 82% of Estimated Losses

**Key Findings**

- In 2025, **5K risk events** carried about **$1.9M in estimated losses**, equivalent to **20.6% of annual net sales ($9.3M)** and around **1.6 times the annual gross profit ($1.2M)**.
- High and Critical events represented only **25%** of all events but accounted for **82%** of the estimated losses.
- Among High and Critical cases not marked as Resolved, **52% (273) were Mitigated**, while **249 (48%) remained Open or Investigating**, carrying **$269.4K** in estimated losses.

Risk Types with Supporting Analysis:

| Risk Type      |  Cases | Estimated Loss |
| :------------- | -----: | -------------: |
| Overstock      |     24 |     **$38.3K** |
| Stockout       |     20 |     **$29.6K** |
| Late Delivery  |      9 |     **$17.1K** |
| Shipment Delay |     15 |     **$15.9K** |
| **Total**      | **68** |    **$100.9K** |

**Recommended Actions**

Prioritize Open and Investigating High-Critical cases, using the following guide:

- **Overstock:** Review the top product-warehouse positions flagged by the revised rule before placing new orders or reallocating stock.
- **Stockout:** Prioritize replenishment by reviewing warehouse-level stockout rates and most demanded products.
- **Delays:** Use the strict delivery rate to identify late shipments missed by the source's two-day delay rule to improve shipment visibility and enable earlier follow-up

  _These actions help prioritize and address current risks more effectively, while supporting better operational decisions to reduce the occurrence of similar events in the future._

  **Notes:**

- Estimated losses represent potential financial loss, not confirmed actual losses.
- The recorded `root_cause` and `resolution_action` fields showed inconsistent associations with risk types so both were excluded from the analysis.
- The dataset provides current risk statuses only, with no status-change history so time spent in each status cannot be measured.

---

### Insight 02 - High and Top Discount Tiers Accounted for 45% of Net Sales but Only 26% of Gross Profit

#### Key Findings

High and Top discount tiers generated **$8.45M (45%)** of the $18.8M Paid Net Sales, and only **$629K (26%)** of the $2.4M Gross Profit.

| Discount (Limit) | Units Sold | Gross Profit | Margin |
| ---------------- | ---------: | -----------: | -----: |
| (0%) No Discount |      6,366 |        $194K |  21.3% |
| (5%) Low         |     32,598 |        $949K |  19.2% |
| (10%) Medium     |     31,981 |        $667K |  14.8% |
| (15%) High       |     31,767 |        $443K |  10.1% |
| (20%) Top        |     32,049 |        $186K |   4.6% |

Units sold nearly flat across discount tiers, while gross profit dropped significantly at higher discounts. This raised the question of whether those discounts were really necessary.

#### New Discount Scenarios

| Discount Scenario | Profit Increase | Volume Drop to Break-even |
| ----------------- | --------------: | ------------------------: |
| High & Top at 5%  |         +$1.04M |                   **62%** |
| High & Top at 10% |          +$545K |                   **46%** |

#### Recommended Actions

- **Discount Optimization:** Test these discount limits on selected High-Top products. The scenarios reveal a potential Gross Profit increase as long as the combined units sold drop stays below the Break-even levels.
- **Inventory Alignment:** Check whether High-Top discount products align with products flagged by the new Overstock rule. This could help determine whether overstock was the reason for these discounts. If confirmed, the new Overstock rule could help identify excess stock earlier to reduce the need for such discounts in the future.

#### Notes

- Scenario calculations assume unchanged product and unit costs, with the same proportional unit-sales decline across both High and Top tiers.
- Volume Drop to Break-even shows the percentage drop in combined High and Top unit sales that would bring Gross Profit back to its current level.

---

### Insight 03 - Bronze Customers Represent the Largest Repeat-Purchase Opportunity

#### Key Finding:

Among **3,923 eligible customers**, 1,807 (46%) placed a second paid order within 180 days of their first recorded paid order.

Repeat-purchase rates increased across all loyalty tiers over longer follow-up periods, but Bronze customers consistently had the lowest rates.

| Loyalty Level | Eligible Customers | Within 30 Days | Within 90 Days | Within 180 Days |
| ------------- | -----------------: | -------------: | -------------: | --------------: |
| Bronze        |              1,442 |             5% |            14% |             27% |
| Silver        |              1,282 |            10% |            27% |             47% |
| Gold          |                843 |            14% |            35% |             61% |
| Platinum      |                356 |            25% |            58% |             84% |
| **Total**     |          **3,923** |        **10%** |        **27%** |         **46%** |

The rates are cumulative and use the same eligible customer group. Bronze customers make **37% of this group**, yet only 27% made a second purchase within 180 days.

#### Business Opportunity:

- A **5%** in Bronze's 180-day repeat-purchase rate (27% to 32%) translates to about **72 additional returning customers**.
- Bronze customers have an average paid order value of **$1.5K**, similar to Platinum customers.
- At the historical Bronze average paid order value of $1.5K, one additional paid order from each customer would represent about **$108K in additional net sales**.

#### Recommended Action:

Target Bronze customers who have not placed a second paid order with personalized product recommendations and offers.

Test targeted Rewards rather than broad discounts, given the lower gross margins observed at higher discount levels in Insight 02. Compare the campaign against a control group, measuring 180-day repeat-purchase rates and gross profit per targeted customer.

#### Notes

- **Eligible customers** are customers whose first recorded paid order was on or before 21-Jan-2026 to ensure a full 180-day period through 20-Jul-2026.
- Analyzed paid orders only, with one record per order to identify each customer's first and second paid-order. A repeat purchase is defined as a second distinct paid order, including orders placed on the same day.
- The first recorded paid order may not represent a customer's first-ever purchase.
