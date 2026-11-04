# -Bank-Statement-Processing-Classification

# Bank Statement Processing & Classification System

A prototype that takes bank statement PDFs (text-based **or** scanned), extracts account details and
transactions, classifies every transaction **without any LLM**, and exports the result to Excel or CSV.

## Features

- Upload one or many PDF statements (multiple banks at once)
- **Auto-detects PDF type per page**: text layer is read directly, scanned pages go through OCR
- Extracts account details: holder name, account number, IFSC, MICR, branch, customer ID, period, bank
- Extracts transactions: date, description, debit, credit, balance
- **Non-LLM classification** using three layers: user feedback, heuristic rules, and a traditional ML model
- Editable category column, with a retrain button that learns from your corrections
- Data-quality checks: balance reconciliation, duplicate flags, warnings
- Summary tables and charts (by category, by month)
- Export to **Excel (.xlsx)** (Transactions, Account Info, Category Summary sheets) and **CSV**
- Password-protected PDFs supported

## Quick start

```bash
python -m venv venv
venv\Scripts\activate          # Windows   (Mac/Linux: source venv/bin/activate)
pip install -r requirements.txt
python -m streamlit run app.py
```

Then open http://localhost:8501 and upload a PDF.

## Project structure

```
app.py            Streamlit UI (upload, review/edit, charts, export)
extractor.py      PDF type detection, OCR, account info, transaction parsing, validation
classifier.py     Rule engine + ML classifier + counterparty extraction
requirements.txt  Dependencies
```

## Architecture

```
PDF(s)
  |
  v
[Per-page detection]  text layer >= 40 chars ?
  |-- yes --> PyMuPDF word extraction
  |-- no  --> render page (250 DPI) --> RapidOCR (ONNX, no external binary)
  v
[Line reconstruction]  cluster words/boxes by y-position, order by x
  v
[Account info]  regex patterns with label boundaries, bank from IFSC prefix
  v
[Transaction parser]  line-based state machine (works across layouts)
  v
[Validation]  order detection, direction inference, balance reconciliation, duplicates
  v
[Classifier]  user overrides -> rules -> TF-IDF + Logistic Regression
  v
Editable table --> Excel / CSV
```

## How extraction works

The parser is **line-based rather than template-based**, so it is not tied to a single bank's layout.

1. A line that starts with a date begins a new transaction. A serial-number column before the date
   (ICICI style) and a second "value date" column are handled.
2. The run of numbers at the **end** of the line is read as amounts: `[amount, balance]` or
   `[debit, credit, balance]`. A trailing integer column (e.g. Axis "Init. Br") is ignored.
3. Lines without a date continue the previous description (multi-line narrations). If the amounts
   were wrapped onto a later line, they are picked up there.
4. Headers, footers, page numbers, "Opening Balance", "Closing Balance" and summary lines are skipped.
5. **Debit vs credit** is decided in this order:
   explicit column position, then **running-balance difference**, then `Dr`/`Cr` markers, then keywords.
6. If balances only reconcile in reverse, the statement is treated as **newest-first** and re-ordered.

## How classification works (no LLM)

Each transaction goes through three layers, in order:

| Layer | What it does |
|---|---|
| 1. User feedback | Corrections you make in the UI are remembered and reused |
| 2. Heuristic rules | ~25 categories using regex keywords; direction-aware (e.g. "salary" only counts on credits). Generic channel rules (UPI, NEFT, cheque, wallet) are held back so a merchant name can win |
| 3. ML model | TF-IDF **character n-grams** + Logistic Regression, trained on a seed dataset. Handles typos, unseen merchants and OCR-merged words. Low-confidence predictions become `Uncategorized` |

Every row shows its `method` (rule / ml / user-feedback) and `confidence`. Clicking
**Apply my category corrections** retrains the model with your edits.

**Counterparty extraction** pulls out who the money went to/came from: a known-merchant lexicon
(space-insensitive), then structural parsing of UPI/NEFT/IMPS strings, with word-splitting for OCR-merged names.
Payment apps (PhonePe, Paytm, Google Pay) are only used when no real merchant is visible.

Categories: Salary & Income, Interest Earned, Refund & Cashback, Dividend, Cash Deposit, ATM Withdrawal,
Taxes, Bank Charges & Fees, EMI & Loans, Insurance, Investments, Food & Dining, Groceries, Fuel,
Travel & Transport, Shopping, Mobile & Internet, Utilities & Bills, Entertainment & Subscriptions,
Healthcare, Education, Rent & Housing, Cheque, Wallet & Payments, Bank Transfer (NEFT/IMPS/RTGS),
UPI Transfer, Uncategorized.

## Edge cases handled

| Scenario | Handling |
|---|---|
| Text vs scanned PDFs, or mixed within one file | Per-page auto-detection |
| Separate debit/credit columns, or one amount column | Balance-difference direction inference |
| `Dr`/`Cr`, `(Dr)`/`(Cr)`, negative or bracketed amounts | Parsed as markers/sign |
| Newest-first statements | Auto-detected, re-ordered |
| Multi-line descriptions, wrapped amounts | Continuation-line merging |
| Repeating page headers/footers | Skip patterns |
| Opening/closing balance rows | Detected and excluded from transactions |
| Serial-number column, value-date column, branch-code column | Stripped during parsing |
| Date formats (`01/04/24`, `1 Apr 2024`, `01-04-2024`) | dateutil, day-first |
| Indian number format (`1,23,456.78`) | Regex-based |
| Password-protected PDFs | Password field in sidebar |
| OCR errors | Balance reconciliation flags suspicious rows (`MISMATCH`) |
| Duplicate rows | `possible_duplicate` flag |
| OCR-merged words (`AMAZONREFUND`) | Char n-gram ML plus space-insensitive keyword fallback |
| Unrecognised layout | Warning shown with a hint to use Force OCR / higher DPI |

## Output

**Transactions** columns: `source_file, date, description, counterparty, debit, credit, balance,
category, method, confidence, balance_check, possible_duplicate, page`

The Excel file also has an **Account Info** sheet and a **Category Summary** sheet (count, total debit,
total credit per category). Columns are auto-sized and the header row is frozen.

> Tip: CSV files cannot store column widths, so Excel may show `####` in the date column until you
> widen it. The `.xlsx` export avoids this.

## Testing

The extractor was tested on synthetic statements in five different bank-style layouts (text-based,
multi-page, newest-first) plus two scanned versions. All 7 files matched the known answers on every
row for debit, credit, balance and date (37/37 each), with account name/number/IFSC/bank detected on all.

## Known limitations

- Tested on synthetic statements; real bank PDFs will need some tuning of skip patterns and layouts.
- Text from other columns (e.g. a reference-number column) can leak into the middle of a description when
  descriptions wrap, and fixed-width wrapping can split words. Amounts, dates and categories are unaffected.
- OCR is slower (several seconds per page) and can merge words or misread digits; the balance check is
  the safety net. Account holder names from scans may lose their spaces.
- The ML model is trained on a small seed set; accuracy improves with user corrections or a labelled dataset.
- Not designed for statements where amounts have no decimals or for non-INR formats.

## Future improvements

- **Column-aware parsing** using header x-positions to cleanly separate description, reference and amount columns
- Per-bank templates and automatic bank detection to choose parsing rules
- `pdfplumber` table detection for bordered tables as a first attempt
- Image preprocessing for scans (deskew, denoise, contrast) and OCR digit correction
- Training the classifier on a larger labelled dataset; merchant list maintained as a CSV
- Recurring-transaction detection (rent, EMI, subscriptions) and anomaly flagging
- Database storage, user accounts, REST API (FastAPI) and background processing for large files
- Automated test suite with a corpus of anonymised statements

## Tech stack

Python, Streamlit, PyMuPDF, RapidOCR (ONNX), scikit-learn, pandas, openpyxl, python-dateutil, wordninja

