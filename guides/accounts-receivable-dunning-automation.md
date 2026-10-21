---
layout: default
title: "Zero-Touch B2B Accounts Receivable Follow-Up Engine — Arkgon Systems"
---
> **FTC Affiliate Disclosure:** Arkgon Systems is an independent B2B technical publication and affiliate marketer (not an official vendor partner). This guide contains affiliate links—if you click through and purchase or subscribe, we may earn a commission at no additional cost to you.

> **Implementation Stack:** Deploy this architecture using [**Make** (1 Month Free Core Plan / 10,000 Operations)](https://www.make.com/en/register?pc=b2bworkflow) and [**HubSpot**](https://www.hubspot.com/products/crm), or download the ready-to-import [**Arkgon Make + HubSpot JSON Blueprint & Schema Pack ($19)**](https://platypusprime.gumroad.com/l/dncmng).

# Zero-Touch B2B Accounts Receivable Follow-Up Engine

*Published by Arkgon Systems Engineering Group*  
*Notice: This guide contains affiliate links to third-party integration platforms and CRM software, including Make and HubSpot. Arkgon Systems is an independent automation consultancy and is not an official partner, joint venture, or direct subsidiary of any referenced software vendor.*

## 2. Prerequisites & Schema Configuration

To run this pipeline without manual intervention, you must set up the exact custom properties in [HubSpot](https://www.hubspot.com/products/crm) and define an explicit JSON contract between your billing provider (e.g., Stripe Billing, Chargebee, NetSuite, or QuickBooks Online) and the [Make](https://www.make.com/en/register?pc=b2bworkflow) middleware.

### HubSpot CRM Property Definitions

You can track AR data using HubSpot standard Deals, standard HubSpot Invoices, or a dedicated Custom Object named `Invoices` (`2-xxxxxx`). The schema below targets a production **Custom Object** or **Deal Extension** model.

Create the following property group: `accounts_receivable_metadata`.

| Field Label | Internal Name | Field Type | Target Object | Description / Allowed Values |
| :--- | :--- | :--- | :--- | :--- |
| **Invoice Identifier** | `invoice_id` | Single-line text | Invoices / Deals | Unique ERP Primary Key (e.g., `INV-2024-0988`). Marked as unique. |
| **Invoice Status** | `invoice_status` | Dropdown select | Invoices / Deals | `open`, `paid`, `past_due`, `uncollectible`, `void` |
| **Invoice Total Amount** | `invoice_total_amount` | Number (Currency) | Invoices / Deals | Gross invoice value in base operational currency. |
| **Invoice Balance Due** | `invoice_balance_due` | Number (Currency) | Invoices / Deals | Real-time outstanding balance balance remaining. |
| **Invoice Due Date** | `invoice_due_date` | Date picker | Invoices / Deals | Midnight UTC timestamp of statutory due date. |
| **Days Past Due (DPD)** | `ar_days_past_due` | Number | Invoices / Deals | Integer calculating delta between current date and due date. |
| **Dunning Stage** | `ar_dunning_stage` | Dropdown select | Invoices / Deals | `stage_0_predue`, `stage_1_friendly`, `stage_2_urgent`, `stage_3_default_notice`, `stage_4_collections` |
| **Last Dunning Notice Sent** | `last_dunning_notice_date` | Date picker | Invoices / Deals | Date stamp of most recently emitted customer reminder. |
| **Payment Portal URL** | `payment_portal_url` | Single-line text | Invoices / Deals | Direct authenticated deep-link to the customer payment gateway. |
| **AR Escalation Assignee** | `ar_escalation_assignee` | Single-line text / User | Invoices / Deals | HubSpot User ID of the Account Executive or Collector handling the account. |
| **Idempotency Key** | `ar_last_idempotency_hash` | Single-line text | Invoices / Deals | SHA-256 state tracking hash to prevent duplicate automated dunning. |

### Webhook Ingestion Contract (JSON Payload Schema)

Your billing application or ERP system must publish an HTTP `POST` webhook to Make's Custom Webhook listener whenever an invoice is created, transitions to past due, or receives a payment.

```json
{
  "event_id": "evt_998374a8f902b11c",
  "event_type": "invoice.payment_overdue",
  "timestamp": 1718016000,
  "data": {
    "invoice_id": "INV-10948",
    "external_customer_id": "CUST-8831",
    "customer_email": "accounts.payable@acmecorp.com",
    "customer_company_name": "Acme Industrial Corporation",
    "issue_date": "2024-05-10T00:00:00Z",
    "due_date": "2024-06-09T00:00:00Z",
    "currency": "USD",
    "total_amount": 14500.00,
    "balance_due": 14500.00,
    "payment_portal_url": "https://billing.provider.com/pay/inv_session_998374a8f",
    "line_items": [
      {
        "description": "Enterprise Platform License - Q2",
        "quantity": 1,
        "unit_price": 14500.00,
        "amount": 14500.00
      }
    ]
  }
}
```

---

## 3. Step-by-Step Make + HubSpot Scenario Build

This end-to-end integration scenario parses incoming ERP webhook payloads, checks for existing CRM records, calculates real-time aging parameters, and routes the invoice through a dynamic multi-tiered dunning state machine.

```
+--------------------------------------------------------------------------------------------------+
|                                  MAKE SCENARIO TOPOLOGY                                          |
|                                                                                                  |
| [Mod 1: Custom Webhook]                                                                          |
|       |                                                                                          |
| [Mod 2: Data Store (Idempotency Lookup)]                                                         |
|       |                                                                                          |
| [Mod 3: HubSpot (Search CRM Objects: Invoices)]                                                  |
|       |                                                                                          |
| [Mod 4: HubSpot (Upsert Company/Contact Link)]                                                   |
|       |                                                                                          |
| [Mod 5: Tools (Date Variable / DPD Calculator)]                                                  |
|       |                                                                                          |
| [Mod 6: Flow Control (Router)]                                                                   |
|       |                                                                                          |
|       |-- Branch A: Balance == 0 (Paid / Reconciled)                                             |
|       |     `--> [Mod 7: HubSpot Update Object: Clear AR flags]                                  |
|       |                                                                                          |
|       |-- Branch B: DPD <= 0 (Stage 0: 3-Day Courtesy Notice)                                    |
|       |     `--> [Mod 8: HubSpot Transactional Email / Task Generation]                          |
|       |                                                                                          |
|       |-- Branch C: DPD between 1 and 7 (Stage 1: Soft Reminder)                                 |
|       |     `--> [Mod 9: HubSpot Transactional Email API Dispatch]                               |
|       |                                                                                          |
|       |-- Branch D: DPD between 8 and 21 (Stage 2: Firm Notice + AE Notification)                |
|       |     `--> [Mod 10: HubSpot Send Email] ---> [Mod 11: HubSpot Create Task for AE]          |
|       |                                                                                          |
|       \-- Branch E: DPD >= 22 (Stage 3+: Default Notice + Credit Freeze)                         |
|             `--> [Mod 12: HubSpot Send Demand Email] ---> [Mod 13: HubSpot Set Company Hold]     |
|                                                                                                  |
| [Mod 14: Data Store (Save Idempotency Key)]                                                      |
+--------------------------------------------------------------------------------------------------+
```

### Module 1: Webhook Listener & Ingestion Gate
*   **Platform Engine:** [Make](https://www.make.com/en/register?pc=b2bworkflow)
*   **Module Type:** `Custom Webhook` (`gateway-webhook`)
*   **Configuration:**
    *   Set **IP Whitelisting** to your ERP/Billing Gateway CIDR blocks (e.g., Stripe webhook IP ranges).
    *   Enable **JSON pass-through**.
    *   Set **Strict Validation** to active.

### Module 2: Deduplication via Make Data Store
*   **Module Type:** `Data Store: Search Records`
*   **Data Store Name:** `AR_Idempotency_Cache`
*   **Key Expression:**  
    `sha256(1.data.invoice_id + ":" + 1.data.balance_due + ":" + formatDate(now; "YYYY-MM-DD"))`
*   **Filter Logic:**  
    *Condition:* If record `Key` **Exists**, proceed to an immediate `Commit & Stop` directive. Do not re-notify customers if the exact balance status has already been processed today.

### Module 3: HubSpot Entity Search
*   **Platform Engine:** [HubSpot](https://www.hubspot.com/products/crm)
*   **Module Type:** `HubSpot CRM: Search Objects`
*   **Target Object Type:** `Invoices` (or `Deals` if using the Deal architecture)
*   **Search Filters:**
    ```json
    {
      "filterGroups": [
        {
          "filters": [
            {
              "propertyName": "invoice_id",
              "operator": "EQ",
              "value": "{% raw %}{{{% endraw %}1.data.invoice_id}}"
            }
          ]
        }
      ],
      "properties": [
        "hs_object_id",
        "invoice_id",
        "invoice_status",
        "invoice_balance_due",
        "ar_dunning_stage",
        "last_dunning_notice_date"
      ]
    }
    ```

### Module 4: Dynamic DPD Computation & Normalization
*   **Module Type:** `Tools: Set Variable`
*   **Variable Name:** `days_past_due`
*   **Formula Calculation:**
    ```javascript
    floor((parseDate(1.data.due_date; "YYYY-MM-DDTHH:mm:ssZ") - now) / -86400000)
    ```
    *Note: A positive number indicates the invoice is overdue. A negative number indicates an upcoming due date.*

### Module 5: Strategic Routing & Escalation Logic
Add a **Router** module with five discrete paths. Configure execution limits and conditional filters explicitly on each branch.

---

#### Route A: Paid & Reconciled (Balance Resolved)
*   **Condition Filter:** `{% raw %}{{{% endraw %}1.data.balance_due}} <= 0` OR `{% raw %}{{{% endraw %}1.data.event_type}} = "invoice.paid"`
*   **Step A1:** `HubSpot: Update an Object`
    *   `ObjectID`: `{% raw %}{{{% endraw %}3.hs_object_id}}`
    *   `invoice_status`: `paid`
    *   `invoice_balance_due`: `0`
    *   `ar_dunning_stage`: `resolved`
*   **Step A2:** `HubSpot: Create an Engagement (Timeline Note)`
    *   `Note Body`: `"Invoice {% raw %}{{{% endraw %}1.data.invoice_id}} fully settled. Automation lock released."`

---

#### Route B: Stage 0 – Pre-Due Courtesy (DPD: -3 to 0)
*   **Condition Filter:** `{% raw %}{{{% endraw %}days_past_due}} >= -3` AND `{% raw %}{{{% endraw %}days_past_due}} <= 0` AND `{% raw %}{{{% endraw %}1.data.balance_due}} > 0`
*   **Step B1:** `HubSpot: Send Transactional Email`
    *   `To`: `{% raw %}{{{% endraw %}1.data.customer_email}}`
    *   `Email ID / Template ID`: Select your configured pre-due reminder template (e.g., `88910248`).
    *   `Custom Properties Array`:
        ```json
        {
          "invoice_id": "{% raw %}{{{% endraw %}1.data.invoice_id}}",
          "amount_due": "{% raw %}{{{% endraw %}formatNumber(1.data.balance_due; 2; \".\"; \",\")}}",
          "due_date": "{% raw %}{{{% endraw %}formatDate(1.data.due_date; \"MMMM DD, YYYY\")}}",
          "payment_link": "{% raw %}{{{% endraw %}1.data.payment_portal_url}}"
        }
        ```
*   **Step B2:** `HubSpot: Update an Object`
    *   `ar_dunning_stage`: `stage_0_predue`
    *   `last_dunning_notice_date`: `{% raw %}{{{% endraw %}now}}`

---

#### Route C: Stage 1 – Soft Overdue Reminder (DPD: 1 to 7)
*   **Condition Filter:** `{% raw %}{{{% endraw %}days_past_due}} >= 1` AND `{% raw %}{{{% endraw %}days_past_due}} <= 7` AND `{% raw %}{{{% endraw %}1.data.balance_due}} > 0`
*   **Step C1:** `HubSpot: Send Transactional Email`
    *   `Email ID / Template ID`: `88910249` (Stage 1 Soft Reminder)
    *   `Tokens`: Include statement reconciliation table and updated payment link.
*   **Step C2:** `HubSpot: Update an Object`
    *   `ar_dunning_stage`: `stage_1_friendly`
    *   `last_dunning_notice_date`: `{% raw %}{{{% endraw %}now}}`

---

#### Route D: Stage 2 – Firm Warning & AE Engagement (DPD: 8 to 21)
*   **Condition Filter:** `{% raw %}{{{% endraw %}days_past_due}} >= 8` AND `{% raw %}{{{% endraw %}days_past_due}} <= 21` AND `{% raw %}{{{% endraw %}1.data.balance_due}} > 0`
*   **Step D1:** `HubSpot: Send Transactional Email`
    *   `Email ID / Template ID`: `88910250` (Stage 2 Escalation Warning)
*   **Step D2:** `HubSpot: Create a Task`
    *   `Task Title`: `"AR ESCALATION: {% raw %}{{{% endraw %}1.data.customer_company_name}} is {% raw %}{{{% endraw %}days_past_due}} Days Overdue"`
    *   `Priority`: `HIGH`
    *   `Due Date`: `{% raw %}{{{% endraw %}addDays(now; 1)}}`
    *   `Owner ID`: Associated Account Owner / AE (`{% raw %}{{{% endraw %}3.hubspot_owner_id}}`)
{% raw %}{{{% endraw %}1.data.balance_due}} on Invoice {% raw %}{{{% endraw %}1.data.invoice_id}}."`
*   **Step D3:** `HubSpot: Update an Object`
    *   `ar_dunning_stage`: `stage_2_urgent`
    *   `last_dunning_notice_date`: `{% raw %}{{{% endraw %}now}}`

---

#### Route E: Stage 3/4 – Default Warning & Credit Freeze (DPD: >= 22)
*   **Condition Filter:** `{% raw %}{{{% endraw %}days_past_due}} >= 22` AND `{% raw %}{{{% endraw %}1.data.balance_due}} > 0`
*   **Step E1:** `HubSpot: Send Transactional Email`
    *   `Email ID / Template ID`: `88910251` (Formal Notice of Impending Credit Suspension)
    *   `CC`: `ar-escalations@yourdomain.com`
*   **Step E2:** `HubSpot: Update Company Object`
    *   `Associated Company ID`: `{% raw %}{{{% endraw %}3.associations.companies.results[1].id}}`
    *   `account_status`: `CREDIT_SUSPENDED`
    *   `ar_hold_active`: `true`
*   **Step E3:** `HubSpot: Create a Task`
    *   `Task Title`: `"CRITICAL: Send to Third-Party Collections Agency / Legal Review"`
    *   `Priority`: `URGENT`
    *   `Owner ID`: AR Finance Director ID
*   **Step E4:** `HubSpot: Update an Object`
    *   `ar_dunning_stage`: `stage_3_default_notice`
    *   `last_dunning_notice_date`: `{% raw %}{{{% endraw %}now}}`

---

## 4. Error Handling, Rate Limits & Idempotency Guardrails

Enterprise dunning systems directly impact cash flow and customer trust. An unhandled loop can spam customers with collection notices, while a failed webhook can drop an overdue balance entirely.

```
+--------------------------------------------------------------------------------------------------+
|                            ERROR HANDLING & RETRY TOPOLOGY                                       |
|                                                                                                  |
| [HubSpot API Call]                                                                               |
|         |                                                                                        |
|         +--- Success ---> [Continue Workflow]                                                    |
|         |                                                                                        |
|         \--- Error: HTTP 429 / 5xx                                                               |
|                   |                                                                              |
|                   v                                                                              |
|           [Error Directive: Break]                                                               |
|             - Attempts: 5                                                                        |
|             - Interval: Exponential Backoff (1m, 2m, 4m, 8m, 16m)                                |
|                   |                                                                              |
|                   +--- Retries Succeeded ---> [Continue Workflow]                                |
|                   |                                                                              |
|                   \--- Retries Exhausted                                                         |
|                             |                                                                    |
|                             v                                                                    |
|                     [Dead-Letter Logging]                                                        |
|                       `--> SQS / Cold Store + Slack Alert                                        |
+--------------------------------------------------------------------------------------------------+
```

### 1. HubSpot Rate Limit Defense (HTTP 429 Handling)
HubSpot applies rate limits based on account tier: standard Private Apps permit up to 100 requests per 10 seconds, with total daily limits scaling by subscription.

To prevent rate-limit failures:
*   Attach a **Break** error handler to all HubSpot modules.
*   Configure the handler parameters:
    *   **Number of attempts:** `5`
    *   **Execution interval:** Exponential backoff, starting at `1` minute, doubling every retry (`1m, 2m, 4m, 8m, 16m`).
*   Activate **Sequential Execution** in the [Make Scenario Settings](https://www.make.com/en/register?pc=b2bworkflow) to prevent concurrent executions from saturating your burst API quota.

### 2. Idempotency Key Architecture
To prevent duplicate dunning notices from network retries or redelivered webhooks:

1.  Construct a unique transactional string:

2.  Store this key in a low-latency cache (Make Data Store or Redis instance) with a 24-hour Time-to-Live (TTL).
3.  Check the cache before dispatching any transactional email. If the hash matches an existing record, exit the execution path immediately (`Commit Directive`).

### 3. Dead-Letter Queue (DLQ) & Alerting Pipeline
When an unhandled exception occurs (such as an invalid contact email or missing company association):

```json
{
  "dlq_meta": {
    "execution_id": "{% raw %}{{{% endraw %}var.execution_id}}",
    "timestamp_utc": "{% raw %}{{{% endraw %}now}}",
    "scenario_id": "{% raw %}{{{% endraw %}var.scenario_id}}",
    "error_message": "{% raw %}{{{% endraw %}1.error.message}}",
    "error_type": "{% raw %}{{{% endraw %}1.error.type}}",
    "failed_payload": "{% raw %}{{{% endraw %}1.data}}"
  }
}
```

*   Attach an **Error Handler Route** to the scenario router.
*   Route failed payloads directly to an AWS SQS queue or append them to a secured **Dead-Letter Audit Log**.
*   Trigger an immediate alert via incoming Webhook to your internal Slack or Microsoft Teams engineering room (`#ops-ar-telemetry`).

---

## 5. Production Deployment Checklist & Turnkey Blueprint

Follow this cutover protocol to move the integration from sandbox testing to production.

```
+--------------------------------------------------------------------------------------------------+
|                              PRODUCTION DEPLOYMENT GANTT                                         |
|                                                                                                  |
| Task                                            D-5   D-4   D-3   D-2   D-1   GO-LIVE  POST-CUT  |
| ------------------------------------------------------------------------------------------------ |
| 1. HubSpot Schema & Scopes Provisioning          [X]                                             |
| 2. Private App Generation & Token Secure Store         [X]                                       |
| 3. Dynamic Email Template Assembly                           [X]                                 |
| 4. Make Scenario Import & Data Store Init                         [X]                            |
| 5. Deterministic Payload Unit Testing                                   [X]                      |
| 6. Dark Launch (Shadow Execution, No Outbound)                                  [X]              |
| 7. Production Cutover & Live Webhook Ingestion                                          [X]       |
| 8. Telemetry Monitoring & DLQ Audit Review                                              [=====>] |
+--------------------------------------------------------------------------------------------------+
```

### Phase 1: Environment & Token Provisioning
- [ ] Create a dedicated **HubSpot Private App** within your production [HubSpot CRM](https://www.hubspot.com/products/crm) portal.
- [ ] Grant only the minimal required scopes:
  - `crm.objects.custom.read` & `crm.objects.custom.write`
  - `crm.objects.deals.read` & `crm.objects.deals.write`
  - `crm.objects.contacts.read` & `crm.objects.contacts.write`
  - `crm.objects.companies.read` & `crm.objects.companies.write`
  - `transactional-email`
- [ ] Store API keys and webhook secrets inside [Make Variables](https://www.make.com/en/register?pc=b2bworkflow) or AWS Secrets Manager. Never hardcode plaintext authentication tokens in module configurations.

### Phase 2: Template Assembly
- [ ] Build transactional email templates in HubSpot with token bindings for:
  - `{% raw %}{{{% endraw %}contact.firstname}}`
  - `{% raw %}{{{% endraw %}deal.invoice_id}}`
  - `{% raw %}{{{% endraw %}deal.invoice_balance_due}}`
  - `{% raw %}{{{% endraw %}deal.invoice_due_date}}`
  - `{% raw %}{{{% endraw %}deal.payment_portal_url}}`
- [ ] Test template rendering across desktop and mobile clients (including Dark Mode styling) to ensure payment links render correctly.

### Phase 3: Sandbox Testing Protocol
- [ ] Run a test suite using synthetic payloads to verify scenario routing:
  - **Test Case Alpha:** Run a standard on-time transaction (`balance_due = 0`). Ensure Route A executes and closes the record without sending an email.
  - **Test Case Beta:** Run a near-due invoice (`DPD = -2`). Verify Route B sends the courtesy email and sets the dunning stage to `stage_0_predue`.
  - **Test Case Gamma:** Run an overdue invoice (`DPD = 14`). Verify Route D triggers the escalation email and creates the high-priority task for the account owner.
  - **Test Case Delta:** Run a severely delinquent account (`DPD = 35`). Confirm the system sets the account to `CREDIT_SUSPENDED` and tags the Finance Director.
- [ ] Verify idempotency handling: Replay Test Case Gamma within 60 seconds. Confirm the second execution drops the notification path and triggers no duplicate emails.

### Phase 4: Production Cutover
- [ ] In your ERP or billing gateway, update the webhook destination URL to the production Make endpoint.
- [ ] Enable scenario scheduling: set the execution mode to **Immediately as data arrives**.
- [ ] Turn on the scenario toggle in Make.
- [ ] Run a live invoice validation test by generating a $1.00 staging invoice against an internal sandbox account. Verify processing end-to-end.

---

### Turnkey Deployment Asset

To skip manual module configuration, import our pre-configured scenario blueprint directly into your environment.

Download the **[Arkgon Turnkey AR Engine JSON Blueprint Pack](https://platypusprime.gumroad.com/l/dncmng)**.

This asset bundle includes:
1. Complete, exportable **Make Scenario JSON files** containing all 14 modules, complete routing expressions, and error-handling topologies.
2. The custom **HubSpot Property Definition Script** for one-click schema setup via the HubSpot API.
3. Four responsive, pre-built **HTML Transactional Dunning Templates** designed for Stage 0 through Stage 4 escalations.
4. A Postman collection containing complete synthetic test suites for local and sandbox validation.

---

## Deploy This Architecture

* **Provision Free Trial Workspaces:** [Create a Make Account (10,000 Free Operations)](https://www.make.com/en/register?pc=b2bworkflow) and connect your [HubSpot database](https://www.hubspot.com/products/crm).
* **Import the Production Scenario:** [Download the Arkgon Complete JSON Blueprint & Schema Pack ($19)](https://platypusprime.gumroad.com/l/dncmng) or view all systems in the [Arkgon Blueprint Directory](../blueprints.html).
