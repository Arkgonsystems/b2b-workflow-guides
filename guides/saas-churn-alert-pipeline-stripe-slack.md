[← Back to All B2B Workflow Guides](../index.html) | [⚡ Get the Turnkey JSON Blueprint Pack ($19)](https://platypusprime.gumroad.com/l/dncmng)

> **FTC Affiliate Disclosure:** We are an independent B2B technical publication and affiliate marketer (not an official vendor partner). This guide contains affiliate links—if you click through and purchase or subscribe, we may earn a commission at no additional cost to you.

> **Quick-Start Stack:** Deploy this workflow with [**Make** (1 Month Free Core Plan / 10,000 Operations)](https://www.make.com/en/register?pc=b2bworkflow) + [**Airtable**](https://airtable.com/invite/r/85BpxD3P), or download our [**Turnkey Make + Airtable JSON Blueprint & Schema Pack**](https://platypusprime.gumroad.com/l/dncmng) to import the complete scenario in 60 seconds.

---
layout: default
title: "Automated SaaS Failed Payment & Churn Recovery Pipeline"
---

# Automated SaaS Failed Payment & Churn Recovery Pipeline Using Make and Airtable

*Disclosure: This guide contains partner links for software tools we architect with. If you deploy via these links, we may earn a commission at no additional cost to you.*

---

## 1. Executive Bottleneck Analysis

Involuntary churn accounts for 20% to 40% of total SaaS customer turnover. When a recurring subscription payment fails due to card expiration, insufficient funds, or fraud heuristics, standard billing engine behaviors (such as native Stripe retries) operate blindly without contextual escalation. 

In a B2B SaaS organization generating $3M to $10M ARR with 1,000 to 3,000 active subscriptions, payment failures average a 4% to 8% monthly rate. This generates roughly 80 to 240 failed invoices each month. Without an automated, cross-functional dunning pipeline, teams manage this workload manually across Finance, Customer Success (CS), and Account Management.

The manual breakdown per week typically includes:
- **Triage & Webhook Auditing:** Manually pulling CSVs of failed charges from Stripe/Braintree and filtering out trial churn: **3.5 hours/week**.
- **Cross-Referencing Customer Context:** Looking up subscription value, account owner, contract status, and CRM history across HubSpot or Salesforce: **4.0 hours/week**.
- **Ad-Hoc Communications:** Manually emailing billing contacts, chasing account owners to intervene, and drafting bespoke notices: **5.5 hours/week**.
- **Reconciliation & Status Verification:** Checking bank feeds against CRM updates, resetting account entitlements, or cancelling service: **3.0 hours/week**.

Total operational overhead reaches **16.0 hours per week**—the equivalent of 0.4 FTE diverted from strategic revenue growth. 

{affiliateAirtable}) for real-time lifecycle tracking, operational overhead drops to **under 0.5 hours per week** (spent strictly on escalated enterprise edge cases). More importantly, time-to-first-dunning-contact falls from 48 hours to under 60 seconds, recovering 35% to 55% of revenue that would otherwise convert into permanent churn.

---

## 2. Architecture Blueprint

The recovery system uses an event-driven architecture. Real-time billing events push via webhooks directly to [Make](https://www.make.com/en/register?pc=b2bworkflow), which validates, normalizes, and logs them into [Airtable](${affiliateAirtable}). Airtable functions as an operational datastore and state machine that drives downstream automated dunning campaigns and human escalation.

```
+-----------------------------------------------------------------------------------+

|                              BILLING GATEWAY (STRIPE)                             |
|        [invoice.payment_failed]                     [invoice.payment_succeeded]   |
+---------------------------+--------------------------------------+----------------+

                            |                                      |
                            v                                      v
+-----------------------------------------------------------------------------------+

|                           MAKE.COM INGESTION ENGINE                               |
|   +------------------------------------+   +------------------------------------+ |
|   | 1. Custom Webhook (HMAC Verif.)    |   | 1. Recovery Webhook Listener        | |
|   | 2. Idempotency Key Validation      |   | 2. Search Existing Failed Record   | |
|   | 3. Customer Tier Segmentation Logic|   | 3. Patch Record -> 'Recovered'     | |
|   +-----------------+------------------+   +-----------------+------------------+ |
+---------------------|----------------------------------------|--------------------+
                      v                                        v
+-----------------------------------------------------------------------------------+

|                        AIRTABLE RECOVERY STATE ENGINE                             |
|  +------------------------------------------------------------------------------+ |
|  | Base: 'Revenue Recovery Orchestrator'                                        | |
|  | - Evaluates Failure Code & Tier                                             | |
|  | - Computes Escalation Path (Grace Period, Retry Cadence, CS Notification)    | |
|  +------------------------------------------------------------------------------+ |
+---------------------+-------------------------------------------------------------+

                      |
        +-------------+-------------+

        |                           |
        v (Tier: SMB / Growth)      v (Tier: Enterprise / Key Account)
+-----------------------+   +-------------------------------------------------------+

| AUTOMATED DUNNING     |   | CONTEXTUAL ESCALATION                                 |
| - Transactional Email |   | - Internal Slack Alert to Dedicated CSM               |
| - Self-Serve Pay Link |   | - CRM Deal Stage Updated to 'At-Risk'                 |
| - In-App Banner Event |   | - Executive Escalation Sequence (14-Day Overdue)       |
+-----------------------+   +-------------------------------------------------------+
```

### Datastore Schema: Failed Invoices Table

The core Airtable table requires strict field validations to prevent race conditions during automated updates:

| Field Name | Data Type | Validation Rules / Configuration | Purpose |
| :--- | :--- | :--- | :--- |
| Record ID | Auto-number | System-generated, indexed primary key | Primary identifier |
| Invoice ID | Single line text | Unique, Regex constraint: ^in_[a-zA-Z0-9]+$ | Stripe invoice identifier |
| Customer ID | Single line text | Required, Regex constraint: ^cus_[a-zA-Z0-9]+$ | Stripe customer identifier |
| Customer Email | Email | Validated RFC 5322 format | Target address for automated dunning |
| Amount Due | Currency | Precision: 2 decimals, minimum value 0.01 | Outstanding balance |
| Customer Tier | Single select | Options: Enterprise, Growth, Self-Serve | Routing condition for recovery paths |
| Failure Code | Single select | Options: insufficient_funds, card_expired, do_not_honor, generic_decline | Determines messaging variant |
| Dunning Stage | Single select | Options: Initial_Failed, Day_3_Retry, Day_7_Escalated, Day_14_Lockout, Recovered, Written_Off | Drives scenario triggers |
| Hosted Invoice URL | URL | Validated HTTPS protocol, domain: stripe.com | Direct payment update link |
| Next Action Due | Formula | DATETIME_DIFF and DATEADD computation | Dictates execution window |
| CS Account Owner | Collaborator | Must map to internal workspace user | Notification target for Tier 1 accounts |

---

## 3. Step-by-Step Webhook & Module Configuration

Follow this sequential configuration inside [Make](https://www.make.com/en/register?pc=b2bworkflow) to establish the primary recovery pipeline.

### Step 1: Webhook Ingestion & HMAC Verification
1. Add a **Custom Webhook** module. Label it `Stripe Ingestion: Failed Payments`.
2. Copy the generated URL and register it in Stripe under **Developers > Webhooks**. Select the event `invoice.payment_failed`.
3. Add a **Crypto: Create a HMAC** module immediately following the webhook to verify the `stripe-signature` header against your endpoint secret to mitigate replay and spoofing attacks.

```json
{
  "event": "invoice.payment_failed",
  "data": {
    "object": {
      "id": "in_1N4cXYZExample998",
      "customer": "cus_M123456789",
      "customer_email": "ops@targetclient.com",
      "amount_due": 450000,
      "currency": "usd",
      "hosted_invoice_url": "https://invoice.stripe.com/i/acct_123/test_xyz",
      "attempt_count": 1,
      "charge": {
        "failure_code": "card_expired"
      }
    }
  }
}
```

### Step 2: Idempotency & Duplicate Check
1. Add an [Airtable](${affiliateAirtable}) **Search Records** module.
2. Select your Base: `Revenue Recovery Orchestrator` and Table: `Failed Invoices`.
3. Set the Filter Formula to:
   ```text
   {Invoice ID} = '{% raw %}{{{% endraw %}1.data.object.id}}'
   ```
4. Configure an execution limit of `1`.

### Step 3: Conditional Branching (Router)
Place a **Router** module directly after the search step:

* **Route A (New Record Insertion):**
  * **Filter Condition:** Set filter to `Total number of bundles` from Search Records equal to `0`.
  * **Module:** Airtable **Create a Record**.
  * Map incoming webhook attributes:
 `1.data.object.id`
 `1.data.object.customer`
 `1.data.object.customer_email`
 `1.data.object.amount_due / 100` *(convert cents to standard currency)*
 `1.data.object.hosted_invoice_url`
 `1.data.object.charge.failure_code`
 `Initial_Failed`

* **Route B (Existing Record Update):**
  * **Filter Condition:** Set filter to `Total number of bundles` greater than `0`.
  * **Module:** Airtable **Update a Record**.
  * Target Record ID from the initial search step. Increment the `Attempt Count` field and update the `Failure Code` if it has changed from the initial run.

### Step 4: Segmentation & Dispatch Router
Insert a second Router conditioned on the `Customer Tier` field:

* **Branch 1 (Enterprise Accounts - Amount $\ge$ $2,000):**
  * Add a **Slack: Create a Message** module targeting your private `#cs-revenue-alerts` channel.
  * Message body:
    ```text
    :alert: *Enterprise Payment Failure*: <{% raw %}{{{% endraw %}1.data.object.customer_email}}>
{% raw %}{{{% endraw %}1.data.object.amount_due / 100}}
    Reason: {% raw %}{{{% endraw %}1.data.object.charge.failure_code}}
    Action Required: Direct CSM outreach before next automated retry.
    Airtable Reference: https://airtable.com/appXXXXXXXX/tblYYYYYYYY
    ```
* **Branch 2 (Self-Serve / Growth Accounts - Amount < $2,000):**
  * Add an Email/Transactional Delivery module (e.g., SendGrid, Postmark, or native Make SMTP).
  * Set dynamic template parameters passing the `hosted_invoice_url` to a prominent "Update Payment Method" button.

### Step 5: Webhook Listener for Successful Recovery
Create a distinct, secondary scenario within [Make](https://www.make.com/en/register?pc=b2bworkflow):
1. **Custom Webhook** listening exclusively for Stripe's `invoice.payment_succeeded` event.
2. **Airtable Search Records** by `{Invoice ID} = '{% raw %}{{{% endraw %}1.data.object.id}}'`.
3. **Airtable Update Record**:
 `Recovered`
 `{% raw %}{{{% endraw %}now}}`
4. Send an internal notification closing the recovery loop and halting further dunning actions.

---

## 4. Data Transformation & Validation Formula

The most fragile component of SaaS payment recovery is calculating dynamic escalation cadences based on failure classifications. Hard declines (`stolen_card`, `do_not_honor`) require immediate customer action, whereas soft declines (`insufficient_funds`) should be delayed to match common payroll windows.

Paste the following formula into the `Next Action Due` field in [Airtable](${affiliateAirtable}):

```text
IF(
  {Dunning Stage} = "Recovered",
  BLANK(),
  IF(
    OR({Failure Code} = "card_expired", {Failure Code} = "do_not_honor"),
    /* Hard decline: Short cycle (24-hour escalation) */
    DATEADD(CREATED_TIME(), 1, "days"),
    /* Soft decline: Algorithmic staging based on current dunning tier */
    SWITCH(
      {Dunning Stage},
      "Initial_Failed", DATEADD(CREATED_TIME(), 3, "days"),
      "Day_3_Retry", DATEADD(CREATED_TIME(), 7, "days"),
      "Day_7_Escalated", DATEADD(CREATED_TIME(), 14, "days"),
      "Day_14_Lockout", DATEADD(CREATED_TIME(), 21, "days"),
      BLANK()
    )
  )
)
```

### Formula Logic Explained:
1. **Terminal Condition Check:** If `{Dunning Stage}` is marked as `Recovered`, the formula evaluates to blank, terminating active scheduling views.
2. **Failure Categorization:** The `OR()` logical block inspects the specific decline code passed down by Stripe. For non-retryable events (`card_expired`, `do_not_honor`), it overrides default delays and forces the next escalation event to trigger within **24 hours** (`DATEADD(..., 1, "days")`).
3. **Algorithmic Dunning Tiering:** For non-critical failures, the `SWITCH()` statement dynamically increases the runway between retries. This spaces communications out to Day 3, Day 7, Day 14, and Day 21, matching standard B2B accounts payable processing schedules without spamming finance teams.

---

## 5. Turnkey Implementation Blueprint & Deployment

To launch this architecture with minimal configuration overhead, use the turnkey deployment checklist:

### Pre-Deployment Checklist
1. **Base Instantiation:** Create your base within [Airtable](${affiliateAirtable}). Ensure column API identifiers mirror the schema table from Section 2.
2. **Environment Authorization:** Set up your webhook connections using [Make](https://www.make.com/en/register?pc=b2bworkflow). Authenticate your production Stripe instance with restricted read/write permissions scoped to Invoices, Charges, and Customers.
3. **Error Directives:** Implement a Make `Commit` and `Break` directive on the primary Webhook module. Set it to store failed executions for 30 days to protect against webhook drops during downstream API maintenance.
4. **End-to-End Test Execution:** Trigger a synthetic `invoice.payment_failed` event using Stripe CLI:
   ```bash
   stripe trigger invoice.payment_failed
   ```
   Verify that the record instantly appears in Airtable with its correctly computed `Next Action Due` date.

### Deployment Resources
* Accelerate deployment by using pre-configured blueprints: [Download the Complete Turnkey Blueprint](https://loudplatypus6-blip.github.io/b2b-workflow-guides/blueprints.html).
* Need execution capacity for this pipeline? [Claim 1 Month of Make.com Core Free (10,000 Operations Included)](https://www.make.com/en/register?pc=b2bworkflow).
* Sign up for [Make](https://www.make.com/en/register?pc=b2bworkflow) and configure your database on [Airtable](${affiliateAirtable}) to deploy your automated recovery engine today.

---

## Turnkey Implementation Blueprint & Partner Promo

* **Get the Complete 70-Workflow JSON & Schema Pack ($19):** [Download the Complete Turnkey Blueprint Pack on Gumroad](https://platypusprime.gumroad.com/l/dncmng)
* **Free Starter JSON Blueprint:** [View the Starter JSON Scenario Hub](https://loudplatypus6-blip.github.io/b2b-workflow-guides/blueprints.html)
* **Activate 10,000 Free Make.com Operations:** [Claim 1 Month of Make.com Core Free](https://www.make.com/en/register?pc=b2bworkflow)
* **Free Relational Database Workspace:** [Launch Airtable Free](https://airtable.com/invite/r/85BpxD3P)

<!-- table-safe-v8 -->

<!-- table-safe-v6 -->

---

## Deploy This Workflow in 60 Seconds

* **Start Building Free:** [Launch Make (10,000 Free Operations)](https://www.make.com/en/register?pc=b2bworkflow) and connect your [Airtable workspace](https://airtable.com/invite/r/85BpxD3P).
* **Skip the Manual Setup:** [Download the Turnkey JSON Blueprint & Schema Pack ($19)](https://platypusprime.gumroad.com/l/dncmng) or browse all bundles on the [Blueprint Hub](../blueprints.html).
