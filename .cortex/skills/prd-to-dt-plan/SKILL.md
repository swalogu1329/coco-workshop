---
name: prd-to-dt-plan
description: "Turn PRD-style requirement files into a structured implementation plan for a Snowflake Dynamic Table. Use when: the user provides business requirements, source onboarding docs, or PRD files (XLSX, CSV) and wants an implementation plan for a Silver/Gold Dynamic Table. Triggers: PRD, business requirements, source onboarding, implementation plan, Dynamic Table plan, DT plan, new source system, Silver layer plan, pipeline plan from requirements."
---

# PRD to Dynamic Table Plan

Turn product/business requirement documents into a structured, reviewable implementation plan for a target Snowflake Dynamic Table.

## When to Use

- User provides PRD-style files (XLSX, CSV) describing new source systems, field mappings, or business rules
- User wants to understand what changes are needed for a Dynamic Table before writing any SQL
- User asks to "plan" or "analyze" requirements for a pipeline update

## Inputs

Ask the user for these inputs. Only `prd_paths` is strictly required — infer the rest from file contents when possible.

| Input | Required | Description |
|-------|----------|-------------|
| `prd_paths` | Yes | One or more file paths to PRD/requirements files (XLSX or CSV) |
| `target_dynamic_table` | Recommended | Fully qualified name of the target DT (e.g. `DB.SCHEMA.SILVER_AP_INVOICES`). If not provided, infer from file contents or ask. |
| `existing_sources` | No | List of source systems already integrated. Helps distinguish new vs. existing. If target DT exists in Snowflake, query its definition instead of asking. |

## Workflow

### Step 1: Parse the PRD Files

**Goal:** Extract structured data from the requirement files.

**Actions:**
1. Run the parse script to extract sheets/rows from XLSX or read CSV directly:
   ```bash
   uv run --project <SKILL_DIRECTORY> python <SKILL_DIRECTORY>/scripts/parse_prd.py <file_path> [<file_path2> ...]
   ```
   The script outputs JSON to stdout with sheet names as keys and rows as arrays of objects.
2. For CSV files, read them directly with the Read tool — no script needed.
3. Identify which sheets/files contain **source onboarding** info vs. **business rules** vs. **field mappings**. Use column headers as signals:
   - Source onboarding: columns like `Source System`, `ERP Platform`, `Region`, `Go-Live`, `Status`
   - Business rules: columns like `Rule ID`, `Category`, `Rule Description`, `Decision`
   - Field mappings: columns like `Source Field`, `Target Field`, `Data Type`, `Transformation`

**Output:** Parsed, categorized content ready for analysis.

### Step 2: Identify the Target DT Context

**Goal:** Understand what the target Dynamic Table looks like today.

**Actions:**
1. If `target_dynamic_table` is provided and exists in Snowflake, run:
   ```sql
   SELECT GET_DDL('TABLE', '<target_dynamic_table>');
   ```
   to retrieve the current definition, including existing UNION ALL branches and column list.
2. If the DT does not exist yet, note that this is a greenfield build.
3. Identify which source systems are already integrated (look for `SOURCE_SYSTEM` literals or table references in the DDL).

**Output:** Current state of the target DT (or "greenfield").

### Step 3: Analyze Requirements

**Goal:** Produce the three required output sections by cross-referencing the parsed PRD content against the target DT context.

**Actions:**

**3a — New Source Systems:**
- Compare source systems mentioned in the PRD against those already in the DT.
- For each new source, extract: name, platform, region, data delivery method, refresh cadence, go-live status, and any blocking dependencies (legal, technical).

**3b — Silver-Layer Changes (Fields + Business Rules):**
- Walk every business rule row. For each rule, determine:
  - Does it apply at Bronze, Silver, or Gold? (Only include Silver-layer rules in this section.)
  - Is it a new rule or a change to an existing rule?
  - What is the implementation approach? (CASE statement, QUALIFY dedup, new column, DMF, etc.)
- Walk any field mapping rows. Flag new columns, renamed columns, or type changes.
- Cross-reference: if a new source introduces fields that don't exist in the current DT, flag them.

**3c — Ambiguities and Open Questions:**
Apply these heuristics to surface problems. **Never guess or assume a resolution — always surface the question.**

| Heuristic | What to Flag |
|-----------|-------------|
| **Undecided rules** | Any rule marked "NEEDS DECISION", "TBD", "OPEN QUESTION", or lacking a confirmed owner/date |
| **Contradictions** | Two rules that conflict (e.g., "no FX at Silver" vs. a threshold stated in USD equivalent) |
| **Format ambiguity** | Source data with multiple formats (e.g., cost center codes changed mid-stream) and no normalization rule |
| **Missing mappings** | A new source is listed but no field mapping or status mapping is provided for it |
| **Blocking dependencies** | Legal reviews, unsigned agreements, unprovisioned connectors |
| **Layer ambiguity** | A rule that could apply at Silver or Gold with no explicit assignment |
| **Unstated defaults** | What happens to NULLs? What timezone are timestamps in? What if a required field is missing from a source? |

**CRITICAL RULE:** If information is missing or ambiguous, add it to the Open Questions list. Do not invent an answer. The purpose of this skill is to make implicit assumptions explicit so they can be resolved before code is written.

**⚠️ STOP**: Present the draft analysis to the user for review before producing the final output.

### Step 4: Produce the Final Plan

**Goal:** Deliver the structured output document.

Format the plan as a markdown document with these exact sections:

```
# Dynamic Table Implementation Plan
## Target: <FULLY_QUALIFIED_DT_NAME>
### Generated from: <list of input files>

---

## 1. New Source Systems
For each new source:
- **Name** / Platform / Region
- Data delivery method and cadence
- Go-live status and blocking dependencies
- Key contacts

## 2. Silver-Layer Changes
### 2a. New UNION ALL Branches
- Source table references to add
- SOURCE_SYSTEM literal value
- System-specific columns to drop at the boundary

### 2b. Business Rules
For each rule affecting Silver:
- Rule ID + summary
- Implementation approach (CASE, QUALIFY, new column, etc.)
- Which sources it applies to

### 2c. New or Changed Columns
- Column name, type, source, transformation

## 3. Ambiguities and Open Questions
Numbered list. Each item includes:
- The question
- Why it matters (what breaks or is undefined without an answer)
- Suggested owner or audience for the decision

## 4. Out of Scope (Noted for Later)
Items explicitly deferred to Gold, Phase 2, or future work.
```

Write the plan to `<project_root>/<target_dt_name>_plan.md` (lowercase, underscores).

**⚠️ STOP**: Present the final plan and ask the user if they want revisions.

## Stopping Points

- ✋ After Step 3: Draft analysis review
- ✋ After Step 4: Final plan review

## Output

A markdown file at `<project_root>/<target_dt_name>_plan.md` containing the four-section implementation plan. The plan is a decision-support document — it does not generate SQL. Its job is to make every assumption visible so the team can resolve open questions before implementation begins.

## Notes

- **XLSX support** requires the parse script (`scripts/parse_prd.py`). CSV files are read directly.
- If the target DT exists in Snowflake, the skill queries its DDL for context. If there is no active Snowflake connection, skip this step and note it in the output.
- This skill deliberately does NOT generate SQL. The plan is the deliverable. SQL authoring is a separate step (use the `dynamic-tables` or `sql-author` skill).
