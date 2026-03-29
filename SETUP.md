# Setup Guide — PDHBC Debugging Agent

> Tested on: Windows 11 | Python 3.9 | 16GB RAM | No GPU required

---

## Prerequisites

| Requirement | Version | Notes |
|------------|---------|-------|
| Python | 3.9+ | Tested on Python 3.9 |
| Jupyter Notebook | Any | Via pip or Anaconda |
| Internet access | Required | For Groq API + model download |
| Groq API Key | Free | console.groq.com — no credit card needed |

---

## Step 1 — Get a Groq API Key

1. Go to **https://console.groq.com**
2. Sign up with Google account (free, no credit card)
3. Navigate to **API Keys → Create API Key**
4. Copy the key — format: `gsk_xxxxxxxxxxxxxxxxxxxx`

---

## Step 2 — Clone the Repo

```bash
git clone <your-repo-url>
cd HSBC_Conversion
```

---

## Step 3 — Install Dependencies

Open Jupyter Notebook and run **Cell 1** of `pdhbc_agent.ipynb`:

```python
!pip install pandas xlrd openpyxl
!pip install python-docx
!pip install langchain langchain-community
!pip install faiss-cpu
!pip install sentence-transformers
!pip install groq
```

> ⚠️ After Cell 1 completes — do **Kernel → Restart** before running Cell 2.

---

## Step 4 — Prepare Your Data Files

Place your Excel files in the `data/` folder with these exact names:

| File | Table | Required Columns |
|------|-------|-----------------|
| `credit_applications.xls` | credit_applications | relationship_id, credit_proposal_serial_no, app_status, app_proposal_type, app_approval_dt |
| `basic_details.xls` | basic_details | relationship_id, customer_id, credit_proposal_serial_no, final_rating, final_pd_basic_details, internal_score_card_basic_details |
| `multi_ratings.xls` | multi_ratings | relationship_id, customer_id, credit_proposal_serial_no, final_rating, final_pd_multi_ratings, internal_score_card_multi_ratings |
| `downstream.xls` | downstream | relationship_id, customer_id, local_customer_id |

---

## Step 5 — Add Your Groq API Key

In `pdhbc_agent.ipynb`, **Cell 5**, replace:

```python
GROQ_API_KEY = "gsk_xxxxxxxxxxxx"
```

With your actual key from Step 1.

> ⚠️ Never commit your API key to Git. The `.gitignore` already excludes `.env` files — store your key there if needed.

---

## Step 6 — Run the Notebook

Run cells in order: **Cell 1 → 2 → 3 → 4 → 5 → 6**

| Cell | What It Does | Expected Output |
|------|-------------|-----------------|
| Cell 1 | Install packages | All installs succeed — restart kernel after |
| Cell 2 | Load all 4 Excel tables | Shapes printed for credit_applications, basic_details, multi_ratings, downstream |
| Cell 3 | Read BRD + ERD + Notebook | BRD: 7000+ chars, ERD: 4000+ chars, Notebook: 4000+ chars |
| Cell 4 | Build FAISS vector store | 40+ chunks built and saved to faiss_index/ |
| Cell 5 | Connect to Groq LLM | Test response about COALESCE/rating logic printed |
| Cell 6 | Define agent functions | No output — functions loaded into memory |

---

## Step 7 — Run Your First Query

In **Cell 7**:

```python
finance_query = "Why is final_rating null for relationship 1000101?"
debug_agent(finance_query);
```

Expected flow:
```
============================================================
📩 QUERY: Why is final_rating null for relationship 1000101?
============================================================
🔍 Step 1: Retrieving relevant logic from BRD/ERD...
✅ RAG context retrieved

🔍 Step 2: Extracting IDs from query...
✅ Relationship ID found: 1000101

🔍 Step 3: Looking up data across all tables...
✅ Data lookup complete
📊 Issues detected:
   ℹ️  credit_applications: CN proposal type exists — will be excluded
   ℹ️  credit_applications: Lapsed (L) proposals exist — will be excluded

🔍 Step 4: Building prompt for LLM...
🔍 Step 5: Calling Groq LLM...

============================================================
📋 FINAL ANSWER:
============================================================
...structured root cause answer...
============================================================
```

---

## Troubleshooting

| Error | Cause | Fix |
|-------|-------|-----|
| `RuntimeError: sqlite3 version` | ChromaDB incompatible with Python 3.9 | Use FAISS (already in Cell 4) |
| `XLRDError` on .xls file | File is actually .xlsx renamed | Change engine to `openpyxl` in Cell 2 |
| `Model decommissioned` (Groq) | Old Groq model retired | Use `llama-3.3-70b-versatile` in Cell 5 |
| Answer prints twice | Jupyter auto-displays return value | Add semicolon: `debug_agent(query);` |
| Only 26 chunks created | docx tables not read | Ensure Cell 3 uses `doc.tables` loop |
| local_customer_id not resolving | LCL ID regex not matching | Check format is LCL-IN-xxxx / LCL-FR-xxxx / LCL-DE-xxxx |

---

## .gitignore — What Gets Excluded

```
# API Keys
*.env

# FAISS vector store — auto-regenerated, large binary
faiss_index/

# Jupyter checkpoints
.ipynb_checkpoints/

# Python cache
__pycache__/
*.pyc

# Excel lock files
~$*.xls
~$*.xlsx
```
