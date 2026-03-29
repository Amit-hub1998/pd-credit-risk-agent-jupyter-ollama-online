# PDHBC Intelligent Debugging Chatbot

> **Phase 1 — Agent Inside Jupyter Notebook**  
> HSBC Data Engineering Team | Credit Risk Pipeline

---

## Overview

This project implements an intelligent debugging chatbot for the **PDHBC Credit Risk Model**. It allows Finance teams to query specific attribute values and get structured root-cause answers — without involving DA/BA teams manually.

The agent combines:
- **RAG (Retrieval-Augmented Generation)** over BRD, ERD, and existing Jupyter Notebook logic
- **Pandas-based data lookup** across all 4 source tables
- **Groq LLM** (LLaMA 3.3 70B) for natural language structured answers

---

## Table Name Reference

| Table Name | File | Description |
|-----------|------|-------------|
| `credit_applications` | credit_applications.xls | Credit proposal records — anchor table for all joins |
| `basic_details` | basic_details.xls | Baseline risk ratings — fallback source when multi_ratings absent |
| `multi_ratings` | multi_ratings.xls | Primary risk ratings — preferred source, overrides basic_details |
| `downstream` | downstream.xls | Downstream customer ID mapping — customer-level, no proposal granularity |

---

## Key Variable Reference

| Variable | Table | Type | Nullable | Description |
|----------|-------|------|----------|-------------|
| `relationship_id` | All tables | VARCHAR(20) | NO | Root join key across all 4 tables. Prefix: 10x=India, FR=France, 20x=Germany |
| `credit_proposal_serial_no` | credit_applications, basic_details, multi_ratings | VARCHAR(10) | NO | Proposal-level join key. Format: YYMMDD |
| `customer_id` | basic_details, multi_ratings, downstream | VARCHAR(20) | NO | Customer identifier — mirrors relationship_id in current dataset |
| `final_rating` | basic_details, multi_ratings, final output | DECIMAL(4,1) | YES | Resolved credit risk rating. COALESCE(multi_ratings value, basic_details value) |
| `local_customer_id` | downstream | VARCHAR(20) | YES | Regional customer identifier. Format: LCL-IN-xxxx / LCL-FR-xxxx / LCL-DE-xxxx. ~60% populated |
| `app_status` | credit_applications | CHAR(1) | NO | Proposal status. A=Approved, L=Lapsed, R=Rejected. Pipeline keeps A only |
| `app_proposal_type` | credit_applications | VARCHAR(5) | NO | Proposal type. RN=Renewal, NL=New Loan, CN=Change-in-Terms. CN excluded by pipeline |
| `app_approval_dt` | credit_applications | DATE | YES | Approval date. Used to select latest proposal per relationship_id |
| `final_pd_basic_details` | basic_details | DECIMAL(6,4) | YES | Probability of Default from basic_details. Fallback when multi_ratings absent |
| `final_pd_multi_ratings` | multi_ratings | DECIMAL(6,4) | YES | Probability of Default from multi_ratings. Primary source |
| `rating_source` | final output | VARCHAR(10) | NO | Traceability flag. Values: multi_ratings / basic_details / MISSING |

---

## Folder Structure

```
HSBC_Conversion/
│
├── pdhbc_agent.ipynb                    ← Main agent notebook (entry point)
│
├── data/
│   ├── credit_applications.xls          ← Credit Applications table
│   ├── basic_details.xls                ← Basic Details (baseline ratings)
│   ├── multi_ratings.xls                ← Multi Ratings (primary ratings)
│   └── downstream.xls                   ← Downstream (customer ID mapping)
│
├── docs/
│   ├── BRD_CreditRiskPipeline.docx      ← Business Requirements Document
│   └── ERD_CreditRiskPipeline.docx      ← Entity Relationship Document
│
├── notebook/
│   └── CFC_PD_ECB.ipynb                 ← Existing model notebook (RAG source)
│
├── faiss_index/                          ← Auto-generated vector store (git ignored)
│
├── README.md                             ← This file
├── SETUP.md                              ← Step-by-step setup guide
├── ARCHITECTURE.md                       ← Technical architecture details
└── .gitignore
```

---

## Example Queries

```python
# Null rating debugging
"Why is final_rating null for relationship 1000101?"

# Downstream ID reverse lookup
"For local_customer_id LCL-IN-5530 what is the latest credit proposal?"

# Missing downstream ID
"Why is local_customer_id blank for relation 1000305?"

# General logic questions
"How is final_rating calculated in the pipeline?"

# Source tracing
"Where does final_rating come from for FR100201?"
```

---

## Key Design Decisions

| Decision | Reason |
|----------|--------|
| Jupyter Notebook agent | No new infrastructure — works in existing Citrix setup |
| FAISS over ChromaDB | ChromaDB requires SQLite 3.35+ which fails on Python 3.9 / Windows 11 |
| Groq LLM | Free, fast, no GPU needed, accessible via API key |
| Pandas over PySpark | Lightweight for local prototyping with mock Excel data |
| RAG on BRD + ERD + Notebook | Agent understands both business logic and technical implementation |

---

## Team

| Role | Responsibility |
|------|---------------|
| DA/BA | Run agent notebook, respond to Finance queries |
| Data Engineering | Maintain pipeline logic and RAG documents |
| Finance | Submit queries via the agent |
