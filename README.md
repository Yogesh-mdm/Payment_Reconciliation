
# Payment Reconciliation Pipeline

A Python + SQL pipeline that detects mismatches between a payment processor and an internal ledger — built to understand how reconciliation works in real payment systems.

---

## Why I Built This

Reconciliation is one of the most critical — and error-prone — operations in any payments company. When a processor says a transaction settled but the internal ledger has no record of it, real money is at risk. I built this pipeline from scratch to understand exactly where and how these mismatches happen, and how to catch them systematically.

---

## What It Does

Takes two data sources:
- `processor_transactions.csv` — what the payment processor recorded
- `internal_ledger.csv` — what the internal system recorded

Runs them through an ETL pipeline that:
1. **Extracts** raw CSV data from both sources
2. **Transforms** and normalizes inconsistencies between systems
3. **Loads** cleaned data into a SQLite database
4. **Reconciles** both sides using SQL queries
5. **Outputs** a structured mismatch report in Excel

---

## Project Structure
```
stripe_recon_project/
├── processor_transactions.csv   ← raw processor data
├── internal_ledger.csv          ← raw internal ledger data
├── recon_pipeline.ipynb         ← Jupyter notebook (main pipeline)
├── payments_recon.db            ← SQLite database (auto-created)
└── recon_report.xlsx            ← mismatch report (auto-generated)
```

---

## How to Run It

**Step 1 — Install dependencies**
```bash
pip install jupyter pandas openpyxl
```

**Step 2 — Launch the notebook**
```bash
jupyter notebook
```

**Step 3 — Open `recon_pipeline.ipynb` and run all cells top to bottom**

The SQLite database and Excel report are created automatically.

---

## The 3 Mismatch Types This Pipeline Catches

### Type 1 — Missing from Ledger
Transaction shows as settled in the processor but has no corresponding entry in the internal ledger.

**What this means operationally:** Money moved but was never recorded internally. This is a revenue recognition risk and can cause incorrect financial reporting.

**What a finance team should do:** Cross-check against the bank settlement file. If confirmed, create a manual ledger entry and investigate why the pipeline missed it.

---

### Type 2 — Ghost Entry in Ledger
A record exists in the internal ledger with no matching transaction in the processor.

**What this means operationally:** The ledger shows money that the processor has no record of. Could indicate a duplicate posting, a failed reversal, or a data pipeline bug.

**What a finance team should do:** Treat as a suspected duplicate. Flag for investigation before any payout is made against this entry.

---

### Type 3 — Amount Mismatch
Both the processor and ledger have a record for the same transaction ID, but the amounts differ.

**What this means operationally:** The most dangerous mismatch type. Could indicate FX conversion errors, partial captures being recorded as full, or fee deductions being applied inconsistently.

**What a finance team should do:** Pull the original authorization record and compare. Escalate to the processor if the discrepancy is on their side.

---

## Key Design Decision — Semantic Drift

The processor and ledger use different labels for the same transaction state:

| Processor | Ledger | Meaning |
|---|---|---|
| `success` | `settled` | Transaction completed |
| `failed` | `failed` | Transaction declined |
| `refunded` | `refunded` | Amount returned |

Without normalizing these labels first, every successful settled transaction would appear as a mismatch — a false positive that would make the entire report useless.

The Transform step maps `success → settled` before any reconciliation logic runs. This is the first thing to check if the pipeline ever produces unexpected results.

---

## Sample Output

Running the pipeline produces this summary:
```
✅ Data loaded into SQLite

📊 Reconciliation Summary
Missing from ledger:     15 rows
Ghost entries in ledger: 10 rows
Amount mismatches:       10 rows
Total mismatches:        45 rows

✅ recon_report.xlsx saved
```

The Excel report has 3 tabs — one per mismatch type — so a finance team can filter and action each category separately.

---

## What I Would Build Next at Scale

This pipeline works for a small dataset. At Large scale, here is what would need to change:

| Limitation now | Production solution |
|---|---|
| SQLite (single file) | PostgreSQL or BigQuery |
| Manual script run | Scheduled with Airflow or cron |
| No alerting | Slack/email alert if mismatch rate exceeds 1% |
| No audit trail | Add `detected_at` and `resolved_by` columns |
| No idempotency | Re-running creates duplicates — needs dedup logic |
| No late arrival handling | Processors send files 24-48hrs late — pipeline needs backdated matching |
| Single currency math | Multi-currency needs FX normalization before amount comparison |

---

## What I Learned Building This

**1. Silent bugs are the real risk.**
A mismatch that throws an error is easy to catch. A mismatch that produces wrong numbers silently — like semantic drift or integer division in percentage calculations — is what actually causes financial reporting errors.

**2. Reconciliation granularity matters.**
Matching on `transaction_id` alone is not enough at scale. You need to think about currency, date, merchant, and direction of money flow before declaring two records a match.

**3. The output is only as useful as its framing.**
A list of mismatches means nothing without context on what each type means and what action to take. I designed the report with finance users in mind, not just engineers.

---

## Tech Stack

- **Python 3** — pipeline scripting
- **Pandas** — data transformation
- **SQLite** — lightweight database for reconciliation queries
- **SQL** — LEFT JOINs, aggregations for mismatch detection
- **Jupyter Notebook** — interactive development and output visualization
- **OpenPyXL** — Excel report generation
