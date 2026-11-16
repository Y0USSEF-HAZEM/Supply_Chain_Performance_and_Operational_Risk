# Supply Chain Performance & Operational Risk Analysis - Project Log

**Author:** Youssef Hazem Youssef  
**start date:** 12 October 2026  
**Tools:** Power BI, Power Query, DAX  
**Purpose:** Record key investigations, analytical and modeling decisions, validation checks, and unresolved limitations faced during the project.

---

## Entry 01 - Inconsistent Source Calculations Made Paid Sales Appear to Generate $8.9M Gross Loss

**Date:** 17 October 2026  
**Area:** Sales & Orders - Financial Data Validation  
**Status:** Order-item calculations applied, header-level matching remains unverified.

### Problem

The initial calculation using `fact_orders[final_amount]` produced a negative gross profit of **$8.9M** under the Paid filter. Excluding tax made the result worse, increasing the negative gross profit to **$10M**.

The original `fact_order_items[line_total]` also produced a negative gross profit and did not match the expected row-level calculation.

### Decision

I rebuilt the `order-line` amounts using:

`(quantity * unit_price) - discount_amount + tax_amount`

I then excluded tax when calculating net sales and used the corrected item-level data as the basis for the core sales measures, rather than relying on the unexplained header-level `final_amount`.

### Results

Under the Paid filter, the new calculations produced:

| Metric         |    Amount |
| -------------- | --------: |
| Paid Net Sales |    $18.8M |
| Total Cost     |    $16.4M |
| Gross Profit   | **$2.4M** |

### Notes

- Included only orders with `Paid` payment status.
- Excluded tax from product sales.
- Excluded customer shipping amounts from the sales after finding a 1:1 match with shipping costs.
- Calculated product cost using item quantities and product costs.
- The unit of recorded product costs has not been independently verified.

---

## Entry 02 - A Lenient Overstock Rule May Have Left Excess Stock Unnoticed

**Date:** 21 October 2026  
**Area:** Inventory & Warehouses - EDA & Business Logic  
**Status:** Analytical classification applied, original source logic remains unverified.

### Problem

While reviewing inventory levels, I noticed that many positions had Available Stock much higher than their combined reorder parameters, yet only 4,207 of 25,000 (16.8%) product-warehouse positions were classified as `Overstock`. This raised concerns that the source classification might be too lenient in identifying excess stock.

### Investigation

I tested different thresholds against the combined product `reorder_level` and `reorder_quantity` to assess whether they could regenerate the source's classification logic.

Using the original `available_stock`, a threshold of about 290% of the combined reorder parameters produced an Overstock share close to the source. However, matching the overall share does not confirm that all same positions were classified as Overstock.

| Multiplier | Overstock % |
| ---------: | ----------: |
|        1.0 |         68% |
|        1.5 |         52% |
|        2.0 |         36% |
|        3.0 |         15% |
|        4.0 |          6% |

### Decision

I applied a **150% threshold** against the product reorder parameters:

`available_stock > 1.5 × (reorder_level + reorder_quantity)`

The 1.5 multiplier sets the Overstock threshold 50% above the combined reorder parameters.

| Inventory Status | Classification Logic                                        |
| ---------------- | ----------------------------------------------------------- |
| Out of Stock     | Available Stock ≤ 0                                         |
| Low Stock        | 0 < Available Stock ≤ Reorder Level                         |
| Overstock        | Available Stock > 150% × (Reorder Level + Reorder Quantity) |
| In Stock         | other positions                                             |

### Results & Business Context

The new classification identified **13,030 of 25,000 positions (52%)** as Overstock, over three times the source count. Bringing additional excess-stock positions to light.

Overstock is the largest Inventory risk type in the source with about **$136K in estimated losses**. The revised rule gives planners additional positions to review before making reorders or stock-allocation decisions, potentially reducing future losses.

### Limitations

- The original classification logic remains unverified.
- The 150% threshold is an analytical assumption, not a documented rule.
- It counts product-warehouse positions, not unique SKUs.
- The classification identifies potential excess-stock positions, but does not confirm that every position is genuinely overstocked.

---

## Entry 03 - Why Warehouse Utilization Could Not Be Measured Reliably

**Date:** 22 October 2026  
**Area:** Inventory & Warehouses - Data Validation  
**Status:** Closed - Warehouse Utilization discontinued; a separate reorder-risk measure introduced.

### Problem

The initial `Warehouse Utilization %` measure returned incredibly high values, reaching **13,699%** for some warehouses. Reviewing the data revealed two critical issues:

1. `fact_inventory[available_stock]` contains quantities in mixed units **(Piece, Box, Kg, Liter, Pack)** that cannot be meaningfully aggregated into a single physical quantity.
2. The measurement unit for `dim_warehouses[storage_capacity]` was undocumented.

### Investigation & Analytical Decision

Based on the capacity values and warehouse regions, I considered whether `storage_capacity` might be recorded in **square feet (ft²)**. However, the available documentation did not confirm this assumption.

I also explored using `dim_products[weight_kg]` to estimate total inventory weight. But weight could not be meaningfully compared with the undocumented warehouse capacity unit. Estimating the space taken up by inventory would require product dimensions, which were not available in the dataset.

So i removed the `Warehouse Utilization %` measure rather than present unsupported conclusions about physical warehouse space.

### Decision

I introduced `Reorder Risk %` to measure the share of product-warehouse positions classified as `Low Stock` or `Out of Stock` within the current filter context. It highlights inventory positions that need restocking, rather than measuring warehouse space or capacity usage.

---

## Entry 04 - Product Master Data Quality: Repeated Descriptions and Inconsistent Attributes

**Date:** 24 October 2026  
**Area:** Product Data - EDA & Data Quality  
**Status:** Data quality limitations documented, no unsupported corrections applied.

### Problem

During product data profiling, I found multiple records sharing the same `product_name`, `brand`, and `category_name` but having different `product_id` values.

Grouping 5,000 product records by these three attributes produced only 248 distinct combinations. This revealed the repetition of product descriptions across records, although it did not confirm if they were duplicate products or different versions.

### Investigation

Example:

- **Penguin Programming Book:** Records sharing the same product name, brand, and category had different product IDs and other attributes, including supplier, prices, discount percentages, and stock units. The observed units included `Liter`, `Box`, `Kg`, and `Piece`. This raised questions about the consistency of the stock-unit attribute.

### Decision

- I retained the records and their unique `product_id` keys rather than merging or deleting them, as the available data did not confirm if they were duplicate products or different versions.
- I kept `stock_unit` as a source attribute for data quality validation.
- I left stock units, prices, supplier assignments, and other attributes unchanged because the available data did not provide a reliable basis for correcting them.

### Notes

- **Inventory Aggregation:** The inconsistent stock-unit definitions prevent treating quantities across different products as a directly comparable physical measure.
- **Product Cost:** The `Total Cost` measure calculates costs by multiplying `quantity` from `fact_order_items` by the related `cost_price` from `dim_products`, using `product_id` to match records. This avoids relying on repeated product descriptions for product matching.

---

## Entry 05 - Enabling Supplier-Based Inventory Analysis

**Date:** 26 October 2026  
**Area:** Inventory & Suppliers - Data Modeling  
**Status:** Implemented and validated.

### Problem

The `Inventory & Suppliers` page initially contained supplier and inventory analyses that could not filter each other. Supplier slicers had no relationship path to `fact_inventory`, limiting the ability to analyze inventory in the context of individual suppliers.

### Modeling Decision

I created an active one-to-many, single-direction relationship `dim_suppliers`=>`dim_products` using `supplier_id`.

Since `dim_products` is already connected to `fact_inventory` through `product_id`, this created a filtering path from suppliers to inventory through the product dimension, without requiring a direct relationship between suppliers and inventory record.

### Validation

Tested supplier selections in the report and confirmed that they filtered the related products and inventory records as expected.

---

## Entry 06 - All Risk Types Shared the Same Recorded Causes and Resolution Actions

**Date:** 28 October 2026  
**Area:** Logistics & Risk - Data Quality  
**Status:** Data quality limitations documented, affected fields excluded from analytical insights.

### Problem

While reviewing the risk data, I found that every `risk_type` appeared with all 8 distinct `root_cause` values and all 8 `resolution_action` values.

### Investigation

The analysis covered **5,000 risk events across 25 risk types**. The results showed the following pattern:

| Column              | Distinct Values | Risk Types |
| ------------------- | --------------: | ---------: |
| `root_cause`        |               8 |         25 |
| `resolution_action` |               8 |         25 |

Two examples from the data:

| Risk Type  | Root Cause                         | Resolution Action  |
| ---------- | ---------------------------------- | ------------------ |
| Fire Alarm | Heavy rainfall disrupted transport | Route optimized    |
| Heavy Rain | Incorrect inventory records        | Expedited shipment |

### Decision

- Excluded `root_cause` and `resolution_action` from analytical insights because Using them to explain risk events could lead to misleading decisions.
- Retained the original source values without manual corrections, since there was no reliable source to confirm the right connections.

### Analytical Limitation

The available data do not provide reliable conclusions for analyzing root causes or resolution actions by risk type. Further investigation would require more reliable source.

---

## Entry 07 — Conflicting Expected Dates and Delay Labels Changed the Delivery Rate

**Date:** 1 November 2026  
**Area:** Logistics & Risk - EDA & KPI Definition  
**Status:** Shipment expected date retained as the analytical standard.

### Problem

While reviewing delivery performance, I found two issues affecting the reported delivery rate:

- The order and shipment tables contained different expected delivery dates.
- The source's `Delayed` status did not mark shipments arriving exactly one day late.

Both issues could lead to different interpretations of delivery performance.

### Investigation

**1. Expected Delivery Date Comparison**

I merged `fact_orders` into `fact_shipment` and compared the expected delivery dates from both tables.

The comparison revealed differences in **13,349 of 15,000 shipments (89%)**. Shipment expected dates ranged from **8 days earlier to 7 days later** than order expected dates.

Among **9,078 completed shipments, excluding Returned and Pending**, **3,544 (39%)** received different On-Time/Late classifications depending on which expected date was used:

| Status                              | Count |
| ----------------------------------- | ----: |
| Order On-Time **but** Shipment Late | 1,546 |
| Order Late **but** Shipment On-Time | 1,998 |

Using the shipment expected date resulted in **452 more On-Time shipments**, increasing the strict on-time rate from **58% to 63%**. This classification aligned with the `is_on_time` column.

**2. Source Delay Classification**

I checked shipments with `shipment_status = "Delayed"` and found no records with `delay_days = 1`.

Further investigations showed that:

- **2,788 shipments** arrived exactly one day after the shipment expected date but were not classified as `Delayed` and had no recorded `delay_reason`.
- **600 shipments** classified as `Delayed` were at least two days late, with an average delay of **4.5 days**.

The recorded delay reasons among the 600 shipments classified as `Delayed` had similar counts with no clear reason dominating.

This indicated that the source's `Delayed` criteria starts at two days after the shipment expected date.

**3. One-Day Tolerance**

Allowing delivery up to one day after the shipment expected date increased the on-time rate from **63% to 93%**.

The source's `Delayed` status covered only **7% of eligible shipments**, while the strict analysis classified **37% as late**. The `is_on_time` column matches the strict analysis, showing that the gap lies in the `shipment_status` rather than a complete lack of delay tracking.

### Decision

I retained the shipment expected date as the analytical standard because actual delivery dates are recorded only in `fact_shipment`.

I introduced two separate delivery measures, both excluded Returned and Pending shipments:

- **63% Strict On-Time:** Any delay counted as late.
- **93% One-Day Tolerance:** Allowed being one day late (aligned with the shipment status criteria).
- _Orders expected date resulted **58%**._

These measures represent different delivery-performance definitions.

### Business Context

- **Financial Risk:** Logistics risk events accounted for **$608K** in estimated losses, including **$140K recorded for delivery delays**.
- **Monitoring Gap:** The source's `Delayed` status marked only **7% of eligible shipments**, while the strict analysis identified **37% as late**.
- **Business Value:** The new delivery measure provides clearer visibility into delayed shipments, helping to reduce future delay losses.

### Limitation

The dataset does not document which expected date represents the confirmed customer delivery agreement or whether a one-day tolerance is an approved policy. Therefore, the shipment expected date is used as the analytical standard, not claimed as the formally confirmed customer delivery promise.
