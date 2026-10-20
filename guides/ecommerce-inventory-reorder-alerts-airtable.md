---
layout: default
title: "Multi-SKU Low Inventory Reorder Forecasting System"
---

# Multi-SKU Low Inventory Reorder Forecasting System Using Airtable and Make

*Disclosure: This guide contains partner links for software tools we architect with. If you deploy via these links, we may earn a commission at no additional cost to you.*

---

## 1. Executive Bottleneck Analysis

Scaling mid-market B2B distributors, wholesale brands, and multi-channel operators typically manage between 500 and 10,000 active SKUs across disparate warehouses. In an unautomated operations stack, replenishment forecasting relies on disconnected inventory spreadsheets, periodic exports from ERPs or 3PL portals, and ad-hoc supplier lead time assumptions.

### Operational Drag: Manual vs. Automated Architecture

* **Daily Stock & Sales Aggregation:** 1.5 hours/day (7.5 hours/week) spent pulling CSVs from Shopify, Amazon, and 3PLs to reconcile on-hand vs. committed units.
* **Velocity & Lead Time Variance Math:** 1.0 hour/day (5.0 hours/week) spent manually adjusting consumption rates, factoring seasonality, and cross-referencing vendor lead times.
* **Purchase Order (PO) Drafting & Disruption Handling:** 1.25 hours/day (6.25 hours/week) compiling lines in ERPs, verifying Minimum Order Quantities (MOQs), and drafting purchase emails.
* **Total Resource Burden:** **18.75 hours/week** of high-cost operations bandwidth.
* **Automated Baseline:** **< 45 minutes/week**, strictly dedicated to reviewing system-generated Purchase Order drafts and handling supplier exceptions.

### Financial Exposure

The cost of this friction is asymmetrical:
1. **Stockouts:** Average revenue loss of 4.1% to 6.8% on top-line GMV due to unfulfilled wholesale agreements and dead stock ranks on digital channels.
2. **Over-Ordering Working Capital Drag:** Carrying costs average 20% to 30% annually per surplus SKU unit due to tied-up cash flow, warehouse storage fees, and dead inventory risk.

Transitioning to an automated forecasting and automated reordering engine using [Airtable](https://www.airtable.com) and [Make](https://www.make.com/en/register?pc=b2bworkflow) eliminates manual spreadsheet reconciliation, continuously recomputes dynamic safety stock, and creates ready-to-issue POs autonomously.

---

## 2. Architecture Blueprint

The system decouples inventory tracking into three operational layers: **State Capture**, **Deterministic Calculation**, and **Autonomous Execution**.

### ASCII Data-Flow Diagram

```
+-----------------------------------------------------------------------------------+

|                            INGESTION & VELOCITY LAYER                             |
+-----------------------------------------------------------------------------------+
  [ Shopify / ERP / WMS ]                [ 3PL / Carrier Webhooks ]

             |                                       |
             v                                       v
  [ Daily Ledger Ingestion ]             [ Lead Time Actuals Ingestion ]
             +-------------------+-------------------+

                                 |
                                 v
+-----------------------------------------------------------------------------------+

|                        CENTRAL RECORD: AIRTABLE DATABASE                          |
+-----------------------------------------------------------------------------------+
  +--------------------+       +--------------------+       +---------------------+

  |   Inventory_SKUs   | <---> |    Sales_Ledger    |       |      Suppliers      |
  |--------------------|       |--------------------|       |---------------------|
  | - On-Hand Stock    |       | - SKU Link         |       | - Vendor Profile    |
  | - Allocated Units  |       | - Units Sold       |       | - Published Lead Tm |
  | - Dynamic ROP      |       | - Date Logged      |       | - Historic Variance |
  +--------------------+       +--------------------+       +---------------------+
            ^                                                          ^

            |                                                          |
            +--------------------------+-------------------------------+
                                       v
                           +------------------------+

                           |    Purchase_Orders     |
                           |------------------------|
                           | - PO Number (Autonum)  |
                           | - Line Items / Units   |
                           | - Status: Draft/Sent   |
                           +------------------------+

                                       |
+-----------------------------------------------------------------------------------+

|                       ORCHESTRATION LAYER: MAKE.COM CORE                          |
+-----------------------------------------------------------------------------------+
  [ Daily Scheduled Run: 04:00 UTC ]

             |
             v
  [ Airtable Module: Search SKUs needing Reorder (Stock <= Dynamic ROP) ]

             |
             +---> [ Filter: Active Supplying Vendor AND No Open PO Pending ]

                         |
                         v
  [ Iterator / Aggregator: Group Low SKUs by Supplier ]

                         |
                         v
  [ Airtable Module: Generate PO Header & Line Items ]

                         |
                         v
  [ Slack / Email Module: Alert Procurement Director with Approval Link ]
```

### Exact Schema Architecture

The following tables must be provisioned within a single base on [Airtable](https://www.airtable.com).

| Field Name | Table | Data Type | Validation Rules / Formulas |
| :--- | :--- | :--- | :--- |
| SKU | Inventory_SKUs | Single line text | Primary key; Regex match: `^[A-Z0-9]{3,4}-[A-Z0-9]{3,4}$` |
| Status | Inventory_SKUs | Single select | Options: Active, Discontinued, Phase-Out |
| Stock_On_Hand | Inventory_SKUs | Number (Integer) | Value >= 0 |
| Stock_Allocated | Inventory_SKUs | Number (Integer) | Value >= 0; Unshipped committed orders |
| Stock_Available | Inventory_SKUs | Formula | `{Stock_On_Hand} - {Stock_Allocated}` |
| Lead_Time_Days | Inventory_SKUs | Rollup | Target: `Suppliers.Avg_Lead_Time_Days`; Default: 14 |
| Target_Service_Level | Inventory_SKUs | Percent | Default: 95% (Z-Factor: 1.645) |
| Sales_Velocity_30D | Inventory_SKUs | Rollup | SUM of `Sales_Ledger.Units_Sold` over last 30 days / 30 |
| Sales_StdDev_30D | Inventory_SKUs | Rollup | Standard Deviation of `Sales_Ledger.Units_Sold` over last 30 days |
| Safety_Stock | Inventory_SKUs | Formula | `ROUNDUP(1.645 * SQRT(({Lead_Time_Days} * POWER({Sales_StdDev_30D}, 2))), 0)` |
| Dynamic_ROP | Inventory_SKUs | Formula | `ROUNDUP(({Sales_Velocity_30D} * {Lead_Time_Days}) + {Safety_Stock}, 0)` |
| Reorder_Flag | Inventory_SKUs | Formula | `IF(AND({Status}="Active", {Stock_Available}<={Dynamic_ROP}), 1, 0)` |
| Open_PO_Count | Inventory_SKUs | Rollup | Count of `Purchase_Orders` linked where Status != "Received" |
| Supplier_Link | Inventory_SKUs | Link to Record | Links to `Suppliers` table (Single Record) |
| Order_Quantity_MOQ | Inventory_SKUs | Number (Integer) | Minimum Order Quantity; Value >= 1 |
| Vendor_Name | Suppliers | Single line text | Primary key; Unique constraint |
| Contact_Email | Suppliers | Email | Valid RFC 5322 address |
| Avg_Lead_Time_Days | Suppliers | Number (Integer) | Value >= 1 |
| PO_Number | Purchase_Orders | Autonumber | Primary key; Format: Prefix `PO-` |
| Supplier_Ref | Purchase_Orders | Link to Record | Links to `Suppliers` table (Single Record) |
| PO_Status | Purchase_Orders | Single select | Options: Draft, Awaiting Approval, Sent, Received |
| Line_Items | Purchase_Orders | Link to Record | Links to `Inventory_SKUs` (Multiple Records) |

---

## 3. Step-by-Step Webhook & Module Configuration

Follow this end-to-end setup to orchestrate the forecasting and reorder scenario in [Make](https://www.make.com/en/register?pc=b2bworkflow).

### Step 1: Scenario Trigger & Ingestion Optimization

1. Create a new scenario in [Make](https://www.make.com/en/register?pc=b2bworkflow).
2. Add a **Schedule** trigger. Set execution frequency to **Every Day at 04:00 UTC**. Running early morning ensures inventory syncs from midnight 3PL syncs are settled.

```json
{
  "schedule": {
    "type": "cron",
    "time": "04:00",
    "timezone": "UTC"
  }
}
```

### Step 2: Fetch Low-Stock Records via Airtable Search

1. Add the **Airtable > Search Records** module.
2. Select your Base and choose table `Inventory_SKUs`.
3. In the **Formula** field, pass a strict filter to fetch records requiring purchase intervention without active reorders:

```text
AND({Reorder_Flag} = 1, {Open_PO_Count} = 0, {Status} = 'Active')
```

4. Set **Max Records** to `500`. Ensure fields returned include: `SKU`, `Stock_Available`, `Dynamic_ROP`, `Sales_Velocity_30D`, `Order_Quantity_MOQ`, and `Supplier_Link`.

### Step 3: Group Records by Supplier (Data Aggregation)

Because multiple SKUs share single suppliers, drafting individual POs per SKU floods vendors and increases shipping overhead.

1. Add a **Flow Control > Numeric Aggregator** or **Array Aggregator** directly after the Airtable module.
2. Set the **Source Module** to the `Airtable > Search Records` step.
3. Group by: `Supplier_Link[1]` (the Record ID of the linked supplier).
4. Aggregate attributes into a structured array:
   * `SKU_Record_ID`: `id`

<!-- table-safe-v4 -->
