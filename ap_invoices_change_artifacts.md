# AP Invoices Pipeline — Change Artifacts

Handoff document for the PRD-driven update to `COCO_WORKSHOP.PUBLIC.SILVER_AP_INVOICES`
and `COCO_WORKSHOP.PUBLIC.GOLD_AP_INVOICES`.

---

## 1. PRD Input Files

The business requirements that drove this change. All three are in the repo root.

| File | Contents |
|------|----------|
| `sample_business_requirements_source_onboarding.csv` | New source system requests (Baan IV, Workday FM) with contacts, delivery method, go-live status, blockers |
| `sample_business_requirements_column_mapping.csv` | Field-level mapping from each source column to the Silver target schema, including types, transforms, and open questions |
| `sample_business_requirements_business_rules.csv` | Business rules BR-001 through BR-010 with decisions, owners, and layer assignments |

---

## 2. Implementation Plan

| File | Contents |
|------|----------|
| `silver_ap_invoices_plan.md` | The structured plan generated before any SQL was written. Contains: summary of changes, source-to-Silver mapping table, 6 open questions with owners, DDL delta, and 8 validation queries. Review this first to understand the design rationale. |

---

## 3. PRD Evaluator Skill (Reusable)

A project-local Cortex Code skill that turns PRD-style files into DT implementation plans.
Located at `.cortex/skills/prd-to-dt-plan/`.

| File | Purpose |
|------|---------|
| `.cortex/skills/prd-to-dt-plan/SKILL.md` | Skill definition — workflow, inputs, output format, heuristics for surfacing open questions |
| `.cortex/skills/prd-to-dt-plan/scripts/parse_prd.py` | Python script to parse XLSX files into JSON (CSV files are read directly by the skill) |
| `.cortex/skills/prd-to-dt-plan/pyproject.toml` | Dependency manifest (`openpyxl>=3.1`) |
| `.cortex/skills/prd-to-dt-plan/automation_prompt.md` | The prompt used by the daily automation (reads from `@COCO_WORKSHOP.PUBLIC.PRD_FILES`) |

**To reuse:** invoke `/prd-to-dt-plan` in a Cortex Code session (available after session restart), or follow the workflow in `SKILL.md` manually.

---

## 4. Snowflake Objects Created

All objects live in `COCO_WORKSHOP.PUBLIC`.

### Bronze (source tables)

| Object | Type | Description |
|--------|------|-------------|
| `BRONZE_BAAN_AP_INVOICES` | TABLE | 6 sample rows. Includes a duplicate pair (BAN-001/BAN-004 share `BAN_INVOICE_REF`) and a credit memo (BAN-006) |
| `BRONZE_WORKDAY_AP_INVOICES` | TABLE | 5 sample rows. Mixed statuses (Approved, In Review), multi-currency (USD, GBP, EUR), multi-tenant (WD-T1, WD-T2) |

### Silver

| Object | Type | Description |
|--------|------|-------------|
| `SILVER_AP_INVOICES` | DYNAMIC TABLE | 16-column common schema. `TARGET_LAG = DOWNSTREAM`, warehouse `COCO_WORKSHOP_WH`. Applies: Baan dedup (BR-003), status normalization (BR-001), SOURCE_SYSTEM literal (BR-007), column drops (BR-008) |

### Gold

| Object | Type | Description |
|--------|------|-------------|
| `GOLD_AP_INVOICES` | DYNAMIC TABLE | 18 columns (Silver + `INVOICE_AMOUNT_USD` + `HIGH_VALUE_FLAG`). `TARGET_LAG = 2 HOURS`. Applies: FX conversion (BR-002), payment terms normalization (BR-005), high-value flag (BR-004) |
| `TREASURY_FX_RATES` | TABLE | Reference table: daily FX rates (currency → USD). Keyed on `(CURRENCY_CODE, RATE_DATE)`. Loaded with sample rates for June 1–6, 2025 |
| `PAYMENT_TERMS_MAP` | TABLE | Reference table: maps source-specific terms (N30, Net 30, NET30) to standard format (NET30/NET60). 6 rows |

### Automation

| Object | Location | Description |
|--------|----------|-------------|
| `COCO_ROUTINE_PRD_TO_DT_PLAN` | `USER$SWALOGU1329.PUBLIC` | AGENT TASK, daily at 9:00 AM UTC. Reads CSVs from `@COCO_WORKSHOP.PUBLIC.PRD_FILES`, produces a 5-section plan in the fire's conversation transcript |
| `@COCO_WORKSHOP.PUBLIC.PRD_FILES` | STAGE | Landing zone for PRD files. Currently contains the 3 sample CSVs |

---

## 5. Validation Queries

Copy-paste these to verify the pipeline is healthy. All should pass against the sample data.

### Silver

```sql
-- Row counts (expect BAAN:5, WORKDAY:5)
SELECT SOURCE_SYSTEM, COUNT(*) AS silver_rows
FROM COCO_WORKSHOP.PUBLIC.SILVER_AP_INVOICES
GROUP BY SOURCE_SYSTEM;

-- Dedup check (expect 0 rows)
SELECT INVOICE_NUMBER, COUNT(*) AS cnt
FROM COCO_WORKSHOP.PUBLIC.SILVER_AP_INVOICES
WHERE SOURCE_SYSTEM = 'BAAN'
GROUP BY INVOICE_NUMBER HAVING cnt > 1;

-- Status normalization (expect only APPROVED, PENDING)
SELECT SOURCE_SYSTEM, APPROVAL_STATUS, COUNT(*) AS cnt
FROM COCO_WORKSHOP.PUBLIC.SILVER_AP_INVOICES
GROUP BY SOURCE_SYSTEM, APPROVAL_STATUS;

-- NULL checks on required fields (expect all zeros)
SELECT SOURCE_SYSTEM,
  COUNT_IF(INVOICE_ID IS NULL) AS null_id,
  COUNT_IF(INVOICE_NUMBER IS NULL) AS null_num,
  COUNT_IF(VENDOR_ID IS NULL) AS null_vendor,
  COUNT_IF(INVOICE_AMOUNT IS NULL) AS null_amt,
  COUNT_IF(APPROVAL_STATUS IS NULL) AS null_status
FROM COCO_WORKSHOP.PUBLIC.SILVER_AP_INVOICES
GROUP BY SOURCE_SYSTEM;

-- Dropped columns (expect 0 rows)
SELECT COLUMN_NAME FROM COCO_WORKSHOP.INFORMATION_SCHEMA.COLUMNS
WHERE TABLE_SCHEMA='PUBLIC' AND TABLE_NAME='SILVER_AP_INVOICES'
  AND COLUMN_NAME IN ('BAN_COMPANY','WD_TENANT_ID');
```

### Gold

```sql
-- Row parity (expect both = 10)
SELECT
  (SELECT COUNT(*) FROM COCO_WORKSHOP.PUBLIC.SILVER_AP_INVOICES) AS silver,
  (SELECT COUNT(*) FROM COCO_WORKSHOP.PUBLIC.GOLD_AP_INVOICES) AS gold;

-- FX conversion completeness (expect 0 nulls)
SELECT SOURCE_SYSTEM, COUNT_IF(INVOICE_AMOUNT_USD IS NULL) AS null_usd
FROM COCO_WORKSHOP.PUBLIC.GOLD_AP_INVOICES
GROUP BY SOURCE_SYSTEM;

-- Payment terms normalized (expect only NET30, NET60)
SELECT PAYMENT_TERMS, COUNT(*) AS cnt
FROM COCO_WORKSHOP.PUBLIC.GOLD_AP_INVOICES
GROUP BY PAYMENT_TERMS;

-- High-value flag (expect 2 rows: WD-005 and BAN-005)
SELECT INVOICE_ID, SOURCE_SYSTEM, INVOICE_AMOUNT_USD
FROM COCO_WORKSHOP.PUBLIC.GOLD_AP_INVOICES
WHERE HIGH_VALUE_FLAG = TRUE;
```

---

## 6. Open Questions (Unresolved)

These were surfaced during analysis and remain open. Resolve before promoting to production.

| # | Question | Owner |
|---|----------|-------|
| 1 | Payment terms: normalize at Silver or Gold? (Currently Gold.) | Sarah Chen + David Kim |
| 2 | Baan cost center BC-XX vs BC-XXX: normalize or pass through? | Karen van der Berg |
| 3 | BR-004 high-value flag: should large credit memos also trigger review? | Tom Walsh |
| 4 | Workday DPA-2025-0041: legal sign-off still pending | Jennifer Okafor + Legal |
| 5 | SAP and Oracle column mappings needed for full four-source UNION ALL | David Kim |
| 6 | INVOICE_ID uniqueness: composite key (INVOICE_ID, SOURCE_SYSTEM)? | Engineering |
| 7 | FX rate gaps on weekends/holidays: carry-forward or fail? | Treasury + Engineering |
| 8 | Gold target lag 2h is provisional — review after 30 days | Engineering |

---

## 7. Pipeline Diagram

```
BRONZE_BAAN_AP_INVOICES ──┐
                          ├──▶ SILVER_AP_INVOICES ──▶ GOLD_AP_INVOICES
BRONZE_WORKDAY_AP_INVOICES┘    (TARGET_LAG=DOWNSTREAM)   (TARGET_LAG=2h)
                                                               │
                           TREASURY_FX_RATES ──────────────────┤
                           PAYMENT_TERMS_MAP ──────────────────┘
```
