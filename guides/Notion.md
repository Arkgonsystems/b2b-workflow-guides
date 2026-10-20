[← Back to All B2B Workflow Guides](../index.html) | [⚡ Get the Turnkey JSON Blueprint Pack ($19)](https://platypusprime.gumroad.com/l/dncmng)

> **FTC Affiliate Disclosure:** We are an independent B2B technical publication and affiliate marketer (not an official vendor partner). This guide contains affiliate links—if you click through and purchase or subscribe, we may earn a commission at no additional cost to you.

> **Quick-Start Stack:** Deploy this workflow with [**Automated Weekly Client KPI Executive Rollup** (1 Month Free Core Plan / 10,000 Operations)](https://www.make.com/en/register?pc=b2bworkflow) + [**Make**](https://www.make.com/en/register?pc=b2bworkflow), or download our [**Turnkey Automated Weekly Client KPI Executive Rollup + Make JSON Blueprint & Schema Pack**](https://platypusprime.gumroad.com/l/dncmng) to import the complete scenario in 60 seconds.

# client-reporting-kpi-dashboard-looker-sheets

## 1. Executive Architecture & ROI Breakdown

Client reporting and executive KPI rollups are fundamental to agency retention, yet they traditionally suffer from extreme friction. Account managers waste valuable billable hours manually aggregating metrics from HubSpot, Google Analytics, and ad networks into rigid Google Sheets before constructing Looker Studio dashboards. This manual paradigm creates severe data latency, human calculation errors, and delayed stakeholder insights.

To solve this, modern revenue operations teams implement an automated pipeline utilizing Google Sheets as an intermediary calculation engine, Looker Studio for executive visualization, and [Make](https://www.make.com/en/register?pc=b2bworkflow) as the robust middleware orchestrating the [Automated Weekly Client KPI Executive Rollup](https://www.make.com/en/register?pc=b2bworkflow). This serverless architecture eliminates human intervention, ensures zero data drift, and delivers pristine weekly metrics directly to client stakeholders every Monday at 06:00 UTC.

### Manual vs. Automated Workflow Metrics Comparison

| Operational Metric | Manual Weekly Reporting | Automated Make + Sheets + Looker Pipeline |
| :--- | :--- | :--- |
| **Labor Cost per Client / Month** | 4.5 hours @ $75/hr = $337.50 | 0.1 hours (monitoring only) = $7.50 |
| **Data Latency** | 48–72 hours post-close | Real-time / Scheduled (Max 1 hour lag) |
| **Error Rate (Human Input)** | ~12% (transposition and formula breaks) | < 0.01% (strict programmatic parsing) |
| **Time-to-Insight for C-Suite** | 5 business days into the new week | Instantaneous (Always-on Looker dashboard) |
| **Scalability Ceiling** | ~10 clients per Account Manager | Unlimited (Systematized via Make scenarios) |

---

## 2. Prerequisites & Schema Configuration

Before constructing the pipeline via [Make](https://www.make.com/en/register?pc=b2bworkflow), you must provision your data storage layers. The target Google Sheet acts as the relational staging database, while HubSpot provides the CRM telemetry. 

### Google Sheets Target Schema (`Client_KPI_Master`)

Create a master Google Sheet containing a worksheet named `WeeklyRollup` with the exact column headers below:

*   `row_id` (String): Unique compound key (`client_id` + `ISO_week`)
*   `client_id` (String): Internal agency client identifier
*   `client_name` (String): Display name of the enterprise client
*   `iso_week` (String): Year and week number (e.g., `2024-W42`)
*   `deals_closed_won` (Integer): Total deals moved to Closed-Won stage
*   `pipeline_revenue` (Float): Total monetary value of closed deals
*   `active_tickets` (Integer): Count of open support/success tickets
*   `avg_response_time_hrs` (Float): Mean customer service response time
*   `last_synced_timestamp` (ISO-8601 String): UTC execution timestamp

### HubSpot Custom Property Configuration

Ensure your HubSpot CRM enterprise or professional account contains the following custom deal and ticket properties to support the aggregation query:

1.  `hs_arr_contribution` (Number / Currency): Annual recurring revenue attached to the deal.
2.  `client_tier` (Enumeration): Values: `Enterprise`, `Mid-Market`, `SMB`.

### Webhook & API Payload Structure

The pipeline leverages scheduled cron triggers inside [Make](https://www.make.com/en/register?pc=b2bworkflow), querying HubSpot via GraphQL/REST endpoints and transforming the payload into the following JSON structure before writing to Google Sheets:

```json
{
  "client_id": "ent_994821",
  "client_name": "Acme Corporation",
  "iso_week": "2024-W42",
  "metrics": {
    "deals_closed_won": 4,
    "pipeline_revenue": 145000.00,
    "active_tickets": 2,
    "avg_response_time_hrs": 1.45
  },
  "metadata": {
    "version": "2.4",
    "execution_id": "mk_uuid_883a9bc1",
    "timestamp": "2024-10-21T06:00:00Z"
  }
}
```

---

## 3. Step-by-Step Automated Weekly Client KPI Executive Rollup + Make Scenario Build

To deploy the core pipeline using [Make](https://www.make.com/en/register?pc=b2bworkflow), follow this exact module configuration blueprint. You can also fast-track this setup by downloading our [Turnkey JSON Blueprint Pack](https://platypusprime.gumroad.com/l/dncmng).

### Module 1: Schedule Trigger (Make Native)
*   **App/Module:** `Tools` or `Cron` / `Basic Trigger`
*   **Configuration:** Set execution interval to every Monday at 05:00 UTC.

### Module 2: HubSpot CRM — Search Deals
*   **App/Module:** `HubSpot CRM` / `Search Deals`
*   **Filter Criteria:** 
    *   `dealstage` equals `closedwon`
    *   `closedate` is greater than or equal to `{% raw %}{{{% endraw %}addDays(now; -7)}}`
*   **Output Mapping:** Retrieve deal amount, associated company IDs, and close dates.

### Module 3: HubSpot CRM — Get Company Details
*   **App/Module:** `HubSpot CRM` / `Get a Company`
*   **Iterator Linkage:** Iterate over the array of company IDs retrieved from Module 2 to extract the explicit `client_name` and internal tracking tags.

### Module 4: Data Aggregator / Iterator (Make Native)
*   **App/Module:** `Flow Control` / `Text Aggregator` or `Array Aggregator`
*   **Purpose:** Group metrics by `client_id`, summing `pipeline_revenue` and counting `deals_closed_won` over the preceding 7-day rolling window.

### Module 5: Google Sheets — Add a Row
*   **App/Module:** `Google Sheets` / `Add a Row`
*   **Spreadsheet Selection:** Select your live Looker Studio data source sheet (`Client_KPI_Master`).
*   **Worksheet:** `WeeklyRollup`
*   **Column Mapping:**
    *   `row_id`: `{% raw %}{{{% endraw %}1.client_id}}-{% raw %}{{{% endraw %}formatDate(now; "YYYY-[W]WW")}}`
    *   `client_id`: `{% raw %}{{{% endraw %}1.client_id}}`
    *   `client_name`: `{% raw %}{{{% endraw %}3.properties.name}}`
    *   `iso_week`: `{% raw %}{{{% endraw %}formatDate(now; "YYYY-[W]WW")}}`
    *   `deals_closed_won`: `{% raw %}{{{% endraw %}4.total_deals}}`
    *   `pipeline_revenue`: `{% raw %}{{{% endraw %}4.total_revenue}}`
    *   `active_tickets`: `{% raw %}{{{% endraw %}HubSpot_Tickets.count}}`
    *   `avg_response_time_hrs`: `{% raw %}{{{% endraw %}HubSpot_Tickets.avg_time}}`
    *   `last_synced_timestamp`: `{% raw %}{{{% endraw %}now}}`

---

## 4. Error Handling, Rate Limits & Idempotency Guardrails

Production-grade revenue operations require resilient defensive programming to handle external API fluctuations, rate limits, and network anomalies.

### HTTP 429 Rate Limit Mitigation
HubSpot enforces strict API request limits (e.g., 100 requests per 10 seconds for standard OAuth apps). Inside your [Make](https://www.make.com/en/register?pc=b2bworkflow) scenario:
1.  Enable **Auto-commit** and set explicit **Error Handlers** (Break modules) on all HubSpot API interaction points.
2.  Configure exponential backoff retry parameters: Set initial retry delay to 10 seconds with a multiplier of 2x, capping at 3 retries before routing to an alerting branch.

### Idempotency & Deduplication Guardrails
To prevent duplicate rows from being appended to Google Sheets if a scenario is manually rerun or replayed after a failure:
*   Utilize the `row_id` compound key (`client_id` + `ISO_week`).
*   Precede the Google Sheets "Add a Row" module with a **Google Sheets — Search Rows** lookup. 
*   Attach a **Router** with a Filter: Proceed to "Add a Row" *only* if `Total Number of Results = 0`. If results > 0, execute an **Update a Row** module instead, ensuring zero duplicate records in your Looker Studio data source.

### Dead-Letter Logging & Error Notifications
*   Attach a fallback error handler route to every critical module.
*   Route unrecoverable failures into a dedicated Google Sheet tab named `Error_Logs` capturing `timestamp`, `scenario_name`, `error_message`, and `raw_payload`.
*   Trigger an immediate notification to your internal Slack operations channel via a Make Webhook module.

---

## 5. Production Deployment Checklist & Turnkey Blueprint

Before promoting your reporting automation pipeline to production, verify every component against this comprehensive deployment checklist:

1.  [ ] **Authentication Verification:** OAuth tokens for HubSpot and Google Sheets are refreshed, provisioned via dedicated service accounts, and secured with least-privilege scopes.
2.  [ ] **Schema Validation:** Google Sheet column headers precisely match the data mapping variables configured in your [Make](https://www.make.com/en/register?pc=b2bworkflow) scenario without trailing spaces.
3.  [ ] **Timezone Alignment:** Looker Studio dataset settings and Make scenario timestamps are standardized to UTC to prevent rolling window reporting discrepancies.
4.  [ ] **Dry Run Execution:** Execute a manual test run of the Make scenario using a staging client ID to confirm proper JSON payload formatting and row insertion.
5.  [ ] **Looker Studio Connection:** Refresh the Looker Studio data source connection, verifying that calculated fields (e.g., Average Deal Size derived from `pipeline_revenue / deals_closed_won`) render correctly without calculation errors.
6.  [ ] **Blueprint Backup:** Export your final production configuration. For immediate deployment without manual node arrangement, download our fully tested [Turnkey JSON Blueprint Pack](https://platypusprime.gumroad.com/l/dncmng) and import it directly into your Make workspace.

---

## Deploy This Workflow in 60 Seconds

* **Start Building Free:** [Launch Automated Weekly Client KPI Executive Rollup (10,000 Free Operations)](https://www.make.com/en/register?pc=b2bworkflow) and connect your [Make workspace](https://www.make.com/en/register?pc=b2bworkflow).
* **Skip the Manual Setup:** [Download the Turnkey JSON Blueprint & Schema Pack ($19)](https://platypusprime.gumroad.com/l/dncmng) or browse all bundles on the [Blueprint Hub](../blueprints.html).
