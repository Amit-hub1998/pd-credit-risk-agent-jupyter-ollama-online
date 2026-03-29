# Architecture — PDHBC Debugging Agent

---

## High-Level Flow

```
Finance Team Query (natural language)
            │
            ▼
    ┌───────────────────┐
    │   extract_ids()   │  ← Detects relationship_id, serial_no, local_customer_id
    └───────────────────┘
            │
     ┌──────┴──────┐
     ▼             ▼
local_customer_id?   Direct ID?
     │               │
     ▼               ▼
resolve_          lookup_
local_customer_id()   customer()
     │               │
     └──────┬─────────┘
            ▼
   ┌─────────────────────┐
   │  get_rag_context()  │  ← FAISS search over BRD + ERD + Notebook
   └─────────────────────┘
            │
            ▼
   ┌─────────────────────┐
   │   check_nulls()     │  ← Pandas null detection across all 4 tables
   └─────────────────────┘
            │
            ▼
   ┌─────────────────────┐
   │   Build Prompt      │  ← RAG context + live data + null findings combined
   └─────────────────────┘
            │
            ▼
   ┌─────────────────────┐
   │   call_llm()        │  ← Groq API: llama-3.3-70b-versatile
   └─────────────────────┘
            │
            ▼
  Structured Answer to Finance
  (root cause + table + rule + next step)
```

---

## Variable Reference — Complete Glossary

This section defines every variable used in the pipeline and agent, including which table it comes from, how it is used, and what its null behaviour means.

### Core Join Keys

| Variable | Tables | Type | Nullable | How It Is Used |
|----------|--------|------|----------|---------------|
| `relationship_id` | All 4 tables | VARCHAR(20) | NO | Root identifier that links all tables. Every lookup starts here. Prefix tells geography: 10x=India, FR=France, 20x=Germany |
| `credit_proposal_serial_no` | credit_applications, basic_details, multi_ratings | VARCHAR(10) | NO | Proposal-level join key. Combined with relationship_id to form composite PK. Format YYMMDD. NOT present in downstream table |
| `customer_id` | basic_details, multi_ratings, downstream | VARCHAR(20) | NO | Customer identifier. Currently mirrors relationship_id in the dataset. Used as join key alongside relationship_id |

### credit_applications Table Variables

| Variable | Type | Nullable | How It Is Used |
|----------|------|----------|---------------|
| `relationship_id` | VARCHAR(20) | NO | PK — root anchor for all joins. Every other table joins back to credit_applications on this key |
| `credit_proposal_serial_no` | VARCHAR(10) | NO | PK — proposal-level key. Combined with relationship_id uniquely identifies one proposal |
| `app_status` | CHAR(1) | NO | Filter variable. Pipeline keeps only A (Approved). L=Lapsed and R=Rejected rows are excluded before any join happens |
| `app_proposal_type` | VARCHAR(5) | NO | Filter variable. CN (Change-in-Terms) rows are excluded. Only RN (Renewal) and NL (New Loan) reach the output |
| `app_approval_dt` | DATE | YES | Sorting variable. Used to select the single latest approved non-CN proposal per relationship_id using MAX(app_approval_dt) |

### basic_details Table Variables

| Variable | Type | Nullable | How It Is Used |
|----------|------|----------|---------------|
| `relationship_id` | VARCHAR(20) | NO | FK to credit_applications. INNER JOIN — a proposal must exist in basic_details to appear in output |
| `customer_id` | VARCHAR(20) | NO | PK — part of composite key with relationship_id + serial_no. Also used as join key to downstream table |
| `credit_proposal_serial_no` | VARCHAR(10) | NO | PK — completes composite key |
| `final_rating` | DECIMAL(4,1) | YES | Fallback credit risk rating. Used in COALESCE(multi_ratings.final_rating, basic_details.final_rating). If both are NULL, output final_rating is NULL and rating_source = MISSING |
| `final_pd_basic_details` | DECIMAL(6,4) | YES | Fallback Probability of Default. Scale 0.0000–1.0000. Used in COALESCE with multi_ratings value |
| `internal_score_card_basic_details` | VARCHAR(20) | YES | Fallback scorecard identifier. Values: GMIS-xxxx (internal India), EXT-CCxx (external), No-GMIS (unavailable) |

### multi_ratings Table Variables

| Variable | Type | Nullable | How It Is Used |
|----------|------|----------|---------------|
| `relationship_id` | VARCHAR(20) | NO | FK to credit_applications. LEFT JOIN — absent rows fall back to basic_details values |
| `customer_id` | VARCHAR(20) | NO | PK — part of composite key |
| `credit_proposal_serial_no` | VARCHAR(10) | NO | PK — completes composite key. ~40% of rows may be absent particularly for newer 260xxx series proposals |
| `final_rating` | DECIMAL(4,1) | YES | Primary credit risk rating. Takes precedence over basic_details.final_rating in COALESCE resolution |
| `final_pd_multi_ratings` | DECIMAL(6,4) | YES | Primary Probability of Default. Preferred source in COALESCE |
| `internal_score_card_multi_ratings` | VARCHAR(20) | YES | Primary scorecard identifier. Same taxonomy as basic_details |

### downstream Table Variables

| Variable | Type | Nullable | How It Is Used |
|----------|------|----------|---------------|
| `relationship_id` | VARCHAR(20) | NO | FK to credit_applications. LEFT JOIN on relationship_id + customer_id only — no serial number used |
| `customer_id` | VARCHAR(20) | NO | FK — join key alongside relationship_id. No proposal granularity — one local_customer_id per customer across all proposals |
| `local_customer_id` | VARCHAR(20) | YES | Regional customer identifier assigned by local entity. Format: LCL-IN-xxxx (India), LCL-FR-xxxx (France), LCL-DE-xxxx (Germany). ~60% populated — NULL where not yet mapped |

### Final Output Variables

| Variable | Source | Type | Nullable | How It Is Used |
|----------|--------|------|----------|---------------|
| `final_rating` | COALESCE(multi_ratings, basic_details) | DECIMAL(4,1) | YES | Resolved risk rating sent to Finance. NULL only when both sources are absent |
| `final_pd` | COALESCE(multi_ratings, basic_details) | DECIMAL(6,4) | YES | Resolved Probability of Default |
| `internal_score_card` | COALESCE(multi_ratings, basic_details) | VARCHAR(20) | YES | Resolved scorecard identifier |
| `rating_source` | Pipeline logic | VARCHAR(20) | NO | Traceability flag. Values: multi_ratings (primary used), basic_details (fallback used), MISSING (both absent) |
| `local_customer_id` | downstream table | VARCHAR(20) | YES | Carried through from downstream table. NULL where no downstream mapping exists |

---

## COALESCE Resolution Logic

```
final_rating = COALESCE(multi_ratings.final_rating, basic_details.final_rating)

Scenario 1: multi_ratings present  → multi_ratings wins  | rating_source = multi_ratings
Scenario 2: multi_ratings absent   → basic_details used  | rating_source = basic_details
Scenario 3: Both absent            → NULL                 | rating_source = MISSING
```

Same logic applies to `final_pd` and `internal_score_card`.

---

## CREAP Filter Logic

Applied to `credit_applications` before any joins:

```
Step 1: app_status = 'A'                    → Keep Approved only
Step 2: app_proposal_type != 'CN'           → Exclude Change-in-Terms
Step 3: MAX(app_approval_dt)                → Latest proposal per relationship_id
```

Implemented in agent function `get_latest_valid_proposal()`.

---

## ID Detection Logic

The agent handles 3 types of identifiers extracted via regex:

| ID Type | Pattern | Example | Action |
|---------|---------|---------|--------|
| `relationship_id` (India) | `10\d{5,}` | 1000101 | Direct lookup across all tables |
| `relationship_id` (Germany) | `20\d{5,}` | 20100374 | Direct lookup across all tables |
| `relationship_id` (France) | `FR\d+` | FR100201 | Direct lookup across all tables |
| `credit_proposal_serial_no` | 6-digit number | 260101 | Narrows lookup to specific proposal |
| `local_customer_id` (India) | `LCL-IN-\d+` | LCL-IN-5530 | Reverse lookup in downstream → get relationship_id |
| `local_customer_id` (France) | `LCL-FR-\d+` | LCL-FR-7721 | Reverse lookup in downstream → get relationship_id |
| `local_customer_id` (Germany) | `LCL-DE-\d+` | LCL-DE-9910 | Reverse lookup in downstream → get relationship_id |

---

## Downstream ID Reverse Lookup

When Finance provides a `local_customer_id` instead of a `relationship_id`:

```
local_customer_id (e.g. LCL-IN-5530)
        │
        ▼
Search downstream table WHERE local_customer_id = 'LCL-IN-5530'
        │
        ▼
Extract relationship_id + customer_id from matching row
        │
        ▼
Use relationship_id for full lookup across all 4 tables
        │
        ▼
Apply credit_applications filters → find latest valid proposal
```

---

## Null Detection Rules

Implemented in `check_nulls()`:

| Check | Condition | Agent Message |
|-------|-----------|---------------|
| multi_ratings entirely absent | No row in multi_ratings for relationship_id | ⚠️ multi_ratings absent — basic_details fallback will be used |
| multi_ratings column null | final_rating / final_pd_multi_ratings is NaN | ⚠️ multi_ratings.{col} is NULL |
| basic_details entirely absent | No row in basic_details for relationship_id | ❌ basic_details absent — final_rating will be MISSING |
| basic_details column null | final_rating in basic_details is NaN | ⚠️ basic_details.{col} is NULL |
| downstream absent | No row in downstream for relationship_id | ℹ️ local_customer_id will be NULL |
| CN proposal | app_proposal_type = CN in credit_applications | ℹ️ CN proposal exists — excluded by filter |
| Lapsed proposal | app_status = L in credit_applications | ℹ️ Lapsed proposal exists — excluded by filter |

---

## RAG Knowledge Base

| Source | Content Captured | Characters |
|--------|-----------------|-----------|
| BRD_CreditRiskPipeline.docx | Business rules, attribute specs, filter logic, geography conventions | ~7,900 |
| ERD_CreditRiskPipeline.docx | Table relationships, join keys, cardinality, null propagation rules | ~5,000 |
| CFC_PD_ECB.ipynb | PySpark implementation logic, cell-by-cell code and comments | ~4,900 |

**Embedding model:** `all-MiniLM-L6-v2` (HuggingFace, ~90MB, CPU-only)  
**Vector store:** FAISS (saved locally to `faiss_index/`)  
**Chunk size:** 500 characters | **Overlap:** 50 characters | **Total chunks:** ~48

---

## Prompt Engineering Strategy

```
[System Context]
You are an expert debugging assistant for the PDHBC Credit Risk Pipeline at HSBC.
Use ONLY the information provided. Do not invent table names or columns.

[Finance Query]
{raw query from Finance team}

[RAG Context — top 4 FAISS chunks]
{relevant BRD/ERD/Notebook logic}

[Live Data — Pandas lookup]
{credit_applications rows}
{basic_details rows}
{multi_ratings rows}
{downstream rows}
{null/filter findings from check_nulls()}

[Output Format]
1. Direct answer
2. Root cause (table + column)
3. Pipeline rule that applies
4. Recommended next step for Finance team
```

Key guardrail: `"Use ONLY the information provided"` prevents the LLM from hallucinating non-existent table names or columns.

---

## Technology Stack

| Component | Technology | Why Chosen |
|-----------|-----------|-----------|
| Data processing | Pandas | Lightweight — no PySpark cluster needed for local prototyping |
| Vector store | FAISS | No SQLite dependency — compatible with Python 3.9 and Windows 11 |
| Embeddings | HuggingFace all-MiniLM-L6-v2 | Free, 90MB, runs on CPU, strong semantic matching |
| LLM | Groq llama-3.3-70b-versatile | Free tier, fast inference, no local GPU required |
| Notebook | Jupyter | Matches existing HSBC Citrix environment exactly |
| Doc parsing | python-docx | Reads both paragraph text and table rows from .docx files |
