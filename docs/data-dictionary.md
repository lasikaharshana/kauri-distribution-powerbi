# Data Dictionary

Kauri Distribution semantic model. Nine source files load into eight tables, plus `DimDate` generated in DAX and `DimChannel` entered manually.

All dates in source files are `dd/mm/yyyy` and are typed using the **English (New Zealand)** locale.

---

## Fact tables

### FactOrders
**Grain:** one row per order line
**Source:** `orders_2025.csv` + `orders_2026.csv`, appended
**Rows:** 7,075
**Key:** OrderID + LineNumber

| Column | Type | Description |
|---|---|---|
| OrderID | Whole number | Sales order number |
| LineNumber | Whole number | Line sequence within the order |
| OrderDate | Date | Date the order was placed — **active** relationship to DimDate |
| PromisedDate | Date | Date promised to the customer — inactive relationship |
| ShipDate | Date | Date the order line left the warehouse — inactive relationship |
| CustomerID | Text | FK to DimCustomer |
| SKU | Text | FK to DimProduct |
| WarehouseID | Text | FK to DimWarehouse — the site that shipped the line |
| QtyOrdered | Whole number | Units the customer requested |
| QtyShipped | Whole number | Units actually shipped |
| UnitPrice | Decimal | Price per unit at time of order |
| DiscountPct | Decimal | Line discount, 0 to 0.125 |
| LineRevenue | Decimal | Calculated in Power Query: `QtyShipped × UnitPrice × (1 − DiscountPct)` |
| Delivery Band | Text | Calculated column: order-to-ship banded 0–2, 3–4, 5–7, 8+ days |
| Is Late | Text | Calculated column: "Late" when ShipDate > PromisedDate |
| Days Late | Whole number | Calculated column: days beyond PromisedDate, blank when on time |

---

### FactInventory
**Grain:** one row per SKU, per warehouse, per business day
**Source:** `inventory_daily.csv`
**Rows:** 41,664 (24 SKUs × 4 warehouses × 434 business days)
**Key:** SnapshotDate + SKU + WarehouseID

| Column | Type | Description |
|---|---|---|
| SnapshotDate | Date/Time | Close-of-day date. Weekdays only — no weekend rows |
| SKU | Text | FK to DimProduct |
| WarehouseID | Text | FK to DimWarehouse |
| OnHandQty | Whole number | Units physically in stock. Zero means a stockout, not missing data |
| OnOrderQty | Whole number | Units on open purchase orders |

**Periodic snapshot table.** Each row restates the same physical stock rather than recording a new event, so measures over it are **semi-additive** — they add across product and location but not across time. The column is typed Date/Time because incremental refresh requires it.

---

### FactPurchaseReceipts
**Grain:** one row per purchase-order line
**Source:** `purchase_receipts.csv`
**Rows:** 1,550
**Key:** PONumber + LineNumber

| Column | Type | Description |
|---|---|---|
| PONumber | Whole number | Purchase order number |
| LineNumber | Whole number | Line sequence within the PO |
| SupplierID | Text | FK to DimSupplier |
| SKU | Text | FK to DimProduct |
| WarehouseID | Text | FK to DimWarehouse — receiving site |
| OrderDate | Date | Date the PO was raised — inactive relationship to DimDate |
| PromisedDate | Date | Supplier's committed date — inactive relationship |
| ReceivedDate | Date | Date goods were received — **active** relationship |
| QtyOrdered | Whole number | Units ordered |
| QtyReceived | Whole number | Units received |
| DelayDays | Whole number | Power Query: `ReceivedDate − PromisedDate`. Negative means early |
| OnTimeFlag | Whole number | Power Query: 1 when ReceivedDate ≤ PromisedDate |
| InFullFlag | Whole number | Power Query: 1 when QtyReceived ≥ QtyOrdered |

The three flags are computed in Power Query rather than DAX because they depend only on values within the row and never on user filter context.

---

### FactTargets
**Grain:** one row per warehouse, per channel, per month
**Source:** `targets_wide.xlsx` (two sheets, unpivoted and appended)
**Rows:** 288 (4 warehouses × 3 channels × 24 months)

| Column | Type | Description |
|---|---|---|
| WarehouseID | Text | FK to DimWarehouse |
| Channel | Text | FK to DimChannel |
| TargetDate | Date | First day of the target month |
| YearMonth | Whole number | `YYYYMM` integer, used by `TREATAS` to match DimDate |
| TargetRevenue | Decimal | Monthly revenue target in NZD |

**Mixed grain.** Targets are monthly while DimDate is daily. Rather than fabricating daily rows, the `Target Revenue` measure strips the date filter and reapplies it at month level using `TREATAS`.

---

## Dimension tables

### DimDate
**Grain:** one row per date
**Source:** generated in DAX with `CALENDAR`
**Rows:** 730 — 1 Jan 2025 to 31 Dec 2026

| Column | Type | Description |
|---|---|---|
| Date | Date | Primary key. Marked as Date Table |
| Year | Whole number | Calendar year |
| MonthNumber | Whole number | 1–12. Sort column for MonthName |
| MonthName | Text | Jan, Feb… sorted by MonthNumber |
| MonthYear | Text | "Jan 2025" — sorted by YearMonth |
| Quarter | Text | Q1–Q4 |
| YearMonth | Whole number | `YYYYMM`. Sort column for MonthYear |
| DayOfWeek | Whole number | Monday = 1 |
| DayName | Text | Mon, Tue… sorted by DayOfWeek |
| IsWorkday | Boolean | TRUE Monday to Friday |
| FiscalYear | Whole number | NZ fiscal year, starting 1 April, named for the year it ends |
| FiscalQuarter | Text | FQ1–FQ4 against the April start |

Covers whole calendar years, extending past the data end date (31 Aug 2026) because time intelligence functions calculate against year boundaries.

---

### DimProduct
**Grain:** one row per SKU
**Source:** `products.csv`
**Rows:** 24

| Column | Type | Description |
|---|---|---|
| SKU | Text | Primary key |
| ProductName | Text | Display name |
| Category | Text | Chilled, Frozen, Ambient, Packaging — the temperature bands |
| ABCClass | Text | A, B, C — movement classification |
| StandardCost | Decimal | Accounting standard cost, used for inventory valuation |
| ListPrice | Decimal | Published price |

The source file contains a duplicate SKU created by storing `PrimarySupplierID` on a product dimension. A product can be dual-sourced, so supplier varies below the dimension's grain. Supplier was removed from DimProduct and the relationship between products and suppliers is expressed through `FactPurchaseReceipts`.

---

### DimCustomer
**Grain:** one row per customer
**Source:** `customers.csv`
**Rows:** 60

| Column | Type | Description |
|---|---|---|
| CustomerID | Text | Primary key |
| CustomerName | Text | Display name |
| Channel | Text | Retail, Wholesale, Online — hidden; use DimChannel |
| Region | Text | NZ region. Blanks replaced with "Unknown" |
| Tier | Text | Gold, Silver, Bronze |

---

### DimSupplier
**Grain:** one row per supplier
**Source:** `suppliers.csv`
**Rows:** 7

| Column | Type | Description |
|---|---|---|
| SupplierID | Text | Primary key |
| SupplierName | Text | Display name |
| Country | Text | NZ, AU, CN |
| StandardLeadTimeDays | Whole number | Quoted lead time. Null where not supplied — not zero, which would be a false claim |

SUP-07 has no rows in any fact table — an onboarded supplier not yet ordered from. It is retained deliberately; visuals control whether inactive members appear.

---

### DimWarehouse
**Grain:** one row per distribution centre
**Source:** `warehouses.csv`
**Rows:** 4

| Column | Type | Description |
|---|---|---|
| WarehouseID | Text | Primary key |
| WarehouseName | Text | Tauranga DC, Auckland DC, Christchurch DC, Hamilton XD |
| Region | Text | NZ region |
| Island | Text | North Island, South Island |
| CapacityPallets | Whole number | Storage capacity |

Hierarchy: Island → Region → WarehouseName. This is the table RLS filters; security propagates from here to all four fact tables.

---

### DimChannel
**Grain:** one row per sales channel
**Source:** entered manually
**Rows:** 3

| Column | Type | Description |
|---|---|---|
| Channel | Text | Retail, Wholesale, Online |

A conformed dimension added so `FactOrders` (through DimCustomer) and `FactTargets` can both be filtered by channel. `DimCustomer[Channel]` alone could not serve — it is not unique, so it cannot sit on the one side of a relationship.

---

### UserAccess
**Grain:** one row per user, per warehouse
**Source:** entered manually — hidden from report view

| Column | Type | Description |
|---|---|---|
| UserEmail | Text | User principal name |
| WarehouseID | Text | Warehouse that user may see |

Supports the dynamic RLS role. The role filters `DimWarehouse` directly using a `CALCULATETABLE` lookup rather than relying on bi-directional filtering from this table.

---

## Key measures

| Measure | Notes |
|---|---|
| `Revenue` | `SUMX` over FactOrders — row-level arithmetic before aggregation |
| `Revenue YTD` / `Revenue LY` / `Revenue MoM %` | Time intelligence, guarded by an `ISBLANK` check so future periods return blank rather than −100% |
| `Target Revenue` | `TREATAS` translates a daily filter to month level |
| `On Hand Qty` | Semi-additive, using `LASTNONBLANK` because the fact table holds weekdays only |
| `Inventory Value` | `SUMX` over SKUs — each product has its own standard cost |
| `Stockout Rate %` | Point in time, at the closing date |
| `Stockout Rate Period %` | Across all days in context — fully additive, unlike the measure above |
| `Fill Rate %` | Units shipped ÷ units ordered |
| `On Time %` | Lines shipped on or before the promised date |
| `OTIF %` | On time **and** in full — requires `FILTER` for a row-level comparison of two columns |
| `Days of Cover` | On-hand ÷ average daily usage. Joins FactInventory to FactOrders through shared dimensions |
| `Supplier OTIF %` | Same logic inbound, using the Power Query flags |

---

## Data quality defects in the source files

Eleven defects were planted and resolved. Documented here because the handling decisions are part of the work.

| Defect | File | Resolution |
|---|---|---|
| Duplicated order line | orders_2025 | Remove Duplicates on OrderID + LineNumber. Found by a Group By grain test — no profiling signal exists for this |
| Leading whitespace in WarehouseID | orders_2025 | Trim, applied to all key columns as routine |
| Currency symbol in UnitPrice | orders_2025 | Replace Values before typing, recovering the number rather than erroring the row |
| Blank QtyShipped | orders_2025 | Replaced with 0. Deleting the row would hide a fill-rate failure |
| Lowercase SKU | orders_2025 | UPPERCASE. Matters in Power Query, which is case-sensitive; the model is not |
| Trailing whitespace in ProductName | products | Trim |
| Currency symbol in StandardCost | products | Replace Values |
| Duplicate SKU | products | Supplier removed from the dimension — an attribute below its grain |
| Blank Region | customers | Replaced with "Unknown" |
| Channel case mismatch | customers | Capitalize Each Word |
| `n/a` lead time | suppliers | Replaced with null, not 0. Zero asserts same-day delivery and distorts the average |

All date columns arrive as `dd/mm/yyyy` and are typed with **Change Type → Using Locale → English (New Zealand)**. Without it, dates after the 12th error visibly and dates on or before the 12th convert silently to the wrong day.
