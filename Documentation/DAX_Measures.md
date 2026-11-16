# Supply Chain Performance & Operational Risk Analysis - Dax Measures

## Sales & Orders

### Total Sales

**Business Definition:** Calculates paid product sales excluding tax, based on the corrected order-line amounts.

**DAX Formula:**

```dax
Total Sales =
CALCULATE(
    SUM(fact_order_items[net_line_total]), dim_orders_junk[payment_status] = "paid"
)
```

**Format & Unit:** Currency ($), Decimal places: 1

**Affected By Page Filters:** Date, Customer Segment, Loyalty Level.

**Validation Test:** Filtered by 'Paid' status and compared the aggregate sums of 'line_total' and 'tax_amount' against 'Total Sales', confirming that the measure precisely subtracts Total Taxes from the Total Sales.

---

### Total Orders

**Business Definition:** Counts distinct orders within the current filter context.

**DAX Formula:**

```dax
Total Orders = DISTINCTCOUNT(fact_order_items[order_id])
```

**Format & Unit:** Whole Number

**Affected By Page Filters:** Date, Customer Segment, Loyalty Level.

**Validation Test:** Confirmed the unfiltered count matches fact_orders and tested category and discount filters.

---

### Paid Orders

**Business Definition:** Counts distinct orders with a payment status of Paid.

**DAX Formula:**

```dax
Paid Orders =
CALCULATE([Total Orders], dim_orders_junk[payment_status] = "paid" )
```

**Format & Unit:** Whole Number

**Affected By Page Filters:** Date, Customer Segment, Loyalty Level.

**Validation Test:** Filtered payment_status = "paid" and compared the distinct count of order_id in fact_order_items with the measure under the same filter context.

---

### AVG Paid Order Value

**Business Definition:** Calculates the average sales revenue per paid order, providing insight into customer spending habits.

**DAX Formula:**

```dax
AVG Paid Order Value = DIVIDE([Total Sales],[Paid Orders])
```

**Format & Unit:** Currency ($), Decimal places: 1

**Affected By Page Filters:** Date, Customer Segment, Loyalty Level.

**Validation Test:** Manually divided the [Total Sales] KPI card by the [Paid Orders] KPI card to verify the average is accurate.

---

### Gross Profit

**Business Definition:** Calculates gross profit on paid orders by subtracting product costs from sales revenue, excluding taxes.

**DAX Formula:**

```dax
Gross Profit = [Total Sales] - [Total Cost]
```

**Format & Unit:** Currency ($), Decimal places: 1

**Affected By Page Filters:** Date, Customer Segment, Loyalty Level.

**Validation Test:** Compared the measure with [Total Sales] - [Total Cost] under the same filter context.

---

### Profit Margin %

**Business Definition:** Represents the percentage of net revenue retained as profit.

**DAX Formula:**

```dax
Profit Margin % = DIVIDE([Gross Profit], [Total Sales])
```

**Format & Unit:** Percentage (%), Decimal places: 1

**Affected By Page Filters:** Date, Customer Segment, Loyalty Level.

**Validation Test:** Manually divided the [Gross Profit] value by the [Total Sales] value to verify mathematical accuracy.

---

### Repeat Customer Rate %

**Business Definition:** Calculates the percentage of purchasing customers who placed more than one paid order.

**DAX Formula:**

```dax
Repeat Customer Rate % =
VAR CustomerOrders =
    ADDCOLUMNS(
        VALUES(dim_customers[customer_id]),
        "@PaidOrders", CALCULATE([Paid Orders])
    )
RETURN
    DIVIDE(
        COUNTROWS(FILTER(CustomerOrders, [@PaidOrders] > 1)), COUNTROWS(FILTER(CustomerOrders, [@PaidOrders] > 0)), 0
    )
```

**Format & Unit:** Percentage (%), Decimal places: 0

**Affected By Page Filters:** Date, Customer Segment, Loyalty Level.

**Validation Test:** Verified the measure against repeat customers divided by purchasing customers, overall and by loyalty levels.

---

### Total Cost (Hidden)

**Business Definition:** Calculates the total Cost of Goods Sold (COGS) for all paid orders by multiplying the item quantity by its respective product cost.

**DAX Formula:**

```dax
Total Cost =
CALCULATE(
    SUMX(
        fact_order_items,
        fact_order_items[quantity] * RELATED(dim_products[cost_price])
    ),
    dim_orders_junk[payment_status] = "paid"
)
```

**Format & Unit:** Currency ($), Decimal places: 1

**Affected By Page Filters:** Date, Customer Segment, Loyalty Level.

**Validation Test:** Created a table visual filtered by 'Paid' status, displaying 'product_name', 'Sum of quantity','cost_price', and the 'Total Cost' measure. Verified row-by-row that multiplying the total quantity by units cost matches the measure's output.

---

### Total Sales MoM % (Hidden)

**Business Definition:** Calculates the month-over-month percentage change in total sales. Returns blank when the date context spans multiple months or no previous-month value exists.

**DAX Formula:**

```dax
Total Sales MoM % =
VAR Prev = [Last Month Sales]
RETURN IF([Month Check] && NOT ISBLANK(Prev), DIVIDE([Total Sales] - Prev, Prev)
)
```

**Format & Unit:** Dynamic, Decimal places: 1

**Affected By Page Filters:** Date (Time Intelligence), Customer Segment, Loyalty Level.

**Validation Test:** verified against a manual month-over-month calculation and confirm the measure returns blank for a multi-month date context.

---

### Gross Profit MoM % (Hidden)

**Business Definition:** Calculates the month-over-month percentage change in gross profit. Returns blank when selecting multiple months or no previous-month value exists.

**DAX Formula:**

```dax
Gross Profit MoM % =
VAR Prev = [Last Month Profit]
RETURN IF([Month Check] && NOT ISBLANK(Prev), DIVIDE([Gross Profit] - Prev, Prev))
```

**Format & Unit:** Dynamic, Decimal places: 1

**Affected By Page Filters:** Date (Time Intelligence), Customer Segment, Loyalty Level.

**Validation Test:** verified against a manual month-over-month calculation and confirm the measure returns blank for a multi-month date context.

---

### Last Month Sales (Hidden)

**Business Definition:** A helper measure calculating the net revenue of the previous calendar month

**DAX Formula:**

```dax
Last Month Sales = CALCULATE([Total Sales], PREVIOUSMONTH(Dim_Date[Date]))
```

**Format & Unit:** General , Auto

**Affected By Page Filters:** Date (Previous Month), Customer Segment, Loyalty Level.

**Validation Test:** Manually validated against the dynamic card visual.

---

### Last Month Profit (Hidden)

**Business Definition:** A helper measure calculating the gross profit of the previous calendar month.

**DAX Formula:**

```dax
Last Month Profit = CALCULATE([Gross Profit], PREVIOUSMONTH(Dim_Date[Date]))
```

**Format & Unit:** General , Auto

**Affected By Page Filters:** Date (Previous Month), Customer Segment, Loyalty Level.

**Validation Test:** Manually validated against the dynamic card visual.

---

### Month Check (Hidden)

**Business Definition:** Checks whether the current date context falls within a single calendar month.

**DAX Formula:**

```dax
Month Check =
VAR MinD = MIN(Dim_Date[Date])
VAR MaxD = MAX(Dim_Date[Date])
RETURN EOMONTH(MinD, -1) = EOMONTH(MaxD, -1)
```

**Format & Unit:** Boolean

**Affected By Page Filters:** Date

**Validation Test:** verified that it returns TRUE for a single-month context and FALSE for a date context spanning multiple months.

---

## Inventory & Suppliers

### Low-Out of Stock Positions

**Business Definition:** Counts inventory positions classified as either (Low Stock or Out of Stock). Each position represents a product within a specific warehouse. The measure counts (product & warehouse) records, not distinct SKUs.

**DAX Formula:**

```dax
Low-Out of Stock Positions =
CALCULATE( [Inventory Positions], Fact_Inventory[Inventory_Status] IN {"Low Stock", "Out of Stock"}
)
```

**Format & Unit:** Whole Number

**Affected By Page Filters:** Warehouse, Supplier, Supplier Type, Supplier Country.

**Validation Test:** Compare the measure against the row count of Fact_Inventory after filtering Inventory_Status to Low Stock and Out of Stock.

---

### AVG Supplier Lead Days

**Business Definition:** Calculates the average recorded lead time for the selected suppliers.

**DAX Formula:**

```dax
AVG Supplier Lead Days = AVERAGE(dim_suppliers[lead_time_days])
```

**Format & Unit:** Decimal Number, Decimal places: 0

**Affected By Page Filters:** Supplier, Supplier Type, Supplier Country.

**Validation Test:** Compared the measure with the average of the lead_time_days column in Power Query and verified that supplier slicers update the measure as expected.

---

### AVG Supplier Rating

**Business Definition:** Calculates the average recorded rating of suppliers.

**DAX Formula:**

```dax
AVG Supplier Rating = AVERAGE(dim_suppliers[supplier_rating])
```

**Format & Unit:** Decimal Number, Decimal places: 1

**Affected By Page Filters:** Supplier, Supplier Type, Supplier Country.

**Validation Test:** Compared the measure with the average of the supplier_rating column and verified that supplier slicers update the measure as expected.

---

### Replenishment Risk %

**Business Definition:** Calculates the percentage of inventory positions classified as Low Stock or Out of Stock.

**DAX Formula:**

```dax
Replenishment Risk % =
DIVIDE(
    [Low-Out of Stock Positions],
    CALCULATE(
        [Inventory Positions], REMOVEFILTERS(Fact_Inventory[Inventory_Status])
    )
)
```

**Format & Unit:** Percentage (%), Decimal places: 0

**Affected By Page Filters:** Warehouse, Supplier, Supplier Type, Supplier Country.

**Validation Test:** Compared the number of at-risk inventory positions with the total inventory positions under the same filter context.

---

### SKUs with Replenishment Risk

**Business Definition:** Counts distinct product IDs with at least one (Low Stock or Out of Stock) position in at least one warehouse.

**DAX Formula:**

```dax
SKUs with Replenishment Risk =
CALCULATE(
    DISTINCTCOUNT(Fact_Inventory[product_id]),
    Fact_Inventory[Inventory_Status] IN {"Low Stock", "Out of Stock"}
)
```

**Format & Unit:** Decimal Number, Decimal places: 1

**Affected By Page Filters:** Warehouse, Supplier, Supplier Type, Supplier Country.

**Validation Test:** Filtered inventory records to Low Stock and Out of Stock, then compared the distinct product-ID count with the measure then verified that products appearing in multiple at-risk positions are counted only once.

---

### Overstock Positions

**Business Definition:** Counts inventory positions classified as Overstock. Each position represents a product within a specific warehouse. The measure counts (product & warehouse) records, not distinct SKUs.

**DAX Formula:**

```dax
Overstock Positions =
CALCULATE(
    [Inventory Positions],
    Fact_Inventory[Inventory_Status] = "Overstock"
)
```

**Format & Unit:** Whole Number

**Affected By Page Filters:** Warehouse, Supplier, Supplier Type, Supplier Country.

**Validation Test:** Compare the measure against the row count of Fact_Inventory after filtering Inventory_Status to Overstock.

---

### Overstock Risk %

**Business Definition:** Calculates the percentage of inventory positions classified as Overstock out of the total inventory positions within the current filter context.

**DAX Formula:**

```dax
Overstock Risk % =
DIVIDE(
    [Overstock Positions],
    CALCULATE(
        [Inventory Positions], REMOVEFILTERS(Fact_Inventory[Inventory_Status])
    ),
    0
)
```

**Format & Unit:** Percentage (%), Decimal places: 0

**Affected By Page Filters:** Warehouse, Supplier, Supplier Type, Supplier Country.

**Validation Test:** Compare the number of Overstock positions with the total inventory positions under the same filter context.

---

### Total SKUs

**Business Definition:** Counts distinct product IDs represented in the inventory records within the current filter context.

**DAX Formula:**

```dax
Total SKUs = DISTINCTCOUNT(Fact_Inventory[product_id])
```

**Format & Unit:** Whole Number

**Affected By Page Filters:** Warehouse, Supplier, Supplier Type, Supplier Country.

**Validation Test:** Compared the measure with the distinct count of fact_inventory[product_id] under the same filter context.

---

### Unique Suppliers

**Business Definition:** Counts distinct supplier names in the supplier dimension within the current filter context.

**DAX Formula:**

```dax
Unique Suppliers = DISTINCTCOUNT(dim_suppliers[supplier_name])
```

**Format & Unit:** Whole Number

**Affected By Page Filters:** Supplier, Supplier Type, Supplier Country.

**Validation Test:** Compared the measure with the distinct count of dim_suppliers[supplier_name] and verified that supplier slicers update the result as expected.

---

### Inventory Positions (Hidden)

**Business Definition:** Counts product-warehouse inventory records.

**DAX Formula:**

```dax
Inventory Positions = COUNTROWS(Fact_Inventory)
```

**Format & Unit:** Whole Number

**Affected By Page Filters:** Warehouse, Supplier, Supplier Type, Supplier Country.

**Validation Test:** Compared the measure with the row count of Fact_Inventory under the same filter context.

---

## Logistics & Risk

### Total Risk Events

**Business Definition:** Counts the recorded risk-event records in the current filter context.

**DAX Formula:**

```dax
Total Risk Events = COUNTROWS(fact_risk_events)
```

**Format & Unit:** Whole Number

**Affected By Page Filters:** Risk Severity, Risk Category, Risk Impact Area, Risk Status.

**Validation Test:** Compared the measure with the row count of fact_risk_events under the same filter context.

---

### Total Estimated Loss

**Business Definition:** Sums the estimated financial loss recorded for risk events in the current filter context.

**DAX Formula:**

```dax
Total Estimated Loss = SUM(fact_risk_events[estimated_loss])
```

**Format & Unit:** Currency ($), Decimal places: 1

**Affected By Page Filters:** Risk Severity, Risk Category, Risk Impact Area, Risk Status.

**Validation Test:** Compared the measure with the sum of fact_risk_events[estimated_loss].

---

### Active Risk Exposure

**Business Definition:** Calculates the total estimated loss of risk events whose resolution status is not Resolved or Mitigated yet.

**DAX Formula:**

```dax
Active Risk Exposure =
CALCULATE(
    [Total Estimated Loss],
    KEEPFILTERS( dim_risk_resolution[status] IN {"Open", "Investigating"})
)
```

**Format & Unit:** Currency ($), Decimal places: 0

**Affected By Page Filters:** Risk Severity, Risk Category, Risk Impact Area, Risk Status.

**Validation Test:** Compared the measure against the sum of estimated_loss for Open and Investigating events under the same filter context, ensuring that Resolved and Mitigated events are excluded.

---

### Estimated Loss Share %

**Business Definition:** Calculates each severity level's share of the total estimated loss.

**DAX Formula:**

```dax
Estimated Loss Share % =
DIVIDE(
    [Total Estimated Loss],
    CALCULATE(
        [Total Estimated Loss],REMOVEFILTERS(dim_risk_classify[severity])),
    0
)
```

**Format & Unit:** Percentage (%), Decimal places: 0

**Affected By Page Filters:** Risk Severity, Risk Category, Risk Impact Area, Risk Status.

**Validation Check:** Confirmed that shares across all severity levels total 100% under the same filters.

---

### Risk Response Rate %

**Business Definition:** Calculates the percentage of risk events that are resolved or mitigatede regardless of the selected risk status.

**DAX Formula:**

```dax
Risk Response Rate % =
DIVIDE(
    CALCULATE(
        [Total Risk Events], dim_risk_resolution[status] IN {"Resolved","Mitigated"}
    ),
    CALCULATE(
        [Total Risk Events],
        REMOVEFILTERS(dim_risk_resolution[status])
    ),0
)
```

**Format & Unit:** Percentage (%), Decimal places: 0

**Affected By Page Filters:** Risk Severity, Risk Category, Risk Impact Area.

**Validation Check:** Compared resolved events with total risk events under the same filters.

---

### On-Time Delivery Rate %

**Business Definition:** Calculates the percentage of eligible shipments delivered on time, excluding Returned and pending shipments.

**DAX Formula:**

```dax
On-Time Delivery Rate % =
DIVIDE(
    CALCULATE(
        [Total Shipments],
        dim_shipment_tracking[is_on_time] = "Yes", dim_ship_junk[shipment_status] <> "Returned"
    ),
    CALCULATE(
        [Total Shipments],
        dim_shipment_tracking[is_on_time] IN {"Yes", "No"}, dim_ship_junk[shipment_status] <> "Returned"
    ),
    0
)
```

**Format & Unit:** Percentage (%), Decimal places: 0

**Affected By Page Filters:** None.

**Validation Test:** Manually verified That Yes/(Yes + No) calculation, excluding Returned shipments.

---

### Estimated Loss per Risk Event

**Business Definition:** Calculates the average estimated loss per risk event.

**DAX Formula:**

```dax
Estimated Loss per Risk Event =
DIVIDE([Total Estimated Loss], [Total Risk Events], 0)
```

**Format & Unit:** Currency ($), Decimal places: 0

**Affected By Page Filters:** Risk Severity, Risk Category, Risk Impact Area, Risk Status.

**Validation Check:** Verified against Total Estimated Loss divided by Total Risk Events under the same filters.

---

### On-Time Rate (1-day tolerance) (Hidden)

**Business Definition:** Calculates the percentage of eligible shipments delivered within one calendar day of the expected shipment date, excluding Returned and pending shipments.

**DAX Formula:**

```dax
On-Time Rate (1-day tolerance) =
DIVIDE(
    CALCULATE(
        [Total Shipments],
        fact_shipment[delay_days] <= 1,
        dim_shipment_tracking[is_on_time] IN {"Yes", "No"}, dim_ship_junk[shipment_status] <> "Returned"
    ),
    CALCULATE(
        [Total Shipments],
        dim_shipment_tracking[is_on_time] IN {"Yes", "No"}, dim_ship_junk[shipment_status] <> "Returned"
    ),
    0
)
```

**Format & Unit:** Percentage (%), Decimal places: 0

**Affected By Page Filters:** None.

**Validation Test:** Verified against 8,478 shipments delivered within one day of the expected shipment date out of 9,078 eligible shipments, giving a rate of 93%.

---

### Risk Events Share% (Hidden)

**Business Definition:** Calculates each severity level's share of total risk events.

**DAX Formula:**

```dax
Risk Events Share% =
DIVIDE(
    [Total Risk Events],
    CALCULATE(
        [Total Risk Events], REMOVEFILTERS(dim_risk_classify[severity])
    ),
    0
)
```

**Format & Unit:** Percentage (%), Decimal places: 0

**Affected By Page Filters:** Risk Severity, Risk Category, Risk Impact Area, Risk Status.

**Validation Check:** Confirmed that shares across all severity levels total is 100% under the same filters.

---

### Estimated Loss Per Shipment (Hidden)

**Business Definition:** Calculates the average estimated loss per shipment.

**DAX Formula:**

```dax
Estimated Loss Per Shipment = DIVIDE([Total Estimated Loss],[Total Shipments])
```

**Format & Unit:** Currency ($), Decimal places: 1

**Affected By Page Filters:** Risk Severity, Risk Category, Risk Impact Area, Risk Status.

**Validation Check:** Verified against Total Estimated Loss divided by Total number of shipments under the same filters.

---

### Total Shipments (Hidden)

**Business Definition:** represents the number of unique shipment records.

**DAX Formula:**

```dax
Total Shipments = COUNTROWS(fact_shipment)
```

**Format & Unit:** Whole Number

**Affected By Page Filters:** None.

**Validation Test:** Confirmed that shipment_id is unique in fact_shipment and that the measure matches the row count of the table.

---

**Note:** Visual interactions including filtering and highlighting between visuals are considered at the page level rather than repeated for each measure.
