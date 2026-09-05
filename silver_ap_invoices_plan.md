# Dynamic Table Implementation Plan
## Target: `SILVER_AP_INVOICES`
### Generated from:
- `sample_business_requirements_column_mapping.csv`
- `sample_business_requirements_source_onboarding.csv`
- `sample_business_requirements_business_rules.csv`

### Status: Greenfield — the target Dynamic Table does not exist yet.

---

## 1. Summary of Requested Changes

Two new source systems are being onboarded into a new Silver-layer Dynamic Table for AP invoices:

| Source | Platform | Region | Delivery | Cadence | Status |
|--------|----------|--------|----------|---------|--------|
| **Baan IV** | Infor Baan (on-prem, Rotterdam DC) | EMEA (NL + UK) | CSV → S3 landing zone | Nightly ~02:00 UTC | Approved — mapping in progress |
| **Workday FM** | Workday (cloud) | Americas (US + CA) | Workday connector (INC-44291) | Hourly | Pending — DPA-2025-0041 in legal review |

The PRD also references two **existing** sources (SAP and Oracle) in the business rules, indicating the Silver DT is expected to UNION ALL across four systems total. However, column mappings for SAP and Oracle were **not provided** in this PRD — only Baan and Workday mappings are defined. This plan covers only the two new sources.

**Key business rules confirmed for Silver:**
- Status normalization via CASE (BR-001)
- No currency conversion at Silver (BR-002)
- Baan deduplication on INVOICE_NUMBER by latest CREATED_AT (BR-003)
- SOURCE_SYSTEM literal on every row (BR-007)
- Drop system-specific columns at the Bronze-to-Silver boundary (BR-008)
- GL codes and cost centers stored as-is — no cross-mapping (BR-006)

**Deferred to Gold / Phase 2:**
- Currency conversion using Treasury FX rate table (BR-002)
- GL account unified chart of accounts mapping (BR-006)
- Payment terms normalization (BR-005 — pending decision)
- Historical time-series table (BR-010)

---

## 2. Source-to-Silver Mapping Summary

### Unified Silver Schema (16 columns)

| # | Silver Column | Type | Baan Source | Workday Source | Transform | Notes |
|---|---------------|------|-------------|----------------|-----------|-------|
| 1 | `INVOICE_ID` | VARCHAR(20) | `BAN_INVOICE_ID` | `WD_INVOICE_ID` | Direct | PK in source. Format differs: BAN-NNN vs WD-NNN |
| 2 | `INVOICE_NUMBER` | VARCHAR(30) | `BAN_INVOICE_REF` | `WD_INVOICE_NUM` | Direct | Business-facing number. Baan dedup key (BR-003) |
| 3 | `VENDOR_ID` | VARCHAR(15) | `BAN_VENDOR_CODE` | `WD_SUPPLIER_ID` | Direct | Workday calls them "suppliers" — standardized to vendor |
| 4 | `VENDOR_NAME` | VARCHAR(100) | `BAN_VENDOR_DESC` | `WD_SUPPLIER_NAME` | Direct | Baan may have UTF-8 special chars (ü, é) |
| 5 | `INVOICE_DATE` | DATE | `BAN_INV_DATE` | `WD_INVOICE_DATE` | Direct | |
| 6 | `DUE_DATE` | DATE | `BAN_PAY_DATE` | `WD_DUE_DATE` | Direct | |
| 7 | `INVOICE_AMOUNT` | NUMBER(18,2) | `BAN_AMOUNT` | `WD_AMOUNT` | Direct | Always positive. Baan credit memos are separate negative records |
| 8 | `CURRENCY_CODE` | VARCHAR(3) | `BAN_CURR` | `WD_CURRENCY` | Direct | Baan: EUR, GBP. Workday: USD (~70%), GBP, EUR |
| 9 | `PAYMENT_TERMS` | VARCHAR(20) | `BAN_PAY_TERMS` | `WD_PAY_TERMS` | **Pass-through (for now)** | Baan: N30/N60. Workday: Net 30/Net 60. Normalization deferred — see Open Question #1 |
| 10 | `PO_NUMBER` | VARCHAR(20) | `BAN_PO_REF` | `WD_PO_NUMBER` | Direct | Nullable. Baan ~85% coverage, Workday ~90% |
| 11 | `LINE_DESCRIPTION` | VARCHAR(200) | `BAN_LINE_DESC` | `WD_MEMO` | Direct | Baan: may contain Dutch text. Workday: "Memo" field |
| 12 | `GL_ACCOUNT` | VARCHAR(20) | `BAN_GL_CODE` | `WD_LEDGER_ACCOUNT` | Direct | Baan: GL-NNN. Workday: LA-NNNN. No cross-mapping at Silver (BR-006) |
| 13 | `COST_CENTER` | VARCHAR(20) | `BAN_COST_CTR` | `WD_COST_CENTER` | Direct | Baan has mixed format (BC-XX / BC-XXX). Workday: WCC-XXXX |
| 14 | `APPROVAL_STATUS` | VARCHAR(20) | `BAN_STATUS` | `WD_APPROVAL_STATUS` | **CASE map (BR-001)** | Baan: POSTED→APPROVED. Workday: Approved→APPROVED, In Review→PENDING |
| 15 | `CREATED_AT` | TIMESTAMP_NTZ | `BAN_CREATED` | `WD_CREATED_DATE` | Direct | Both UTC |
| 16 | `SOURCE_SYSTEM` | VARCHAR(10) | — | — | **Literal** | 'BAAN' or 'WORKDAY' (BR-007) |

### Columns Dropped at Bronze-to-Silver Boundary (BR-008)

| Source | Dropped Column | Reason |
|--------|---------------|--------|
| Baan | `BAN_COMPANY` | Company code — not needed in Silver |
| Workday | `WD_TENANT_ID` | Tenant identifier (WD-T1=US, WD-T2=London) — not needed in Silver |

---

## 3. Open Questions and Assumptions

### Open Questions (Must Resolve Before Implementation)

| # | Question | Why It Matters | Suggested Owner |
|---|----------|----------------|-----------------|
| 1 | **Payment terms normalization: Silver or Gold?** Baan sends N30/N60, Workday sends "Net 30"/"Net 60", SAP sends NET30/NET60. BR-005 is marked "NEEDS DECISION." | If Silver normalizes, the CASE statement goes here. If Gold normalizes, Silver passes through as-is and downstream consumers see inconsistent formats until Gold is built. | Sarah Chen + David Kim |
| 2 | **Baan cost center format change (Jan 2025): normalize or pass through?** Old format BC-XX, new format BC-XXX. Column mapping says "direct map" with no transform, but no business rule explicitly addresses normalization. | Queries filtering on COST_CENTER will need to handle both patterns. If this is intentional (both are valid), document it. If BC-XX should be migrated to BC-XXX, add a transform rule. | Karen van der Berg + Sarah Chen |
| 3 | **BR-004 threshold vs BR-002 no-FX contradiction.** BR-004 flags invoices > $500K "USD equivalent" for manual review via DMF. BR-002 says no currency conversion at Silver. How does the DMF evaluate USD equivalent for EUR/GBP invoices without an FX rate? | The DMF either needs access to the Treasury FX table (which is out of scope) or the threshold must be expressed in each local currency, or the check must be deferred to Gold. | Tom Walsh (Finance Controller) |
| 4 | **Workday DPA-2025-0041 not signed.** Legal review of the cross-entity data sharing agreement is pending. Expected by end of June 2025. | Workday branch cannot go live until this clears. If delayed, ship Baan-only first and add Workday later. | Jennifer Okafor + Legal |
| 5 | **SAP and Oracle column mappings not provided.** The business rules reference four source systems, but this PRD only maps Baan and Workday. Are SAP and Oracle already in a separate Bronze-to-Silver pipeline, or are they expected in this same DT? | If all four go in one DT, we need the SAP and Oracle mappings to build the full UNION ALL. If Baan/Workday are additive branches to an existing DT, we need the existing DDL. | David Kim |
| 6 | **INVOICE_ID uniqueness across sources.** Baan uses BAN-NNN, Workday uses WD-NNN, and the column mapping shows both mapping to INVOICE_ID. Is INVOICE_ID unique within a source or globally? Does the PK become (INVOICE_ID, SOURCE_SYSTEM)? | Affects dedup logic, join patterns, and downstream key relationships. | Engineering / Sarah Chen |

### Assumptions Made (Implicit in the Plan)

| # | Assumption | Basis |
|---|-----------|-------|
| A1 | Silver DT uses `TARGET_LAG = DOWNSTREAM` per BR-009 | Business rules doc |
| A2 | Baan dedup (BR-003) applies only to Baan, not Workday | BR-003 specifically references Baan's nightly extract issue |
| A3 | APPROVAL_STATUS values not matching the CASE statement will pass through as-is (no default to PENDING or error) | Not specified — needs confirmation |
| A4 | Credit memos from Baan (negative INVOICE_AMOUNT) are valid Silver rows, not filtered | Column mapping says "Credit memos come as separate records with negative amounts" with no filter instruction |
| A5 | This DT covers Baan + Workday only for now; SAP + Oracle branches will be added when mappings are provided | Based on what the PRD contains |

---

## 4. DDL Delta Plan for SILVER_AP_INVOICES

Since this is a greenfield build, the full DDL is shown below. Each UNION ALL branch includes the status normalization CASE (BR-001), the SOURCE_SYSTEM literal (BR-007), and the Baan dedup QUALIFY (BR-003).

```sql
CREATE OR REPLACE DYNAMIC TABLE SILVER_AP_INVOICES
  TARGET_LAG = DOWNSTREAM
  WAREHOUSE = -- << specify warehouse >>
AS
WITH baan_deduped AS (
    SELECT
        BAN_INVOICE_ID        AS INVOICE_ID,
        BAN_INVOICE_REF       AS INVOICE_NUMBER,
        BAN_VENDOR_CODE       AS VENDOR_ID,
        BAN_VENDOR_DESC       AS VENDOR_NAME,
        BAN_INV_DATE          AS INVOICE_DATE,
        BAN_PAY_DATE          AS DUE_DATE,
        BAN_AMOUNT            AS INVOICE_AMOUNT,
        BAN_CURR              AS CURRENCY_CODE,
        BAN_PAY_TERMS         AS PAYMENT_TERMS,
        BAN_PO_REF            AS PO_NUMBER,
        BAN_LINE_DESC         AS LINE_DESCRIPTION,
        BAN_GL_CODE           AS GL_ACCOUNT,
        BAN_COST_CTR          AS COST_CENTER,
        CASE BAN_STATUS
            WHEN 'POSTED'   THEN 'APPROVED'
            WHEN 'APPROVED' THEN 'APPROVED'
            WHEN 'PENDING'  THEN 'PENDING'
            ELSE BAN_STATUS                     -- pass-through unknown (Assumption A3)
        END                   AS APPROVAL_STATUS,
        BAN_CREATED           AS CREATED_AT,
        'BAAN'                AS SOURCE_SYSTEM
        -- DROPPED: BAN_COMPANY (BR-008)
    FROM BRONZE_BAAN_AP_INVOICES                -- << confirm Bronze table name >>
    QUALIFY ROW_NUMBER() OVER (
        PARTITION BY BAN_INVOICE_REF
        ORDER BY BAN_CREATED DESC
    ) = 1                                       -- BR-003: Baan dedup
),

workday AS (
    SELECT
        WD_INVOICE_ID         AS INVOICE_ID,
        WD_INVOICE_NUM        AS INVOICE_NUMBER,
        WD_SUPPLIER_ID        AS VENDOR_ID,
        WD_SUPPLIER_NAME      AS VENDOR_NAME,
        WD_INVOICE_DATE       AS INVOICE_DATE,
        WD_DUE_DATE           AS DUE_DATE,
        WD_AMOUNT             AS INVOICE_AMOUNT,
        WD_CURRENCY           AS CURRENCY_CODE,
        WD_PAY_TERMS          AS PAYMENT_TERMS,
        WD_PO_NUMBER          AS PO_NUMBER,
        WD_MEMO               AS LINE_DESCRIPTION,
        WD_LEDGER_ACCOUNT     AS GL_ACCOUNT,
        WD_COST_CENTER        AS COST_CENTER,
        CASE WD_APPROVAL_STATUS
            WHEN 'Approved'  THEN 'APPROVED'
            WHEN 'In Review' THEN 'PENDING'
            ELSE WD_APPROVAL_STATUS             -- pass-through unknown (Assumption A3)
        END                   AS APPROVAL_STATUS,
        WD_CREATED_DATE       AS CREATED_AT,
        'WORKDAY'             AS SOURCE_SYSTEM
        -- DROPPED: WD_TENANT_ID (BR-008)
    FROM BRONZE_WORKDAY_AP_INVOICES             -- << confirm Bronze table name >>
)

SELECT * FROM baan_deduped
UNION ALL
SELECT * FROM workday;
```

**Placeholders to resolve before executing:**
- `WAREHOUSE` — which warehouse runs this DT refresh?
- `BRONZE_BAAN_AP_INVOICES` — confirm actual Bronze table FQN
- `BRONZE_WORKDAY_AP_INVOICES` — confirm actual Bronze table FQN
- SAP and Oracle branches — add when mappings are provided (Open Question #5)

---

## 5. Validation Queries to Run After Implementation

### 5a. Row Counts by Source

```sql
-- Verify both sources are flowing and Baan dedup reduced row count
SELECT
    SOURCE_SYSTEM,
    COUNT(*)             AS silver_rows
FROM SILVER_AP_INVOICES
GROUP BY SOURCE_SYSTEM;
```

Compare Baan silver_rows against raw Bronze count — expect fewer rows due to dedup.

### 5b. Baan Dedup Effectiveness

```sql
-- Confirm no duplicate INVOICE_NUMBERs remain for Baan
SELECT
    INVOICE_NUMBER,
    COUNT(*) AS cnt
FROM SILVER_AP_INVOICES
WHERE SOURCE_SYSTEM = 'BAAN'
GROUP BY INVOICE_NUMBER
HAVING cnt > 1;
```

**Expected result:** 0 rows.

### 5c. Status Normalization Check

```sql
-- All APPROVAL_STATUS values should be APPROVED or PENDING
-- Any other value means an unmapped status variant was passed through
SELECT
    SOURCE_SYSTEM,
    APPROVAL_STATUS,
    COUNT(*) AS cnt
FROM SILVER_AP_INVOICES
GROUP BY SOURCE_SYSTEM, APPROVAL_STATUS
ORDER BY SOURCE_SYSTEM, APPROVAL_STATUS;
```

**Expected values:** Only `APPROVED` and `PENDING`. Any other value is an unmapped variant that needs a rule update.

### 5d. Dropped Columns Verification

```sql
-- These columns should NOT exist in the Silver table
DESCRIBE TABLE SILVER_AP_INVOICES;
-- Confirm absence of: BAN_COMPANY, WD_TENANT_ID, SAP_COMPANY_CODE,
--   SAP_DOCUMENT_TYPE, ORACLE_ORG_ID, ORACLE_SOURCE
```

### 5e. NULL Checks on Required Fields

```sql
-- Every required field from the mapping should have zero NULLs
SELECT
    SOURCE_SYSTEM,
    COUNT_IF(INVOICE_ID IS NULL)       AS null_invoice_id,
    COUNT_IF(INVOICE_NUMBER IS NULL)   AS null_invoice_number,
    COUNT_IF(VENDOR_ID IS NULL)        AS null_vendor_id,
    COUNT_IF(VENDOR_NAME IS NULL)      AS null_vendor_name,
    COUNT_IF(INVOICE_DATE IS NULL)     AS null_invoice_date,
    COUNT_IF(DUE_DATE IS NULL)         AS null_due_date,
    COUNT_IF(INVOICE_AMOUNT IS NULL)   AS null_amount,
    COUNT_IF(CURRENCY_CODE IS NULL)    AS null_currency,
    COUNT_IF(PAYMENT_TERMS IS NULL)    AS null_payment_terms,
    COUNT_IF(GL_ACCOUNT IS NULL)       AS null_gl_account,
    COUNT_IF(COST_CENTER IS NULL)      AS null_cost_center,
    COUNT_IF(APPROVAL_STATUS IS NULL)  AS null_status,
    COUNT_IF(CREATED_AT IS NULL)       AS null_created_at
FROM SILVER_AP_INVOICES
GROUP BY SOURCE_SYSTEM;
```

**Expected result:** All counts = 0 for required fields.

### 5f. Currency Code Distribution

```sql
-- Verify only expected currencies appear (EUR, GBP for Baan; USD, GBP, EUR for Workday)
SELECT
    SOURCE_SYSTEM,
    CURRENCY_CODE,
    COUNT(*) AS cnt
FROM SILVER_AP_INVOICES
GROUP BY SOURCE_SYSTEM, CURRENCY_CODE
ORDER BY SOURCE_SYSTEM, CURRENCY_CODE;
```

### 5g. Payment Terms Variants (Context for Open Question #1)

```sql
-- Snapshot current payment term formats to inform the normalization decision
SELECT
    SOURCE_SYSTEM,
    PAYMENT_TERMS,
    COUNT(*) AS cnt
FROM SILVER_AP_INVOICES
GROUP BY SOURCE_SYSTEM, PAYMENT_TERMS
ORDER BY SOURCE_SYSTEM, PAYMENT_TERMS;
```

### 5h. Baan Cost Center Format Distribution (Context for Open Question #2)

```sql
-- Count BC-XX (old) vs BC-XXX (new) to inform normalization decision
SELECT
    CASE
        WHEN COST_CENTER RLIKE '^BC-[0-9]{2}$'  THEN 'OLD (BC-XX)'
        WHEN COST_CENTER RLIKE '^BC-[0-9]{3}$'   THEN 'NEW (BC-XXX)'
        ELSE 'OTHER'
    END AS format_type,
    COUNT(*) AS cnt
FROM SILVER_AP_INVOICES
WHERE SOURCE_SYSTEM = 'BAAN'
GROUP BY format_type;
```
