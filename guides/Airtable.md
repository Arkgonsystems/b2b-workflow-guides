[← Back to All B2B Workflow Guides](../index.html) | [⚡ Get the Turnkey JSON Blueprint Pack ($19)](https://platypusprime.gumroad.com/l/dncmng)

> **FTC Affiliate Disclosure:** We are an independent B2B technical publication and affiliate marketer (not an official vendor partner). This guide contains affiliate links—if you click through and purchase or subscribe, we may earn a commission at no additional cost to you.

> **Quick-Start Stack:** Deploy this workflow with [**Automated Invoice Reconciliation for Small Agencies** (1 Month Free Core Plan / 10,000 Operations)](https://www.make.com/en/register?pc=b2bworkflow) + [**Make**](https://www.make.com/en/register?pc=b2bworkflow), or download our [**Turnkey Automated Invoice Reconciliation for Small Agencies + Make JSON Blueprint & Schema Pack**](https://platypusprime.gumroad.com/l/dncmng) to import the complete scenario in 60 seconds.

# automated-invoice-reconciliation-make-airtable

Managing cash flow is the difference between agency scale and insolvency. For boutique digital agencies, consultancies, and service firms billing between $50k and $500k monthly, invoice reconciliation is often handled through manual spreadsheet exports, cross-referencing bank portals, Stripe feeds, and accounting ledgers. 

This manual operational drag introduces invoice leakage, payment allocation errors, and delayed revenue recognition. This guide details a production-grade blueprint for deploying an **[Automated Invoice Reconciliation for Small Agencies](https://www.make.com/en/register?pc=b2bworkflow)** framework using **[Make](https://www.make.com/en/register?pc=b2bworkflow)** (formerly Integromat) as the enterprise-grade integration engine and Airtable as the flexible relational ledger and exceptions clearinghouse.

*(Disclosure: Links to Make and related workflow tooling throughout this guide are affiliate links. We are not an official partner or direct representative of any vendor listed.)*

---

## 1. Executive Architecture & ROI Breakdown

Traditional accounting workflows rely on retrospective human audit. A bookkeeper or finance operations manager pulls bank statements at month-end, compares transaction references to issued invoices, enters reconciliation lines into an ERP or spreadsheet, and investigates discrepancies manually.

The automated continuous reconciliation pipeline shifts reconciliation from a batched, 30-day retrospective cycle to an event-driven, sub-minute execution model.

```
       +-------------------------------------------------------------+
       |                  INCOMING TRANSACTION INGEST                |
       |  (Stripe / Mercury / Relay / Brex / Wise Webhook / Polling) |
       +------------------------------+------------------------------+
                                      |
                                      v
       +-------------------------------------------------------------+
       |            MAKE ENGINE (ORCHESTRATION & LOGIC)               |
       |  - Cryptographic Webhook Authentication                     |
       |  - Payload Normalization & Currency Conversion               |
       |  - In-Memory Idempotency Deduplication Key Verification     |
       +------------------------------+------------------------------+
                                      |
               +----------------------+----------------------+
               |                                             |
               v                                             v
+-------------------------------+             +-------------------------------+
|     AIRTABLE RELATIONAL BASE  |             |      DISCREPANCY ROUTER       |
|  - Table: Invoices            |             |  - Tolerance Check (±$0.05)   |
|  - Table: Bank Transactions   |             |  - Partial Payments           |
|  - Table: Accounts / Clients  |             |  - Unidentified Inbound Wire  |
+-------------------------------+             +---------------+---------------+
               |                                             |
               v                                             v
+-------------------------------+             +-------------------------------+
| AUTOMATIC STATUS TRANSITIONS  |             | ESCALATION & EXCEPTION AUDIT  |
| - Marked "Paid in Full"       |             | - Slack / Teams Ops Alerts    |
| - Ledger Reference Stamped    |             | - HubSpot CRM Stage Synced    |
| - Auto-Receipt Dispatched     |             | - Discrepancy Record Created  |
+-------------------------------+             +-------------------------------+
```

### Operational Metrics: Manual vs. Automated Workflow

| Metric | Manual Reconciliation (Agency Avg) | Automated Make + Airtable Architecture | Operational Impact |
| :--- | :--- | :--- | :--- |
| **Reconciliation Latency** | 15–30 days (Month-End Close) | Near real-time (< 60 seconds) | Eliminates delayed cash flow visibility |

| **Labor Cost per Invoice** | $12.00 – $18.50 per invoice | $0.15 – $0.45 (Platform API overhead) | 96% reduction in transactional processing cost |
| **Error Rate (Mismatches)** | 3.5% – 6.2% human entry error | < 0.01% (Deterministic rule engine) | Prevents mistaken collections on paid accounts |
| **DSO (Days Sales Outstanding)** | 42 – 55 Days | 28 – 34 Days | Accelerates collections through immediate receipt |
| **Time to Reconcile 500 Txns**| 22–30 Human Labor Hours | 8–12 Minutes Compute Time | Recovers operational bandwidth for billable work |
| **Data Synchronization** | Siloed in local accounting/sheets | Synced to CRM (e.g., HubSpot) & Ledger | Unified account visibility across departments |

### ROI Projection for an Agency Processing 300 Invoices/Month

- **Manual Investment:** 300 invoices × 0.25 hours = 75 hours/month. At an internal labor cost of $45/hour (Finance Ops/Bookkeeper), annual cost = **$40,500**.
- **Automated Investment:** Make Pro tier + Airtable Team tier = ~$65/month platform costs. Annual maintenance (2 hrs/month at $45/hour) = $1,080. Total annual cost = **$1,860**.
- **Net Annual Direct Savings:** **$38,640** (95.4% direct cost reduction).
- **Secondary Gains:** Immediate churn mitigation by removing billing disputes; zeroed-out double-billing risks.

---

## 2. Prerequisites & Schema Configuration

Before deploying scenarios inside **[Make](https://www.make.com/en/register?pc=b2bworkflow)**, configure your Airtable base to act as an auditable double-entry mirror.

### Airtable Data Schema

Create an Airtable Base named: `[PROD] Finance Engine - Agency Operations`.

#### Table 1: `Invoices`
Represents receivables generated by your agency.

| Field Name | Type | Options / Configuration Details |
| :--- | :--- | :--- |
| `Invoice ID` | Single line text | Primary Field (e.g., `INV-2024-001`) |
| `Client Reference` | Link to `Accounts` | Linked to Accounts table |
| `Amount Due` | Currency | Precision: 2 decimals, Currency symbol: `$` |
| `Amount Paid` | Rollup | Linked via `Bank Transactions`, Field: `Allocated Amount`, Aggregation: `SUM(values)` |
| `Balance Outstanding`| Formula | `ROUND({Amount Due} - {Amount Paid}, 2)` |
| `Status` | Single select | Options: `Draft`, `Issued`, `Partially Paid`, `Paid in Full`, `Overpaid`, `Dispute` |
| `Due Date` | Date | ISO 8601 formatting |
| `Idempotency Hash` | Single line text | Unique indexing constraint (Formula: `Invoice ID & "-" & Amount Due`) |
| `HubSpot Deal ID` | Single line text | Cross-reference for your CRM deals (e.g., in HubSpot) |
| `Transactions` | Link to `Bank Transactions` | Allow linking to multiple records |
| `Discrepancies` | Link to `Discrepancies` | Exception monitoring table link |

#### Table 2: `Bank Transactions`
Captures raw, immutable inbound payment data from bank feeds, Stripe balance transactions, or clearing accounts.

| Field Name | Type | Options / Configuration Details |
| :--- | :--- | :--- |
| `Transaction ID` | Single line text | Primary Field (External Processor ID, e.g., `ch_3Mv...` or `WIRE-99210`) |
| `Gross Amount` | Currency | Precision: 2 decimals |
| `Fee Deducted` | Currency | Precision: 2 decimals (Processor fees) |
| `Net Amount` | Currency | Precision: 2 decimals |
| `Allocated Amount` | Currency | Amount mapped to linked invoice |
| `Payment Source` | Single select | Options: `Stripe`, `Mercury Wire`, `Relay ACH`, `Wise`, `Direct Check` |
| `Payer Name` | Single line text | Sender metadata string |
| `Raw Statement Descriptor` | Long text | Raw unstructured bank memo / description text |
| `Execution Timestamp` | Created time | System recorded date/time |
| `Matched Invoice` | Link to `Invoices` | Link to `Invoices` table |
| `Reconciliation Status` | Single select | Options: `Auto-Matched`, `Manual Review Required`, `Allocated`, `Unmatched` |
| `Deduplication Key` | Single line text | Indexed deduplication hash: `{Payment Source} & "-" & {Transaction ID}` |

#### Table 3: `Reconciliation Logs / Discrepancies`
Maintains a log of payment mismatches, partial collections, short-payments, or unmatched cash transfers.

| Field Name | Type | Options / Configuration Details |
| :--- | :--- | :--- |
| `Log ID` | Autonumber | Primary Field |
| `Associated Invoice` | Link to `Invoices` | Reference to relevant invoice |
| `Associated Transaction`| Link to `Bank Transactions`| Reference to processing event |
| `Discrepancy Type` | Single select | Options: `Underpayment (Short-Pay)`, `Overpayment`, `Currency Variance`, `Orphan Payment` |
| `Variance Amount` | Currency | Difference value calculated |
| `Resolution Status` | Single select | Options: `Open`, `Approved Variance (Written Off)`, `Client Contacted`, `Resolved` |
| `Audit Log JSON` | Long text | Complete Make operational runtime state payload |

### Inbound Webhook Payload Specification

Configure your upstream payment processor (e.g., Stripe, custom payment gateway, or corporate bank webhook) to post directly to your custom webhook module in Make.

**Expected Inbound JSON Schema (`POST` Application/JSON):**

```json
{
  "event_id": "evt_998240182419401",
  "event_type": "payment.settled",
  "timestamp": "2024-10-24T14:32:10Z",
  "data": {
    "transaction_id": "tx_stripe_8820491823",
    "processor": "Stripe",
    "gross_amount": 12500.00,
    "fee_amount": 362.50,
    "net_amount": 12137.50,
    "currency": "USD",
    "payer_name": "Acme Global Dynamics LLC",
    "memo": "Payment for INV-2024-001 - Monthly Retainer",
    "metadata": {
      "invoice_id_reference": "INV-2024-001",
      "hubspot_company_id": "992104"
    }
  }
}
```

---

## 3. Step-by-Step Scenario Build

This scenario processes incoming settlement events, cleans parameters, queries Airtable for matching records, applies reconciliation business logic, updates state vectors, and dispatches exception notifications.

```
+----------------------------------------------------------------------------------------------------+
|                                      MAKE SCENARIO SCHEMATIC                                       |
|                                                                                                    |
|  [Mod 1] Webhook Ingest                                                                            |
|          |                                                                                         |
|          v                                                                                         |
|  [Mod 2] Tools: Set Variable (Deduplication Key)                                                   |
|          |                                                                                         |
|          v                                                                                         |
|  [Mod 3] Data Store: Search Deduplication Record                                                  |
|          |                                                                                         |
|          +--- [Filter: Exists?] ---> (Yes) ---> [Mod 4] Halt Scenario (Idempotent Exit)            |
|          |                                                                                         |
|          +--- [Filter: Does Not Exist]                                                             |
|          |                                                                                         |
|          v                                                                                         |
|  [Mod 5] Data Store: Add Deduplication Record                                                     |
|          |                                                                                         |
|          v                                                                                         |
|  [Mod 6] Text Parser: Extract Pattern (Match INV-[0-9]{4}-[0-9]{3,})                              |
|          |                                                                                         |
|          v                                                                                         |
|  [Mod 7] Airtable: Search Records (Table: Invoices)                                                |
|          |                                                                                         |
|          v                                                                                         |
|  [Mod 8] Router (Reconciliation Path Selector)                                                     |
|          |                                                                                         |
|          +--- [Route 1: Exact Balance Match] -------------------------------------------------+    |
|          |    Filter: `gross_amount` = `Balance Outstanding`                                   |    |
|          |    - [Mod 9]  Airtable: Create Record (Bank Transactions)                           |    |
|          |    - [Mod 10] Airtable: Update Record (Invoices -> Status: Paid in Full)            |    |
|          |                                                                                     |    |
|          +--- [Route 2: Partial Underpayment Match] ------------------------------------------+    |
|          |    Filter: `gross_amount` < `Balance Outstanding` AND `gross_amount` > 0           |    |
|          |    - [Mod 11] Airtable: Create Record (Bank Transactions)                           |    |
|          |    - [Mod 12] Airtable: Update Record (Invoices -> Status: Partially Paid)          |    |
|          |    - [Mod 13] Airtable: Create Record (Reconciliation Logs / Discrepancies)         |    |
|          |    - [Mod 14] Slack: Send Channel Message (#finance-alerts)                         |    |
|          |                                                                                     |    |
|          +--- [Route 3: Overpayment / Unmatched Fallback] ------------------------------------+    |
|               Filter: Fallback (Otherwise)                                                         |
|               - [Mod 15] Airtable: Create Record (Bank Transactions -> Status: Unmatched)         |
|               - [Mod 16] Airtable: Create Record (Reconciliation Logs / Discrepancies)             |
|               - [Mod 17] Slack: Post Alert to Ops Controller with High-Priority Flag              |
+----------------------------------------------------------------------------------------------------+
```

### Module Configuration Runbook

#### Module 1: Webhook Listener
- **Module:** `Custom Webhook` (Core Webhooks)
- **Configuration:** Set name to `Inbound Settlement Listener`. Enable request validation headers if secret passing is supported by the processor.

#### Module 2: Data Normalization & Hashing
- **Module:** `Tools: Set Variables`
- **Variable Name:** `dedup_key`
- **Value Formulation:** 
  ```
  {% raw %}{{{% endraw %}sha256(1.data.processor + "-" + 1.data.transaction_id + "-" + toString(1.data.gross_amount))}}
  ```
- **Variable Name:** `sanitized_reference`
- **Value Formulation:** 
  ```
  {% raw %}{{{% endraw %}upper(trim(1.data.metadata.invoice_id_reference))}}
  ```

#### Module 3: Deduplication Verification
- **Module:** `Data Store: Get a Record`
- **Configuration:** Check Data Store `ProcessedTransactions` with Key `{% raw %}{{{% endraw %}2.dedup_key}}`.

#### Module 4 & 5: Guardrail & State Capture
- Place a Filter immediately following Module 3:
  - **Label:** `Proceed If Non-Duplicate`
  - **Condition:** `Key` *Does not exist*.
- If Key exists, branch termination executes gracefully.
- **Module 5:** `Data Store: Add/Replace a Record`
  - **Store:** `ProcessedTransactions`
  - **Key:** `{% raw %}{{{% endraw %}2.dedup_key}}`
  - **Values:** Field `timestamp` set to `{% raw %}{{{% endraw %}now}}`, `source` set to `{% raw %}{{{% endraw %}1.data.processor}}`.

#### Module 6: Regex Parsing Engine
- Used if metadata is absent and invoice numbers must be derived directly from raw bank descriptors.
- **Module:** `Text Parser: Match Pattern`
- **Pattern:** `(INV-\d{4}-\d{3,5})|([A-Z]{2,4}-\d{3,5})`
- **Text to Search:** `{% raw %}{{{% endraw %}ifempty(2.sanitized_reference; 1.data.memo)}}`

#### Module 7: Airtable Query Execution
- **Module:** `Airtable: Search Records`
- **Base:** `[PROD] Finance Engine - Agency Operations`
- **Table:** `Invoices`
- **Formula:** 
  ```
  {Invoice ID} = '{% raw %}{{{% endraw %}ifempty(2.sanitized_reference; 6.1)}}'
  ```
- **Max Records:** `1`

#### Module 8: Deterministic Router Configurations

##### Route A: Exact Match Reconciliation
*Filter Rules:*
- Condition 1: `{% raw %}{{{% endraw %}1.data.gross_amount}}` Equal to `{% raw %}{{{% endraw %}7.Balance Outstanding}}`
- Condition 2: `{% raw %}{{{% endraw %}7.Invoice ID}}` Exists

*Actions:*
1. **Module 9:** `Airtable: Create Record`
   - **Table:** `Bank Transactions`
   - **Fields:**
     - `Transaction ID`: `{% raw %}{{{% endraw %}1.data.transaction_id}}`
     - `Gross Amount`: `{% raw %}{{{% endraw %}1.data.gross_amount}}`
     - `Net Amount`: `{% raw %}{{{% endraw %}1.data.net_amount}}`
     - `Fee Deducted`: `{% raw %}{{{% endraw %}1.data.fee_amount}}`
     - `Allocated Amount`: `{% raw %}{{{% endraw %}1.data.gross_amount}}`
     - `Payment Source`: `{% raw %}{{{% endraw %}1.data.processor}}`
     - `Payer Name`: `{% raw %}{{{% endraw %}1.data.payer_name}}`
     - `Matched Invoice`: `[{% raw %}{{{% endraw %}7.id}}]` (Array)
     - `Reconciliation Status`: `Auto-Matched`
2. **Module 10:** `Airtable: Update Record`
   - **Table:** `Invoices`
   - **Record ID:** `{% raw %}{{{% endraw %}7.id}}`
   - **Fields:**
     - `Status`: `Paid in Full`

##### Route B: Partial Underpayment Variance
*Filter Rules:*
- Condition 1: `{% raw %}{{{% endraw %}1.data.gross_amount}}` Less than `{% raw %}{{{% endraw %}7.Balance Outstanding}}`
- Condition 2: `{% raw %}{{{% endraw %}1.data.gross_amount}}` Greater than `0`
- Condition 3: `{% raw %}{{{% endraw %}7.Invoice ID}}` Exists

*Actions:*
1. **Module 11:** `Airtable: Create Record` (Bank Transactions)
   - Same structural fields as Route A, with `Allocated Amount`: `{% raw %}{{{% endraw %}1.data.gross_amount}}` and `Reconciliation Status`: `Allocated`.
2. **Module 12:** `Airtable: Update Record` (Invoices)
   - **Record ID:** `{% raw %}{{{% endraw %}7.id}}`
   - **Fields:**
     - `Status`: `Partially Paid`
3. **Module 13:** `Airtable: Create Record` (Reconciliation Logs / Discrepancies)
   - **Associated Invoice:** `[{% raw %}{{{% endraw %}7.id}}]`
   - **Discrepancy Type:** `Underpayment (Short-Pay)`
   - **Variance Amount:** `{% raw %}{{{% endraw %}7.Balance Outstanding - 1.data.gross_amount}}`
   - **Resolution Status:** `Open`
   - **Audit Log JSON:** `{% raw %}{{{% endraw %}toString(1.data)}}`
4. **Module 14:** `Slack: Create a Message`
   - **Channel:** `#finance-reconciliation`
   - **Message:**
     ```markdown
     :warning: *Underpayment Detected on Invoice {% raw %}{{{% endraw %}7.Invoice ID}}*
{% raw %}{{{% endraw %}7.Balance Outstanding}}
{% raw %}{{{% endraw %}1.data.gross_amount}}
{% raw %}{{{% endraw %}7.Balance Outstanding - 1.data.gross_amount}}
     • Payer: {% raw %}{{{% endraw %}1.data.payer_name}}
     • Transaction ID: {% raw %}{{{% endraw %}1.data.transaction_id}}
     _Audit record created in Airtable Discrepancy Queue._
     ```

##### Route C: Orphan / Unmatched Processing Fallback
*Filter Rules:*
- Mark Route as the **Fallback route**. Triggers when no invoice is matched or the record was not found.

*Actions:*
1. **Module 15:** `Airtable: Create Record` (Bank Transactions)
   - Store transaction details with `Reconciliation Status`: `Manual Review Required`.
2. **Module 16:** `Airtable: Create Record` (Reconciliation Logs / Discrepancies)
   - Set `Discrepancy Type` to `Orphan Payment`, `Variance Amount` to `{% raw %}{{{% endraw %}1.data.gross_amount}}`.
3. **Module 17:** Dispatch urgent Slack message requesting operations to review the raw statement descriptor.

---

## 4. Error Handling, Rate Limits & Idempotency Guardrails

Building for financial workloads requires defense-in-depth architecture. If an integration engine stumbles, balances diverge and client relationships degrade.

```
[Standard Processing Execution]
             |
             v (Throws HTTP 429 / Base Locked / Network Drop)
+------------------------------------+
|     MAKE ERROR-HANDLING DIRECTIVE  |
+-----------------+------------------+
                  |
         +--------+--------+
         |                 |
         v                 v
   [Type: 429/5xx]    [Type: Data Format Validation]
         |                 |
         v                 v
   [Sleep / Retry]    [Rollback & Dead-Letter Log]
   (Exponential Backoff:   - Write to DLQ Data Store
    Base: 5s, 10s, 30s)    - Flag "Requires Human Audit"
                           - Dispatch Pager/Slack Alert
```

### Rate Limit Mitigation: Airtable 5-RPS Envelope

Airtable enforces a hard limit of **5 requests per second (RPS)** per base. Exceeding this limit returns an `HTTP 429 Too Many Requests` status, which can lock subsequent base calls for 30 seconds.

1. **Scenario Concurrency Control:** Within Make scenario settings, limit max execution parallelism:
   - Set **Max number of active cycles** to `1`. This serializes execution batches and prevents concurrent webhook executions from exceeding the 5 RPS rate envelope.
2. **Deterministic Sleep Injection:** On batch loops or multi-update sweeps, place the `Tools: Sleep` module set to `250ms` between sequential write operations.

### HTTP 429 Dynamic Retry Strategy (Break Directive)

Attach an **Error Handler** directly to every Airtable module:
1. Right-click the Airtable module -> **Add error handler**.
2. Select the **Break** directive.
3. Configure the Break directive parameters:
   - **Number of attempts:** `3`
   - **Execution interval:** `1` minute (or 5 minutes for resilient retries)
4. Enable **Automatically complete execution** to ensure transactions resume from the failed point once Airtable's throttling window expires.

### Idempotency Enforcement Architecture

Financial networks guarantee at-least-once delivery, meaning webhooks can fire more than once for the same transaction. Without deduplication keys, transactions can double-credit, corrupting your books.

Implement this validation matrix:
1. **Composite Key Generation:**
   Construct a hash string using source parameters:

2. **Make Data Store Caching:**
   Before running any reconciliation logic, test the hash against an internal Make Data Store configured with a 90-day time-to-live (TTL). If the hash is present, terminate the scenario immediately via an empty success path (`Commit`).

### Dead-Letter Queue (DLQ) Logging

When structural schema validation errors occur (e.g., an unsupported currency code like `EUR` sent to a `USD`-only base), execution must not fail silently.

1. Add an error-handling branch using the **Resume** directive.
2. Direct failure outputs to a dedicated Airtable Table: `Dead_Letter_Queue`.
3. Map the fields:
   - `Payload_Dump`: Complete JSON string of the failed payload.
   - `Error_Message`: Module execution error string.
   - `Failed_Module`: Numerical identifier of the throwing node.
   - `Retry_Resolved`: Boolean (default `false`).

---

## 5. Production Deployment Checklist & Turnkey Blueprint

Follow this deployment sequence to push your reconciliation system from staging to live operations without financial leakage.

### Pre-Flight Verification Checklist

- [ ] **Airtable API Authentication:** Configured using Personal Access Tokens (PAT) scoped exclusively to `data.records:read` and `data.records:write` permissions for security containment.
- [ ] **Field Type Integrity:** Verified that `Amount Due` and `Gross Amount` fields share identical currency base units and decimal precision.
- [ ] **Negative Value Testing:** Verified that system fees deducted at the source do not flip gross collection calculations into negative reconciliation variances.
- [ ] **Error Path Validation:** Intentionally executed a scenario run using an invalid transaction ID to ensure the Break directive and Dead-Letter Queue log properly.
- [ ] **De-duplication Verification:** Pushed the same webhook payload twice to confirm that the second execution exits silently without writing duplicates to Airtable.
- [ ] **HubSpot Deal Sync:** Confirmed updates from paid invoices propagate cleanly to associated deals in your CRM.
- [ ] **Data Store Retention Policies:** Data stores configured with adequate space to handle anticipated annual transaction volumes.

### Turnkey Deployment Blueprint

To skip manual module-by-module configuration, import our pre-configured architecture directly into your workspace:

> **Production Resource Pack:**  
> Download our ready-to-run **[Turnkey JSON Blueprint Pack](https://platypusprime.gumroad.com/l/dncmng)**.  
> It contains the complete, battle-tested Make scenario JSON schema with error-handling trees, router logic, and an Airtable Base CSV template.

#### Importing the Blueprint into Make:
1. Register or sign in to **[Make](https://www.make.com/en/register?pc=b2bworkflow)**.
2. Navigate to **Scenarios** -> **Create a new scenario**.
3. Click the three horizontal dots (`...`) in the bottom utility bar and choose **Import Blueprint**.
4. Upload the acquired JSON file from the **[Turnkey JSON Blueprint Pack](https://platypusprime.gumroad.com/l/dncmng)**.
5. Remap your Airtable connection token and target base IDs.
6. Toggle the Scenario status to **ON** and verify the active webhook endpoint.

*(Affiliate Disclosure: Links to Make and related workflow tooling are affiliate links. We are not an official partner of Make or Airtable, but recommend these platforms based on extensive testing in production environments.)*

---

## Deploy This Workflow in 60 Seconds

* **Start Building Free:** [Launch Automated Invoice Reconciliation for Small Agencies (10,000 Free Operations)](https://www.make.com/en/register?pc=b2bworkflow) and connect your [Make workspace](https://www.make.com/en/register?pc=b2bworkflow).
* **Skip the Manual Setup:** [Download the Turnkey JSON Blueprint & Schema Pack ($19)](https://platypusprime.gumroad.com/l/dncmng) or browse all bundles on the [Blueprint Hub](../blueprints.html).
