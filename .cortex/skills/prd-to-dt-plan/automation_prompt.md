You are running unattended in a Snowflake AGENT TASK; complete the task autonomously and do NOT ask clarifying questions.

## Task: PRD-to-Silver DT Implementation Plan

Check for CSV files on stage @COCO_WORKSHOP.PUBLIC.PRD_FILES and produce a structured implementation plan for SILVER_AP_INVOICES.

### Step 1: List and read files

Run:
```sql
LIST @COCO_WORKSHOP.PUBLIC.PRD_FILES;
```

For each .csv file found, create a temporary table and load it:
```sql
CREATE OR REPLACE TEMPORARY TABLE prd_raw_<suffix> AS
SELECT $1 AS raw_line
FROM @COCO_WORKSHOP.PUBLIC.PRD_FILES/<filename>
(FILE_FORMAT => (TYPE='CSV' FIELD_OPTIONALLY_ENCLOSED_BY='"' SKIP_HEADER=0));
```

Alternatively, use SELECT with a file format to read the CSVs. The key files expected are:
- sample_business_requirements_column_mapping.csv — Source-to-Silver field mappings
- sample_business_requirements_source_onboarding.csv — New source system details
- sample_business_requirements_business_rules.csv — Business rules (BR-001 through BR-010)

If no files are found on the stage, report PRD_TO_DT_PLAN_OK no_new_files=true and stop.

### Step 2: Analyze and produce the plan

Cross-reference all three files and produce a plan with these exact sections:

1. **Summary of Requested Changes** — New sources being onboarded, confirmed business rules, items deferred to Gold/Phase 2.

2. **Source-to-Silver Mapping Summary** — Unified Silver schema table showing every source column mapped to its Silver target, with types, transforms, and notes. Include dropped columns.

3. **Open Questions and Assumptions** — Apply these heuristics to surface problems (NEVER guess a resolution):
   - Undecided rules (marked NEEDS DECISION, TBD, OPEN QUESTION)
   - Contradictions between rules
   - Format ambiguity (multiple formats with no normalization rule)
   - Missing mappings for a listed source
   - Blocking dependencies (unsigned legal agreements, unprovisioned connectors)
   - Layer ambiguity (Silver vs Gold)
   - Unstated defaults (NULLs, timezones, missing fields)

4. **DDL Delta Plan** — Full CREATE DYNAMIC TABLE SQL for SILVER_AP_INVOICES with:
   - CTE per source with column aliasing
   - CASE statements for status normalization (BR-001)
   - QUALIFY dedup for Baan (BR-003)
   - SOURCE_SYSTEM literal per branch (BR-007)
   - Dropped columns noted as comments (BR-008)
   - TARGET_LAG = DOWNSTREAM (BR-009)
   - Placeholders clearly marked for warehouse and Bronze table FQNs

5. **Validation Queries** — SQL to run after implementation:
   - Row counts by source
   - Dedup effectiveness (no duplicate INVOICE_NUMBERs for Baan)
   - Status normalization check (only APPROVED/PENDING expected)
   - NULL checks on required fields
   - Currency and payment terms distribution

### Step 3: Report status

End your response with exactly one of:
- PRD_TO_DT_PLAN_OK files_processed=N sections=5
- PRD_TO_DT_PLAN_FAILED:<one-line reason>
