---
layout: default
title: "Automated Weekly Client KPI Executive Rollup"
---

# Automated Weekly Client KPI Executive Rollup Using Make and Notion

*Disclosure: This guide contains partner links for software tools we architect with. If you deploy via these links, we may earn a commission at no additional cost to you.*

---

## 1. Executive Bottleneck Analysis

Scaling B2B professional services and multi-account consultancies inevitably run into administrative drag: compiling weekly executive client reporting. When managing 25 active client accounts, account managers and operations leads spend substantial time manually collecting, validating, normalizing, and formatting cross-platform metrics into executive dashboards.

### The Manual Baseline (25 Client Accounts)

*   **Data Aggregation across Silos:** Pulling metrics from Google Analytics 4, CRM pipelines, ad platforms, and billing engines across 25 accounts at 35 minutes per account totals **14.6 hours per week**.
*   **Data Sanitization and Cross-Checking:** Correcting mismatched date ranges, currency conversions, and formatting discrepancies requires **4.2 hours per week**.
*   **Executive Synthesis and Dashboard Updates:** Manually drafting progress summaries and entering numbers into client-facing portals consumes **5.2 hours per week**.
*   **Total Manual Burden:** **24.0 hours per week** of senior operator capacity.

At a fully loaded blended internal billing rate of $125 per hour, this administrative routine costs **$3,000 weekly**, or **$156,000 annually** in non-billable overhead. Furthermore, manual reporting introduces latency and human transcription error, leading to delayed interventions on underperforming accounts.

### The Automated Architecture

Deploying an event-driven and scheduled data pipeline using [Make](https://www.make.com/en/register?pc=b2bworkflow) connected to a centralized [Notion](https://www.notion.so) Executive Workspace reduces human involvement to an exception-handling protocol. 

The automated pipeline executes every Sunday at 23:59 UTC:
1. Ingests raw weekly metrics across all client entities.
2. Normalizes schemas, calculates WoW (Week-over-Week) performance deltas, and validates statistical thresholds.
3. Conditionally routes updates to create or patch dedicated client pages in [Notion](https://www.notion.so).
4. Flags anomaly-driven records for executive review.

*   **Total Automated Processing Time:** Under 4 minutes (zero human hours).
*   **Human Exception Review:** **1.5 hours per week** dedicated strictly to flagged variance anomalies.
*   **Net Operational Yield:** **22.5 hours recovered weekly** (93.75% efficiency gain), mitigating $146,250 in annual administrative drag.

---

## 2. Architecture Blueprint

The pipeline decouples ingestion, validation, and dashboard presentation to maintain fault tolerance. If an upstream API returns an anomaly, the error-handling routing traps the failure, preserves pipeline state, and logs the incident without halting downstream operations.

```
+-------------------------------------------------------------------------+

|                        DATA EXTRACTION LAYER                            |
|  [CRMs / Ads APIs / Data Warehouse] ---> Webhook / Scheduled Cron (Make)|
+-------------------------------------------------------------------------+

                                    |
                                    v
+-------------------------------------------------------------------------+

|                      INGESTION & SANITIZATION                           |
|       Make: Module 1-3 (JSON Parser -> Variable Formatter)              |
|        - Strip non-numeric artifacts ($ / % / commas)                   |
|        - Verify ISO-8601 reporting timestamps                           |
+-------------------------------------------------------------------------+

                                    |
                                    v
+-------------------------------------------------------------------------+

|                   TRANSFORMATION & LOGIC ROUTER                         |
|       Make: Module 4-5 (Router & Mathematical Aggregator)               |
|        - Compute WoW Performance Variance: [(Current - Prior) / Prior]  |
|        - Branch: If Delta < -15% -> Flag Critical Anomaly               |
+-------------------------------------------------------------------------+

                         |                        |
            [Routine Update]             [Critical Anomaly]

                         |                        |
                         +------------+-----------+

                                      |
                                      v
+-------------------------------------------------------------------------+

|                   TARGET DESTINATION LAYER (NOTION)                     |
|  Notion API: "Query Database" -> Check Idempotency (Client ID + Week)   |
|         - If Record Exists: Execute "Update a Database Item"            |
|         - If Record Absent: Execute "Create a Database Item"            |
|                                     |                                   |
|  Target Database: [Client Weekly Executive Rollup]                     |
+-------------------------------------------------------------------------+
```

### Notion Schema Specification: "Client Weekly Executive Rollup"

The primary rollup database in [Notion](https://www.notion.so) requires explicit property configurations to support dynamic visual badges, historical tracking, and relational rollup lookups.

| Property Name | Property Type | Serialization Key | Validation Rules |
| :--- | :--- | :--- | :--- |
| Client Name | Title | `Name` | Non-empty text string; strict client naming convention |
| Reporting Week | Date | `Reporting_Week` | ISO 8601 date range (Start: Monday, End: Sunday) |
| Actual Spend | Number | `Actual_Spend` | Float format, two decimals, positive value or zero |
| Revenue Generated | Number | `Revenue_Generated`| Float format, two decimals, positive value or zero |
| Blended CAC | Number | `Blended_CAC` | Float format, two decimals, greater than zero |
| CAC Target | Number | `CAC_Target` | Float format, two decimals, baseline benchmark |
| Performance Status | Select | `Performance_Status`| Options: On Track, Caution, Critical Underperformance |
| Anomaly Detected | Checkbox | `Anomaly_Detected` | True if CAC exceeds 1.25x of CAC Target |
| Executive Summary | Rich Text | `Executive_Summary`| Max 2000 characters; auto-populated summary |
| Account Relational | Relation | `Account_Master` | Relates to Master Accounts Registry Database |

---

## 3. Step-by-Step Webhook & Module Configuration

Follow this operational flow to build the ingestion and sync pipeline using [Make](https://www.make.com/en/register?pc=b2bworkflow) and [Notion](https://www.notion.so).

```
   +-------------------+      +-------------------+      +-------------------+

   | 1. Ingest Trigger | ---> | 2. Normalize JSON | ---> | 3. Query Database |
   | Make Webhook/Cron |      | Make Tools Module |      | Notion Module     |
   +-------------------+      +-------------------+      +-------------------+

                                                                   |
                                                                   v
   +-------------------+      +-------------------+      +-------------------+

   | 6. Slack Alert    | <--- | 5. Update/Create  | <--- | 4. Router Filter  |
   | (If Anomaly=True) |      | Notion Module     |      | Idempotency Check |
   +-------------------+      +-------------------+      +-------------------+
```

### Step 1: Ingestion Endpoint
*   **Module:** `Webhooks > Custom Webhook` or `Schedule > Run Scenario Every Sunday at 23:59`.
*   If using an HTTP Webhook, configure your upstream telemetry, reporting pipeline, or reverse-ETL tool to emit an array of client objects containing `client_id`, `reporting_week_start`, `spend`, `revenue`, and `cac`.
*   Ensure IP filtering or an `x-api-key` header verification check is applied in the Webhook configuration to restrict untrusted incoming payloads.

### Step 2: In-Flight Data Transformation & Calculation
*   **Module:** `Tools > Set Multiple Variables`.
*   Calculate derived values using Make expressions to simplify the payload before sending it to Notion:
    *   **Spend Float:** `parseNumber(replace(1.spend; "/[^0-9.-]+/g"; empty); ".")`
    *   **Revenue Float:** `parseNumber(replace(1.revenue; "/[^0-9.-]+/g"; empty); ".")`
    *   **True ROAS:** `round((1.revenue / 1.spend) * 100) / 100`
    *   **CAC Deviation Ratio:** `round(((1.cac - 1.cac_target) / 1.cac_target) * 100) / 100`

### Step 3: Idempotency Verification via Notion API
To prevent duplicate rows when webhooks retry or scenarios re-run, verify existing records before creating new ones.
*   **Module:** `Notion > Search Objects / Query a Database`.
*   **Database ID:** Select your `Client Weekly Executive Rollup` database.
*   **Filter Condition:**
    *   Combine `Client Name` (Text equals `1.client_name`) **AND** `Reporting Week` (Date equals `1.reporting_week_start`).
*   Set **Limit** to `1`.

### Step 4: Conditional Routing
*   **Module:** `Flow Control > Router`.
*   **Route A (Record Exists - Update):**
    *   *Filter:* `Total number of bundles` from Module 3 *Greater than* `0`.
*   **Route B (New Record - Insert):**
    *   *Filter:* `Total number of bundles` from Module 3 *Equal to* `0`.

### Step 5: Notion Record Mutation
*   **Route A Module:** `Notion > Update a Database Item`.
    *   *Database Item ID:* Map `3.id` (retrieved from the query in Step 3).
    *   Map properties to the corresponding sanitized variables from Step 2.
*   **Route B Module:** `Notion > Create a Database Item`.
    *   *Database ID:* Select `Client Weekly Executive Rollup`.
    *   Map incoming fields:
        *   `Name` (Title) -> `1.client_name`
        *   `Actual Spend` (Number) -> `Spend Float`
        *   `Revenue Generated` (Number) -> `Revenue Float`
        *   `Blended CAC` (Number) -> `1.cac`
        *   `Anomaly Detected` (Checkbox) -> `if(CAC Deviation Ratio > 0.25; true; false)`
        *   `Performance Status` (Select) -> `if(CAC Deviation Ratio > 0.25; "Critical Underperformance"; if(CAC Deviation Ratio > 0.10; "Caution"; "On Track"))`

### Step 6: Error Directive Handling
*   Right-click the `Notion > Create/Update` modules and choose **Add error handler**.
*   Attach a `Break` directive with:
    *   *Number of attempts:* `3`
    *   *Interval between attempts:* `5` minutes
    *   *Execution route:* Fallback logging to an operations Slack webhook if failure persists.

---

## 4. Data Transformation & Validation Formula

Ingested financial and operational metrics frequently arrive with dirty strings, rogue white spaces, or null values. 

### Data Sanitization Regex (Applied within Make Tools Module)

This regular expression handles malformed currency strings (e.g., `"$ 12,450.50 USD "`, `"12450.5"`, `"N/A"`, `"--"`), returning a clean numerical string ready for mathematical evaluation:

```regex
(?<=\$)?[+-]?[0-9]{1,3}(?:,?[0-9]{3})*(?:\.[0-9]+)?(?=\s*(?:USD|EUR|[A-Z]{3}))?|^[0-9]+(?:\.[0-9]+)?$
```

**Implementation in Make variable expression:**
```text
parseNumber(replace(1.raw_spend_string; "/[^\d.-]/g"; empty); ".")
```

### Notion 2.0 Dynamic Executive Status Engine

To bypass API processing limitations and avoid cluttering scenarios with complex logic, offload status calculation to a native Notion formula. Paste this formula into the `Executive Health Engine` formula property in your [Notion](https://www.notion.so) database:

```javascript
let(
  cacVariance,
  round(((prop("Blended CAC") - prop("CAC Target")) / prop("CAC Target")) * 1000) / 10,
  roas,
  if(prop("Actual Spend") > 0, round((prop("Revenue Generated") / prop("Actual Spend")) * 100) / 100, 0),
  ifs(
    empty(prop("Actual Spend")) or empty(prop("CAC Target")),
    "⚪ INCOMPLETE DATA",
    cacVariance >= 25,
    "🔴 CRITICAL BREACH: CAC +" + format(cacVariance) + "% | ROAS: " + format(roas) + "x",
    cacVariance >= 10,
    "🟡 CAUTION: CAC +" + format(cacVariance) + "% | ROAS: " + format(roas) + "x",
    "🟢 ON TRACK: CAC " + if(cacVariance > 0, "+" + format(cacVariance), format(cacVariance)) + "% | ROAS: " + format(roas) + "x"
  )
)
```

This formula calculates variance, protects against division by zero on blank spend entries, and returns a uniform status line with color-coded alerts to prioritize executive attention.

---

## 5. Turnkey Implementation Blueprint & Deployment

Deploying this architecture from scratch involves setting up webhooks, mapping Notion schemas, and configuring error routers. To accelerate implementation, you can import pre-built, tested blueprints and database templates directly into your stack.

```
       PRE-BUILT ASSETS                   DEPLOYMENT FLOW
  +-------------------------+     +-------------------------------+

  | Turnkey Blueprint Pack  | --> | 1. Duplicate Notion DB        |
  |  - Make.com JSON file   |     | 2. Import Scenario to Make    |
  |  - Notion Schema Matrix |     | 3. Connect API Credentials    |
  |  - Anomaly Router Flow  |     | 4. Run Idempotency Test       |
  +-------------------------+     +-------------------------------+
```

### Production Deployment Checklist

1. **Deploy Notion Workspace:** Duplicate the executive rollup schema directly into your private or workspace root directory using [Notion](https://www.notion.so). Ensure all property IDs correspond directly with the schema mappings.
2. **Provision Automation Engine:** Initialize your workflow environment on [Make](https://www.make.com/en/register?pc=b2bworkflow).
3. **Import Scenario Blueprints:** Open the Make Scenario builder, click **Import Blueprint**, and upload the pre-configured workflow JSON.
4. **Establish API Connections:** Authenticate the Notion Module with an internal Integration Token configured for read, write, and update permissions on the Target Database.
5. **Execute Dry-Run Simulation:** Run a single-bundle execution using test payload data to verify the idempotency filter:
   * Verify Route B executes on initial run (Record Created).
   * Verify Route A executes on identical secondary run (Record Updated, no duplicate item inserted).

### Enterprise Turnkey Bundle

To deploy this architecture without manual module-by-module configuration, use the link below to get the fully tested blueprint package:

*   [Download the Complete Turnkey Blueprint Pack ($19)](https://platypusprime.gumroad.com/l/dncmng) — Includes the raw Make scenario JSON files, the full Notion production dashboard template with relational rollups, copy-paste formulas, and an edge-case configuration guide.
*   [Claim 1 Month of Make.com Core Free (10,000 Operations Included)](https://www.make.com/en/register?pc=b2bworkflow) to run your entire client reporting framework without hitting operation limits.

<!-- table-safe-v8 -->
