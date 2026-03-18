# PropLedger — Property Accounting System
## Architecture Plan & Claude Code Development Brief

**Entity:** Strathcona 1 LLC (expandable to multi-entity/multi-property)  
**Purpose:** Replace manual Excel-based property bookkeeping with a local, extensible accounting app  
**Source of Truth:** This document. Use it to generate a full sprint-by-sprint development plan before writing any code.

---

## 1. Project Overview

### What This System Does
- Imports bank transactions from Chase (checking + credit cards) and Citizens Bank via CSV
- Categorizes transactions against a property-specific Chart of Accounts
- Tracks tenants, leases, and rent payments per property
- Manages an intercompany loan ledger (Strathcona 1 LLC → Magnet Labs)
- Generates Monthly P&L reports, Annual P&L rollups, Schedule E tax exports, and a Rent Roll
- Is architected from day one for multi-property / multi-entity expansion
- Stubs a Plaid API integration path for future auto-sync

### What It Does NOT Do (out of scope v1)
- Double-entry bookkeeping (single-ledger categorization only, v2 can add)
- Payroll or invoice management
- Lease document storage (flat metadata only)
- Multi-user / role-based access (single-user local app)

---

## 2. Technology Stack

All tools are free and open source. No paid services required to run.

| Layer | Technology | Rationale |
|---|---|---|
| Backend API | **Python 3.11+ / FastAPI** | Simple, self-documenting via OpenAPI, excellent for financial data processing |
| Database | **SQLite** (via SQLAlchemy ORM) | Single-file, zero-server, easy to back up; schema designed for Postgres migration |
| Migrations | **Alembic** | Schema versioning from day one |
| Data Processing | **Pandas** | CSV import parsing, report calculation |
| Excel Export | **openpyxl** | Schedule E and P&L Excel exports |
| Frontend | **React 18 + Vite + TailwindCSS** | Fast local dev, component-based, easy to extend |
| HTTP Client | **Axios** (frontend → backend) | Standard |
| Package Mgmt | **uv** (Python) + **npm** (JS) | Modern, fast |
| Dev Runner | **Makefile** | Single `make dev` command starts everything |

**Future migration paths (do not build yet, but architect for):**
- SQLite → PostgreSQL: swap SQLAlchemy connection string only
- CSV import → Plaid API: interface is stubbed, implementation swapped
- Local → hosted: FastAPI + React are both cloud-deployable as-is

---

## 3. Monorepo Project Structure

```
propledger/
├── Makefile                        # make dev, make migrate, make seed
├── README.md
├── .env.example
│
├── backend/
│   ├── pyproject.toml              # uv-managed dependencies
│   ├── alembic.ini
│   ├── migrations/
│   │   └── versions/
│   ├── data/
│   │   ├── propledger.db           # SQLite database (gitignored)
│   │   └── uploads/                # CSV staging area
│   │
│   └── app/
│       ├── main.py                 # FastAPI app, CORS, router registration
│       ├── database.py             # SQLAlchemy engine + session factory
│       ├── config.py               # Settings via pydantic-settings
│       │
│       ├── models/                 # SQLAlchemy ORM models
│       │   ├── entity.py
│       │   ├── property.py
│       │   ├── bank_account.py
│       │   ├── transaction.py
│       │   ├── category.py
│       │   ├── tenant.py
│       │   ├── lease.py
│       │   ├── rent_payment.py
│       │   └── loan.py
│       │
│       ├── schemas/                # Pydantic request/response schemas
│       │   └── (mirrors models/)
│       │
│       ├── routers/                # FastAPI route handlers
│       │   ├── entities.py
│       │   ├── properties.py
│       │   ├── bank_accounts.py
│       │   ├── transactions.py     # CRUD + bulk categorization
│       │   ├── imports.py          # CSV upload endpoint
│       │   ├── tenants.py
│       │   ├── leases.py
│       │   ├── rent_payments.py
│       │   ├── loans.py
│       │   └── reports.py          # All report generation endpoints
│       │
│       └── services/
│           ├── csv_parser.py       # Chase + Citizens CSV format handlers
│           ├── plaid_stub.py       # Plaid interface (stubbed, not wired)
│           ├── categorizer.py      # Rule-based auto-categorization engine
│           ├── report_pl.py        # Monthly + Annual P&L logic
│           ├── report_schedule_e.py # IRS Schedule E computation
│           ├── report_rent_roll.py  # Tenant/rent roll logic
│           ├── report_loan.py       # Intercompany loan ledger logic
│           └── exporter.py         # Excel/CSV export via openpyxl
│
└── frontend/
    ├── package.json
    ├── vite.config.ts
    ├── tailwind.config.ts
    └── src/
        ├── main.tsx
        ├── App.tsx
        ├── api/                    # Axios API client functions (mirrors backend routers)
        ├── pages/
        │   ├── Dashboard.tsx       # KPI summary: net income, rent collected, YTD
        │   ├── Transactions.tsx    # Table view, bulk categorize, filters
        │   ├── Import.tsx          # CSV drag-and-drop upload UI
        │   ├── Properties.tsx      # Property list + detail
        │   ├── Tenants.tsx         # Rent roll + tenant management
        │   ├── Loans.tsx           # Intercompany loan ledger
        │   └── Reports.tsx         # Report generator + download
        └── components/
            ├── CategorySelect.tsx
            ├── TransactionTable.tsx
            ├── MonthPicker.tsx
            └── ExportButton.tsx
```

---

## 4. Data Model

### Core Entities

**Entity** (LLC or company)
```
id, name, type (LLC | Corp | Individual), tax_id, address, created_at
```

**Property**
```
id, entity_id (FK), name, address, city, state, zip,
property_type (single_family | multi_unit | commercial),
purchase_date, purchase_price, notes
```

**BankAccount**
```
id, entity_id (FK), property_id (FK nullable),
account_name, account_number_last4, bank_name,
account_type (checking | credit_card | savings),
plaid_account_id (nullable, for future Plaid sync),
is_active
```

**Transaction** (the central table)
```
id, bank_account_id (FK), property_id (FK nullable),
transaction_date, post_date, description_raw, description_clean,
amount, transaction_type (debit | credit | payment | fee | transfer),
category_id (FK nullable), entity_tag (nullable — "Magnet Labs"),
notes, import_batch_id (FK), is_reconciled, is_intercompany,
created_at
```

**Category** (Chart of Accounts — property-focused)
```
id, name, code, parent_id (FK nullable — supports subcategories),
category_type (income | operating_expense | debt_service | capital | transfer),
schedule_e_line (nullable — maps to IRS Schedule E line numbers),
is_system (bool — protects built-in categories from deletion)
```

Built-in seed categories (mirroring your existing spreadsheet):
- Income: Rental Income, Refunds/Credits
- Operating Expenses: Repairs & Maintenance, Utilities, Insurance, Bank Fees, Office Expense, Meals & Entertainment, Licenses & Fees, Property Taxes, Professional Fees, Other
- Debt Service: Loan Payments (WF), Loan Payments (Dyntec), Credit Card Payments
- Intercompany: Magnet Labs - R&D, Magnet Labs - Travel, Magnet Labs - SaaS
- Capital: Capital Purchase

**ImportBatch**
```
id, bank_account_id (FK), filename, import_date,
row_count, duplicate_count, source (chase_csv | citizens_csv | plaid | manual),
status (pending | complete | error), notes
```

---

### Tenant / Lease Module

**Tenant**
```
id, property_id (FK), first_name, last_name, email, phone,
turbotenant_id (nullable), notes, is_active
```

**Lease**
```
id, tenant_id (FK), property_id (FK),
unit_description (e.g. "Main Unit", "Lower Unit"),
start_date, end_date (nullable — month-to-month),
monthly_rent, security_deposit, lease_type (fixed | month_to_month),
is_active
```

**RentPayment**
```
id, lease_id (FK), transaction_id (FK nullable — links to matched bank transaction),
payment_date, amount, payment_method, period_month, period_year,
status (expected | received | partial | late), notes
```

---

### Intercompany Loan Module

**Loan**
```
id, lender_entity_id (FK), borrower_entity_id (FK),
principal_total (computed), interest_rate (nullable),
term_months (nullable), start_date, status (open | closed),
notes
```

**LoanAdvance**
```
id, loan_id (FK), transaction_id (FK — the actual expense transaction),
advance_date, amount, category (SaaS | R&D | Travel | Professional),
description, notes
```

---

## 5. CSV Import Architecture

### Supported Formats (v1)

**Chase Checking (Chase5271 format)**
```
Headers: Details | Posting Date | Description | Amount | Type | Balance | Account
Key logic: Negative = debit, Positive = credit; "Account" column carries pre-existing labels
```

**Chase Credit Card (Chase9268 format)**
```
Headers: Card | Transaction Date | Post Date | Description | Category | Type | Amount | Account
Key logic: Amount is signed; "Account" column carries pre-existing labels
```

**Citizens Bank (future, stub with interface)**
```
Interface defined but implementation empty. Same return type as Chase parsers.
```

### Import Service Contract
All CSV parsers must return a standardized `ParsedTransaction` list:
```python
@dataclass
class ParsedTransaction:
    transaction_date: date
    post_date: date
    description_raw: str
    amount: Decimal       # negative = expense, positive = income
    transaction_type: str
    source_label: str     # raw "Account" label from CSV if present
    raw_row: dict         # original row for audit
```

### Duplicate Detection
- Hash: `(bank_account_id + transaction_date + amount + description_raw)`
- On import, flag duplicates but do not block — show count in import summary
- User can review and delete duplicates via UI

### Plaid Stub
```python
# services/plaid_stub.py
class PlaidService:
    """
    Stub implementation. Wire real Plaid SDK here when ready.
    Implement: get_accounts(), get_transactions(start, end), exchange_token()
    """
    def get_transactions(self, account_id: str, start: date, end: date) -> list[ParsedTransaction]:
        raise NotImplementedError("Plaid integration not yet configured")
```

---

## 6. Auto-Categorization Engine

A rule-based engine that runs on import and can be re-triggered manually.

**Rules table** (stored in DB, user-editable via UI):
```
id, match_field (description | amount | transaction_type),
match_type (contains | starts_with | equals | regex),
match_value, category_id (FK), entity_tag (nullable),
priority (lower = higher priority), is_active
```

**Seed rules** (derived from your existing spreadsheet labels):
- description contains "FOREMOST" → Insurance
- description contains "CONSUMERS ENERGY" or "DTE Energy" → Utilities
- description contains "CHARTER TOWNSHIP" → Utilities
- description contains "WF LOAN/LINE" → Loan Payments (WF)
- description contains "Dyntec" → Loan Payments (Dyntec)
- description contains "MONTHLY SERVICE FEE" → Bank Fees
- description contains "OPENAI" or "MIDJOURNEY" or "UIZARD" → Magnet Labs - R&D
- description contains "FIREFLIES" or "EWAY-CRM" → Magnet Labs - SaaS
- description contains "HOME DEPOT" or "LOWES" or "AMAZON MKTPL" → Repairs & Maintenance
- description contains "TurboTenant" or "TURBOTENANT" → Rental Income
- description contains "PURCHASE INTEREST CHARGE" → Bank Fees (Interest)

---

## 7. Reporting Layer

All reports are computed server-side from the Transaction table. No pre-computed aggregates — always derived fresh from source data.

### Report: Monthly P&L
**Endpoint:** `GET /reports/pl?entity_id=&property_id=&month=&year=`  
**Output:** JSON (for UI display) + Excel download option  
**Structure mirrors existing spreadsheet:**
- Total Income (Rental Income + Refunds/Credits)
- Operating Expenses by category
- Debt Service (Loan Payments + CC Payments)
- Net Income / Loss
- Key Metrics sidebar (Expense Ratio, DSCR, Occupancy)

### Report: Annual P&L Rollup
**Endpoint:** `GET /reports/pl/annual?entity_id=&property_id=&year=`  
**Output:** 12-column monthly view + YTD totals column

### Report: Schedule E Export
**Endpoint:** `GET /reports/schedule-e?entity_id=&property_id=&year=`  
**Output:** Excel file formatted to IRS Schedule E Part I lines  
**Line mappings** (from Category.schedule_e_line field):
- Line 3: Rents received
- Line 5: Advertising
- Line 6: Auto and travel
- Line 7: Cleaning and maintenance
- Line 8: Commissions
- Line 9: Insurance
- Line 11: Legal and professional fees
- Line 12: Management fees
- Line 13: Mortgage interest paid to banks
- Line 14: Other interest
- Line 15: Repairs
- Line 16: Supplies
- Line 17: Taxes
- Line 18: Utilities
- Line 19: Depreciation (manual entry only — not auto-computed)
- Line 22: Other

### Report: Rent Roll
**Endpoint:** `GET /reports/rent-roll?property_id=&as_of_date=`  
**Output:** Per-unit table showing tenant, lease dates, monthly rent, YTD collected, YTD expected, variance

### Report: Intercompany Loan Ledger
**Endpoint:** `GET /reports/loan?loan_id=&as_of_date=`  
**Output:** Running balance table — advance date, description, category, amount, cumulative total  
**Also outputs:** Summary by category (SaaS / R&D / Travel), monthly advance totals

---

## 8. API Design Summary

```
# Entities
GET/POST         /entities
GET/PUT/DELETE   /entities/{id}

# Properties
GET/POST         /properties
GET/PUT/DELETE   /properties/{id}

# Bank Accounts
GET/POST         /bank-accounts
GET/PUT/DELETE   /bank-accounts/{id}

# Transactions
GET              /transactions              # filterable: account, property, category, date range, uncategorized
POST             /transactions              # manual entry
PUT              /transactions/{id}         # update category, notes, entity_tag
DELETE           /transactions/{id}
POST             /transactions/bulk-categorize  # [{id, category_id}]
POST             /transactions/re-categorize    # re-run auto-categorizer on selection

# Imports
POST             /imports/upload            # multipart CSV upload
GET              /imports                   # import history
GET              /imports/{id}              # import detail + transaction preview

# Categorization Rules
GET/POST         /categorization-rules
GET/PUT/DELETE   /categorization-rules/{id}

# Tenants
GET/POST         /tenants
GET/PUT/DELETE   /tenants/{id}

# Leases
GET/POST         /leases
GET/PUT/DELETE   /leases/{id}

# Rent Payments
GET/POST         /rent-payments
GET/PUT/DELETE   /rent-payments/{id}
POST             /rent-payments/match-transaction  # link a rent payment to a transaction

# Loans
GET/POST         /loans
GET/PUT/DELETE   /loans/{id}
GET              /loans/{id}/advances

# Reports
GET              /reports/pl                    # Monthly P&L
GET              /reports/pl/annual             # Annual P&L
GET              /reports/schedule-e            # Schedule E (JSON + Excel download)
GET              /reports/rent-roll             # Rent Roll
GET              /reports/loan                  # Loan Ledger
```

---

## 9. Seed Data (Initial State)

On first run (`make seed`), populate:

1. **Entity:** Strathcona 1 LLC (LLC)
2. **Entity:** Magnet Labs, Inc. (Corp)
3. **Property:** Strathcona Property, Clinton Township MI
4. **Bank Accounts:**
   - Chase5271 (Checking, linked to Strathcona 1 LLC)
   - Chase9268 (Credit Card, linked to Strathcona 1 LLC)
   - Chase0505 (Credit Card, linked to Magnet Labs — intercompany)
5. **Chart of Accounts:** Full category list (see Section 4)
6. **Categorization Rules:** All seed rules (see Section 6)
7. **Tenants:** Weideman, Naidu
8. **Leases:** Weideman (from Jan 2025, $1,620.50/mo), Naidu (from Aug 2025, $1,710/mo)
9. **Loan:** Strathcona 1 LLC → Magnet Labs (open, principal TBD, interest TBD)

---

## 10. Development Phases

### Phase 1 — Foundation (Backend Core + Data Model)
- Project scaffold: monorepo, Makefile, uv + npm, .env
- SQLAlchemy models for all entities
- Alembic migrations
- FastAPI app skeleton with all routers registered (empty handlers)
- Database seed script
- `make dev` command: starts uvicorn backend

**Deliverable:** `GET /entities` returns seed data. Database is queryable.

---

### Phase 2 — Transaction Import Engine
- CSV parsers: Chase Checking format + Chase Credit Card format
- Duplicate detection
- Auto-categorizer service with seed rules
- `POST /imports/upload` endpoint — full pipeline: parse → dedupe → auto-categorize → persist
- `GET /transactions` with filters
- `PUT /transactions/{id}` for manual re-categorization
- Plaid stub class (raises NotImplementedError)

**Deliverable:** Upload the real Chase5271 CSV. Transactions appear, ~80% auto-categorized.

---

### Phase 3 — Reporting Engine
- Monthly P&L report service + endpoint
- Annual P&L rollup
- Schedule E computation + Excel export
- Rent Roll report
- Intercompany Loan Ledger report

**Deliverable:** All 5 reports return correct data matching the source spreadsheet figures.

---

### Phase 4 — Frontend
- Vite + React + Tailwind scaffold
- API client layer (Axios)
- Dashboard page: KPI cards (YTD net income, rent collected, uncategorized transaction count)
- Transactions page: filterable table, inline category select, bulk categorize
- Import page: CSV drag-drop upload, import history, duplicate review
- Reports page: parameter picker + download buttons
- Tenants page: rent roll table, lease detail
- Loans page: advance table, running balance

**Deliverable:** Full local web app running at `localhost:5173`.

---

### Phase 5 — Polish & Historical Data Migration
- Import all 2025 CSV data from Chase5271, Chase9268, Chase0505
- Verify totals match source spreadsheet
- Add all 137 Magnet Labs loan advances to loan ledger
- Reconcile report outputs against spreadsheet figures
- Add Citizens Bank CSV parser (if format available)

**Deliverable:** System is live, replaces spreadsheet as source of truth for Strathcona 1 LLC.

---

## 11. Multi-Property Extension Path (v2)

The schema is already multi-property. To add a second property:
1. Add Entity + Property records via UI (no schema changes)
2. Add BankAccount records for new accounts
3. Tag transactions to the new property via `transaction.property_id`
4. All reports are already parameterized by `property_id`

No refactoring required. This is intentional.

---

## 12. Key Engineering Constraints

- **No external services required** to run the app (all local)
- **SQLite WAL mode** enabled — safe for concurrent reads during report generation
- **All amounts stored as INTEGER (cents)** to avoid floating-point errors — convert on display
- **All dates stored as ISO 8601 strings** in SQLite
- **No hardcoded fiscal logic** — all report date ranges are parameters
- **Category codes are stable identifiers** — never rename codes once seeded (names can change)
- **Plaid interface must be defined in Phase 2** even though implementation is empty — keeps the import pipeline's public API stable

---

## 13. Questions for Claude Code to Clarify Before Starting

Before generating the sprint plan, Claude Code should confirm:
1. Confirm Python version (recommend 3.11+) and whether `uv` is acceptable as package manager
2. Confirm OS (macOS assumed — affects Makefile and SQLite path defaults)
3. Confirm whether the frontend should use TypeScript (recommended yes)
4. Confirm whether to include a simple auth layer (recommend no for v1 local use)
5. Ask if any of the 2025 CSV files should be included as sample data for Phase 2 testing

---

*End of Architecture Plan. Feed this document to Claude Code with the prompt:  
"Read this architecture plan in full. Generate a detailed, sprint-by-sprint development plan with specific tasks, file names, and acceptance criteria for each phase. Ask any clarifying questions before starting."*
