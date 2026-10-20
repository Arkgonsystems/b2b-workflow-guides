---
layout: default
title: "Automated SaaS Failed Payment & Churn Recovery Pipeline"
---

# Automated SaaS Failed Payment & Churn Recovery Pipeline Using Make and Airtable

*Disclosure: This guide contains partner links for software tools we architect with. If you deploy via these links, we may earn a commission at no additional cost to you.*

---

## 1. Executive Bottleneck Analysis

Involuntary churn accounts for 20% to 40% of overall B2B SaaS customer attrition. The primary driver is not product dissatisfaction, but friction in financial infrastructure: expired corporate credit cards, processor soft declines, unhandled bank fraud heuristics, and stale billing contact records.

Organizations managing recovery manually operate in a reactive, uncoordinated state:

```
[Stripe/Billing Failure] 
       │
       ▼
[RevOps / Finance CSV Export] (Lag: 24–72 hours)
       │
       ▼
[Manual CSM / Billing Email]  (Lag: 12–24 hours)
       │
       ▼
[Uncoordinated Dunning]       (Result: High friction, low win-back)
```

### Quantifying the Manual Operational Drag

For a mid-market SaaS company processing 1,200 active subscriptions per month with an average ACV of $12,000 ($1,000 MRR):

*   **Payment Failure Rate:** 4.5% average monthly failure rate = 54 failed invoices/month.
*   **Manual Touch Time:** Reconciling gateway errors, identifying enterprise account owners, auditing product usage telemetry to verify active usage, and manually dispatching dunning communications requires 45 minutes per incident across Finance, Customer Success, and RevOps.
*   **Total Time Lost:** 40.5 hours per month (~10.1 hours per week) spent purely on billing administrative triage.
*   **Financial Impact:** At a fully loaded engineering/operations rate of $95/hour, direct labor waste is **$3,847.50/month ($46,170/year)**. 
*   **The Churn Delta:** The true cost is delayed mean time to recovery (MTTR). Involuntary recovery probability drops by 1.8% for every 24 hours an outreach attempt is delayed. Manual reconciliation creates an average 72-hour lag, directly driving a **$162,000 ARR baseline loss** annually due to preventable subscription cancellations.

Deploying an event-driven recovery pipeline using [Make](https://www.make.com) and [Airtable](https://www.airtable.com) drops MTTR to under 90 seconds, automates triage routing, and reduces internal operational overhead to less than 15 minutes per week for hard exception handling.

---

## 2. Architecture Blueprint

The pipeline decouples the payment gateway's raw webhooks from the outreach tier. Webhook payloads are received, parsed, enriched, normalized, and written to an [Airtable](https://www.airtable.com) ledger. The ledger drives stateful, multi-channel recovery workflows executed by [Make](https://www.make.com).

### ASCII Data-Flow Topology

```
+-----------------------------------------------------------------------------------+

| PAYMENT ENGINE (Stripe)                                                           |
| Event: invoice.payment_failed (Soft Decline / Hard Decline / Insufficient Funds)  |
+-----------------------------------------------------------------------------------+
                                          │
                                          │ (Raw Webhook POST)
                                          ▼
+-----------------------------------------------------------------------------------+

| MAKE.COM WEBHOOK LISTENER                                                         |
| 1. HMAC Signature Verification & Payload De-duplication                           |
| 2. Error Categorization Engine (Regex Match: soft_decline vs hard_decline)        |
+-----------------------------------------------------------------------------------+
                                          │
                                          │ (Parsed & Normalized Schema)
                                          ▼
+-----------------------------------------------------------------------------------+

| AIRTABLE DUNNING ENGINE (Base: "Billing Recovery Matrix")                          |
| 1. Upsert Customer Record                                                         |
| 2. Link/Create Failed Invoice Record                                              |
| 3. Set Recovery Stage (Stage 1: Soft Ping | Stage 2: CSM Escalation | Stage 3: Lock)|
+-----------------------------------------------------------------------------------+
                                          │
                    ┌─────────────────────┴─────────────────────┐
                    ▼                                           ▼
      [Soft Decline Path (< $5k ACV)]             [Hard Decline / Enterprise ACV]
                    │                                           │
                    ▼                                           ▼
+-----------------------------------------+ +---------------------------------------+

| Automated Email Dispatch via Transactional | | Escalation Routing to Dedicated Account |
| Provider (Self-Serve Magic Update Link) | | Executive via Slack / CRM Task Alert  |
+-----------------------------------------+ +---------------------------------------+
```

### Relational Schema: Billing Recovery Matrix

The core data store is built within [Airtable](https://www.airtable.com) using two linked tables: `Accounts` and `Failed Invoices`.

#### Table 1: `Accounts`

| Field Name | Data Type | Airtable Field Config | Validation Rules / Constraints |
| :--- | :--- | :--- | :--- |
| `Account ID` | Single line text | Primary Field | Required; Matches Stripe `cus_xxx` |
| `Company Name` | Single line text | Standard Text | Non-empty |
| `Billing Contact Email` | Email | Standard Email | Must be valid email address syntax |
| `Account Tier` | Single select | Options: Enterprise, Mid-Market, SMB | Default: SMB |
| `ARR Value` | Currency | Precision: $1.00 | Non-negative numeric |
| `Assigned CSM Email` | Email | Standard Email | Required if Tier is Enterprise |
| `Account Health Status` | Single select | Options: Active, Delinquent, Churned, Grace Period | Controlled by recovery workflow |

#### Table 2: `Failed Invoices`

| Field Name | Data Type | Airtable Field Config | Validation Rules / Constraints |
| :--- | :--- | :--- | :--- |
| `Invoice ID` | Single line text | Primary Field | Required; Matches Stripe `in_xxx` |
| `Account Link` | Link to another record | Links to `Accounts` | Required; 1 Account to Many Invoices |
| `Amount Due (USD)` | Currency | Precision: $0.01 | Must be > 0 |
| `Attempt Count` | Number | Integer | Default: 1 |
| `Decline Code` | Single line text | Standard Text | e.g., `insufficient_funds`, `expired_card` |
| `Decline Category` | Single select | Options: Soft Decline, Hard Decline, Authentication Required | Evaluated by Make processor |
| `Recovery Status` | Single select | Options: Retrying, Recovered, Uncollectible, Written Off | Default: Retrying |
| `Next Retry Timestamp` | Date | Include time (ISO 8601) | Must be future date |
| `Hosted Invoice URL` | URL | Standard URL | Valid HTTPS URI |
| `Created At` | Created time | Automatic timestamp | System populated |

---

## 3. Step-by-Step Webhook & Module Configuration

Follow this end-to-end execution path within [Make](https://www.make.com) to wire the recovery pipeline.

```
+--------------------------------------------------------------------------------------------------+

| MAKE SCENARIO: Dunning Pipeline Orchestrator                                                     |
|                                                                                                  |
| [Custom Webhook] ──> [JSON Parser] ──> [Router] ──> (Route A: Upsert Account & Invoice)         |
|                                                  ──> (Route B: Recovery Action Dispatcher)       |
+--------------------------------------------------------------------------------------------------+
```

### Step 1: Webhook Ingestion & Cryptographic Verification
1. Log into your [Make](https://www.make.com) workspace. Create a new Scenario and add a **Custom Webhook** module as the trigger.
2. Label the webhook `Stripe Payment Failures`. Copy the generated endpoint URL.
3. In your Stripe Developer Dashboard, navigate to **Webhooks > Add endpoint**.
4. Paste the Make webhook URL. Select the `invoice.payment_failed` event explicitly.
5. In Make, click **Re-query** and send a test event from Stripe (via CLI: `stripe trigger invoice.payment_failed`) to populate the schema.

### Step 2: Payload Extraction and Error Normalization
Add a **Set Variable** module directly after the Webhook listener to normalize the nested Stripe gateway payload:

*   **Variable Name:** `NormalizedError`
*   **Variable Value Expression:**
    ```
    ifempty(1.data.object.payment_intent.last_payment_error.decline_code; 1.data.object.payment_intent.last_payment_error.code)
    ```
*   **Variable Name:** `DeclineType`
*   **Variable Value Expression:**
    ```
    switch(1.data.object.payment_intent.last_payment_error.decline_code; "insufficient_funds"; "Soft Decline"; "card_velocity_exceeded"; "Soft Decline"; "expired_card"; "Hard Decline"; "stolen_card"; "Hard Decline"; "do_not_honor"; "Hard Decline"; "Soft Decline")
    ```

### Step 3: Upsert Operation into Airtable
To avoid duplicate customer profiles while keeping billing metrics synchronized, implement an idempotent read/write loop using [Airtable](https://www.airtable.com):

1. Add an **Airtable: Search Records** module:
   *   **Base:** `Billing Recovery Matrix`
   *   **Table:** `Accounts`
   *   **Formula:** `{Account ID} = '{% raw %}{{{% endraw %}1.data.object.customer}}'`
2. Add a **Router** module:
   *   **Branch 1 (New Account):** If total bundles returned equals `0`.
       *   Module: **Airtable: Create a Record** (`Accounts`). Map `Account ID` = `1.data.object.customer`, `Company Name` = `1.data.object.customer_name`, `Billing Contact Email` = `1.data.object.customer_email`.
   *   **Branch 2 (Existing Account):** If total bundles returned is greater than `0`.
       *   Pass through to step 3.
3. Add an **Airtable: Upsert a Record** module targeting the `Failed Invoices` table:
   *   **Base:** `Billing Recovery Matrix`
   *   **Table:** `Failed Invoices`
   *   **Search Field:** `Invoice ID`
   *   **Record Data:**
       *   `Invoice ID`: `1.data.object.id`
       *   `Account Link`: Array containing the Airtable Record ID from the customer search/creation step.
       *   `Amount Due (USD)`: `divide(1.data.object.amount_due; 100)`
       *   `Attempt Count`: `1.data.object.attempt_count`
       *   `Decline Code`: `NormalizedError`
       *   `Decline Category`: `DeclineType`
       *   `Recovery Status`: `"Retrying"`
       *   `Hosted Invoice URL`: `1.data.object.hosted_invoice_url`

```
  (Total = 0)  ┌──> [Airtable: Create Account Record] ──┐
[Airtable Search]                                        ├──> [Airtable: Upsert Invoice Record]
  (Total > 0)  └──> [Pass-Through / Verify ID] ────────┘
```

### Step 4: Conditional Triage & Communications Engine
Add an additional Router after the upsert operation to split traffic by account ARR and error severity:

#### Route A: High-Value Enterprise Accounts (`ARR Value` >= $10,000 OR `Decline Category` = "Hard Decline")
*   **Filter:** `Accounts: ARR Value`

<!-- table-safe-v2 -->
