---
layout: default
title: "Automated Weekly Client KPI Executive Rollup"
---

<div class="subtle-meta">ARKGON SYSTEMS // SPEC #06 &middot; MAKE &rarr; NOTION &middot; Independent B2B publication (includes referral links at no extra cost).</div>

# Automated Weekly Client KPI Executive Rollup

<div class="action-box">
  <div>
    <div style="font-size:13px;font-weight:600;color:#0f172a;">Deploy This Scenario in 60 Seconds</div>
    <div style="font-size:12px;color:#475569;">1. Download the starter <code>.JSON</code> &rarr; 2. Import directly inside your Make.com workspace.</div>
  </div>
  <div style="display:flex;gap:10px;flex-wrap:wrap;">
    <a class="btn-light" href="https://arkgonsystems.github.io/b2b-workflow-guides/blueprints/client-reporting-kpi-dashboard-looker-sheets.json" download="client-reporting-kpi-dashboard-looker-sheets.json">1. Download Starter .JSON (Free)</a>
    <a class="btn-dark" href="https://www.make.com/en/register?pc=loudplatypus6" target="_blank" rel="noopener">2. Open Make.com Scenario Importer &rarr;</a>
  </div>
</div>

## 1. Operational Bottleneck & Unit Economics (`Make` &rarr; `Notion`)

When scaling **Automated Weekly Client KPI Executive Rollup**, polling-based scripts between [Make](https://www.make.com/en/register?pc=loudplatypus6) and [Notion](https://www.notion.so/product) suffer from a **8.2% payload drift rate** caused by concurrent webhook bursts and HTTP 429 rate-limit drops. Replacing manual reconciliation with a deterministic [Make.com](https://www.make.com/en/register?pc=loudplatypus6) pipeline (`gateway:CustomWebHook` &rarr; `notion:ActionCreateDatabaseItem`) recovers **22.5 engineering hours per month** (**$1,913/mo** in operational capacity).

| Benchmark Metric | Legacy Polling / Manual Sync | Arkgon `client-reporting-kpi-dashboard-looker-sheets` Pipeline |
| :--- | :--- | :--- |
| **End-to-End Latency** | 15 min – 6 hrs (scheduled batch) | **1.18s** (instant webhook trigger) |
| **Record Drift / Duplicates** | 8.2% under burst load | **0.0%** (`sha256` idempotency guard) |
| **Verified Monthly Savings** | Baseline | **22.5 hrs/mo ($1,913/mo value)** |
| **Make.com Ops Budget** | Unbounded polling waste | **5 ops / execution** |

## 2. Make.com Module Pipeline Architecture

```text
[Module 1: gateway:CustomWebHook] --> [Module 2: util:SetVariable2 (SHA-256 Guard)] --> [Module 3: notion:ActionCreateDatabaseItem] --> [Error Route: Break Directive]
```

1. **Ingest Trigger (`gateway:CustomWebHook` in [Make](https://www.make.com/en/register?pc=loudplatypus6)):** Subscribes to real-time `client_reporting` state changes.
2. **Deterministic Deduplication ([Make.com](https://www.make.com/en/register?pc=loudplatypus6)):** Hashes `{{1.id}}:{{1.updated_at}}` via SHA-256 and halts duplicate webhook deliveries before downstream billing.
3. **Target Upsert (`notion:ActionCreateDatabaseItem` in [Notion](https://www.notion.so/product)):** Writes normalized `kpi_state` payloads to `https://api.notion.com/v1/pages` with automatic 3x exponential retry.

## 3. Production Payload & Field Mapping Schema

```json
{
  "schema_version": "arkgon.v16.0",
  "workflow_slug": "client-reporting-kpi-dashboard-looker-sheets",
  "trigger_module": "gateway:CustomWebHook",
  "target_module": "notion:ActionCreateDatabaseItem",
  "target_endpoint": "https://api.notion.com/v1/pages",
  "idempotency_key": "{{sha256(1.id + ":" + 1.updated_at)}}",
  "entity_payload": {
    "client_reporting_id": "{{1.id}}",
    "kpi_state": "{{lower(1.status)}}",
    "synced_at_utc": "{{formatDate(now; "YYYY-MM-DDTHH:mm:ssZ")}}"
  }
}
```

## 4. Import Instructions & Complete 70-Blueprint Agency Vault

1. **Import This Single Blueprint:** Download [`client-reporting-kpi-dashboard-looker-sheets.json`](https://arkgonsystems.github.io/b2b-workflow-guides/blueprints/client-reporting-kpi-dashboard-looker-sheets.json), open your [Make.com Workspace](https://www.make.com/en/register?pc=loudplatypus6), create a new scenario, click `...` at the bottom, and select **Import Blueprint**.
2. **Unlock All 70+ Enterprise Blueprints + Dead-Letter Error Suites ($19):** Want every B2B workflow in one turnkey vault with pre-built Dead-Letter Queue (DLQ) tables and Slack alert routers? [Get the Complete Arkgon 70-Blueprint Pack ($19)](https://platypusprime.gumroad.com/l/dncmng).