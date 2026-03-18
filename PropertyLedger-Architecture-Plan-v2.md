# PropLedger — Property Accounting System
## Architecture Plan v2.0 — Comprehensive 2025 + 2026 Feature Set

**Entity:** Strathcona 1 LLC (expandable to multi-entity/multi-property)
**Supersedes:** PropLedger-Architecture-Plan.md (2025 baseline)
**Purpose:** Replace manual Excel-based property bookkeeping with a local, extensible accounting app that covers the full 2026 feature set while retaining all 2025 foundations.
**Source of Truth:** This document. Feed to Claude Code to generate a sprint-by-sprint development plan before writing any code.

---

## 1. What Changed: 2025 → 2026 Feature Delta

The 2026 spreadsheet introduces eight major new modules and meaningfully upgrades four existing ones. This is not incremental — it transforms the system from a transaction ledger with P&L into a full property investment accounting platform.

### New Modules (net new — do not exist in 2025 plan)
| 2026 Sheet | What It Adds |
|---|---|
| `Config` | Centralized fiscal config: year, property register, mortgage details, CoA mapping, **monthly budget** per line item |
| `Dashboard` | Executive KPI view: EGI, NOI, DSCR, Cash Flow, Magnet Labs receivable; monthly summary table; expense breakdown chart |
| `Balance Sheet` | Full balance sheet: current assets, fixed assets, liabilities (mortgage per property), owner's equity; prior vs. current year comparison; balance check |
| `YoY Comparison` | Prior year vs. current year P&L: $ change + % change for every line item |
| `Maint-CapEx` | Maintenance & Capital Expenditure log: Repair vs. CapEx classification, useful life, annual depreciation calc; feeds Tax Prep |
| `Mortgage Amort` | Full amortization schedule per property: monthly P&I split, annual interest (tax-deductible), year-end loan balance; feeds Balance Sheet |
| `NOI-CashFlow` | Investor-grade cash flow statement: Gross Income → EGI → NOI → Cash Flow After Debt; key metrics (DSCR, break-even occupancy, cash-on-cash) |
| `Ledger` | Unified transaction view: all bank accounts normalized, capacity 1,500 rows, with a **Property** column on every transaction |

### Upgraded Modules (extend 2025 design)
| Module | 2025 | 2026 Additions |
|---|---|---|
| `Mo-P&L` | Monthly actuals only | + Budget column, Variance $, Variance %, Property selector |
| `Rent Roll` | Tenant list + YTD collection | + Monthly ✓/✗ payment grid, occupancy rate, annualized income, Days to Expiry |
| `Tax Prep` | Schedule E line mapping | + Depreciation section (Cost Basis ÷ 27.5), CapEx add-to-basis, Tax Impact Estimate, Imputed Interest on Magnet Loan |
| `Magnet Loan` | Principal ledger only | + 6% interest rate, per-advance daily accrual, Days Outstanding, Accrued Interest, Total Outstanding Balance, Payments Received |

### Transaction-Level Change
Every bank transaction now carries a `property_id` tag. This is the key enabler for per-property reporting across all modules.

---

## 2. Technology Stack

Unchanged from v1 — all free/open source, no paid services required.

| Layer | Technology | Notes |
|---|---|---|
| Backend API | Python 3.11+ / FastAPI | |
| Database | SQLite (SQLAlchemy ORM) | Schema designed for Postgres migration |
| Migrations | Alembic | |
| Data Processing | Pandas | |
| Financial Math | Python `decimal` module | All money as integer cents in DB |
| Excel Export | openpyxl | |
| Frontend | React 18 + Vite + TailwindCSS | |
| Package Mgmt | uv (Python) + npm (JS) | |
| Dev Runner | Makefile | `make dev` starts everything |

---

## 3. Updated Project Structure

```
propledger/
├── Makefile
├── README.md
├── .env.example
│
├── backend/
│   ├── pyproject.toml
│   ├── alembic.ini
│   ├── migrations/versions/
│   ├── data/
│   │   ├── propledger.db
│   │   └── uploads/
│   └── app/
│       ├── main.py
│       ├── database.py
│       ├── config.py
│       │
│       ├── models/
│       │   ├── entity.py
│       │   ├── property.py            # Extended: purchase_price, land_value, cost_basis, units
│       │   ├── property_config.py     # NEW: fiscal year settings, budget entries
│       │   ├── bank_account.py
│       │   ├── transaction.py         # Extended: property_id (was nullable, now encouraged)
│       │   ├── category.py            # Extended: schedule_e_line, pl_line_name mapping
│       │   ├── tenant.py
│       │   ├── lease.py
│       │   ├── rent_payment.py        # Extended: status per month (paid/unpaid/partial)
│       │   ├── loan.py                # Extended: interest_rate, compounding, payments_received
│       │   ├── loan_advance.py        # Extended: days_outstanding, interest_accrued (computed)
│       │   ├── mortgage.py            # NEW: per-property mortgage record
│       │   ├── mortgage_payment.py    # NEW: monthly amortization row
│       │   ├── maintenance_item.py    # NEW: repair vs. capex log
│       │   └── balance_sheet.py       # NEW: year-end snapshot for prior-year column
│       │
│       ├── schemas/                   # Pydantic schemas mirror all models
│       │
│       ├── routers/
│       │   ├── entities.py
│       │   ├── properties.py          # Extended: now includes asset details
│       │   ├── bank_accounts.py
│       │   ├── transactions.py        # Extended: bulk property-tag endpoint
│       │   ├── imports.py
│       │   ├── tenants.py
│       │   ├── leases.py
│       │   ├── rent_payments.py
│       │   ├── loans.py               # Extended: interest summary endpoint
│       │   ├── mortgages.py           # NEW: mortgage CRUD + amortization
│       │   ├── maintenance.py         # NEW: maintenance/capex CRUD
│       │   ├── budgets.py             # NEW: monthly budget CRUD
│       │   └── reports.py             # Extended: 5 new report endpoints
│       │
│       └── services/
│           ├── csv_parser.py
│           ├── plaid_stub.py
│           ├── categorizer.py
│           ├── report_pl.py           # Extended: now includes budget comparison
│           ├── report_schedule_e.py   # Extended: depreciation, tax estimate
│           ├── report_rent_roll.py    # Extended: monthly collection grid
│           ├── report_loan.py         # Extended: interest accrual
│           ├── report_balance_sheet.py # NEW
│           ├── report_yoy.py          # NEW
│           ├── report_noi_cashflow.py  # NEW
│           ├── report_dashboard.py    # NEW
│           ├── depreciation.py        # NEW: building + capex depreciation engine
│           ├── mortgage_amort.py      # NEW: amortization schedule math
│           └── exporter.py
│
└── frontend/
    └── src/
        ├── pages/
        │   ├── Dashboard.tsx          # NEW: KPI tiles, monthly chart, expense donut
        │   ├── Transactions.tsx       # Extended: property tag column + bulk tagger
        │   ├── Import.tsx
        │   ├── Properties.tsx         # Extended: asset details, mortgage summary
        │   ├── Tenants.tsx            # Extended: monthly payment grid
        │   ├── Loans.tsx              # Extended: accrued interest display
        │   ├── Reports.tsx            # Extended: all new report types
        │   ├── BalanceSheet.tsx       # NEW
        │   ├── YoYComparison.tsx      # NEW
        │   ├── NOICashFlow.tsx        # NEW
        │   ├── Maintenance.tsx        # NEW: capex/repair log
        │   ├── MortgageAmort.tsx      # NEW
        │   └── TaxPrep.tsx            # NEW (replaces basic Schedule E page)
        └── components/
            ├── KPICard.tsx
            ├── MonthlyGrid.tsx        # Reusable month × item grid (rent collection, budgets)
            ├── PropertySelector.tsx
            ├── BudgetVarianceRow.tsx
            └── DepreciationSummary.tsx
```

---

## 4. Full Data Model

### 4.1 Existing Models — Extensions Only

**Property** (add to v1)
```
+ units (integer, default 1)
+ purchase_price (integer, cents)
+ land_value (integer, cents)
+ cost_basis_building (integer, cents)        # purchase_price - land_value
+ acquisition_date (date)
+ down_payment (integer, cents)
+ property_type (residential | commercial | multi_unit)
```

**Transaction** (add to v1)
```
+ property_id (FK to Property, nullable)      # was already there in v1, now actively used
```

**Category** (add to v1)
```
+ pl_line_name (string)    # "Rental Income", "Repairs & Maintenance", etc. — the display name
                           # One category can map to a shared P&L line (e.g. Magnet Labs R&D → Office Expense)
```

**Loan** (add to v1)
```
+ interest_rate (decimal, nullable)           # annual rate, e.g. 0.06
+ compounding (simple | compound)
+ maturity_date (date, nullable)
+ repayment_type (balloon | monthly | quarterly)
+ payments_received (integer, cents)
```

**LoanAdvance** (add to v1)
```
+ days_outstanding (integer, computed at query time)
+ interest_accrued (decimal, computed at query time: amount × rate/365 × days)
```

---

### 4.2 New Models

**FiscalConfig**
```
id, entity_id (FK), fiscal_year (integer),
created_at, updated_at

# One record per entity per fiscal year. Stores year-level config.
```

**Budget**
```
id, entity_id (FK), property_id (FK nullable),
fiscal_year (integer), pl_line_name (string),
monthly_amount (integer, cents),
notes (string)

# One row per P&L line per entity per year.
# Feeds Mo-P&L budget vs actual comparison.
```

**Mortgage**
```
id, property_id (FK),
original_loan (integer, cents),
interest_rate (decimal),               # e.g. 0.07
term_months (integer),
origination_date (date),
monthly_payment (decimal),             # auto-computed: standard amortization formula
balance_at_year_start (integer, cents), # manual entry or carried forward
is_active (bool)
```

**MortgagePayment** (amortization rows — computed, stored for performance)
```
id, mortgage_id (FK),
payment_number (integer),
payment_date (date),
payment_amount (decimal),
principal (decimal),
interest (decimal),
remaining_balance (decimal),
fiscal_year (integer)
```

**MaintenanceItem**
```
id, property_id (FK),
transaction_id (FK nullable),          # links to the source bank transaction
item_date (date),
description (string),
cost (integer, cents),
item_type (repair | capex | de_minimis),
vendor (string, nullable),
status (pending | in_progress | completed),
useful_life_years (decimal, nullable), # only for capex
annual_depreciation (decimal),         # cost / useful_life; 0 for repairs
notes (string)

# item_type = repair → immediate Schedule E deduction (Line 14)
# item_type = capex  → added to cost basis, depreciated on separate schedule
# item_type = de_minimis → ≤$2,500 safe harbor, expensed immediately
```

**BalanceSheetSnapshot** (stores prior-year starting balances)
```
id, entity_id (FK), snapshot_date (date),
cash_and_bank (integer, cents),
accounts_receivable_tenants (integer, cents),
prepaid_expenses (integer, cents),
security_deposits_held (integer, cents),
accounts_payable (integer, cents),
credit_card_balance (integer, cents),
other_current_liabilities (integer, cents),
other_long_term_debt (integer, cents),
owner_capital_contributions (integer, cents),
additional_paid_in_capital (integer, cents),
retained_earnings_prior (integer, cents),
owner_draws (integer, cents),
accumulated_depreciation_prior (integer, cents),
notes (string)

# The "editable blue cells" in the spreadsheet.
# Current-year computed values (net income, mortgage balance, etc.) are never stored —
# they are always derived fresh from transactions and mortgage schedule.
```

---

## 5. Reporting Layer — Full Suite

All reports computed server-side from source tables. No stale pre-aggregates.

### Report 1: Monthly P&L (Enhanced)
**Endpoint:** `GET /reports/pl?entity_id=&property_id=&month=&year=`
**New vs 2025:** Adds `Budget`, `Variance $`, `Variance %` columns alongside `Actual`

Logic:
- Actual: SUM of transactions filtered by `property_id`, `category.pl_line_name`, date range
- Budget: look up `Budget` table for same entity/property/year/line
- Variance: Actual - Budget (favorable = income over / expense under)
- Variance %: Variance / Budget (guard against divide-by-zero)

---

### Report 2: Annual P&L Rollup
**Endpoint:** `GET /reports/pl/annual?entity_id=&property_id=&year=`
Unchanged from v1 — 12-column monthly rollup + YTD total.

---

### Report 3: Year-over-Year Comparison (New)
**Endpoint:** `GET /reports/yoy?entity_id=&property_id=&current_year=&prior_year=`
**Output:** Side-by-side P&L: Prior Year | Current Year | $ Change | % Change

Logic:
- Pull annual P&L totals for both years from Transaction table
- Prior year (2025) data is stored as historical transactions — same query, different year filter
- Key Metrics section: OER and NOI Margin for each year

---

### Report 4: Rent Roll (Enhanced)
**Endpoint:** `GET /reports/rent-roll?property_id=&fiscal_year=`
**New vs 2025:** Monthly ✓/✗ collection grid per tenant + summary metrics

Output sections:
1. Property Details (unit count, type)
2. Tenant Register (address, rent, lease dates, deposit, status, days to expiry)
3. Rent Roll Summary (total scheduled monthly rent, occupancy rate, annualized income, avg rent per unit)
4. Monthly Collection Grid: rows = tenants, columns = Jan–Dec, cell = Paid | Unpaid | Partial
   - Driven by `RentPayment.status` per `(tenant_id, period_month, period_year)`
5. Collection Rate per month (% of expected rent actually received)

---

### Report 5: Schedule E / Tax Prep (Enhanced)
**Endpoint:** `GET /reports/tax-prep?entity_id=&property_id=&year=`
**New vs 2025:** Full depreciation section + tax impact estimate + imputed interest

New sections:
- **Depreciation:** Pull `property.cost_basis_building` from Config; divide by 27.5; add `MaintenanceItem` CapEx depreciation
- **Capital Improvements:** Sum of completed CapEx from `MaintenanceItem` table
- **Schedule E Net Income:** Total Income - Total Expenses - Annual Depreciation
- **Tax Impact Estimate:** User inputs marginal rate; system computes tax on income, savings from depreciation
- **Imputed Interest (Magnet Loan):** Outstanding balance × AFR rate → taxable income estimate

---

### Report 6: Intercompany Loan Ledger (Enhanced)
**Endpoint:** `GET /reports/loan?loan_id=&as_of_date=`
**New vs 2025:** Interest accrual per advance

For each advance:
```
interest_accrued = advance_amount × (annual_rate / 365) × days_outstanding
days_outstanding = (as_of_date - advance_date).days
```

Summary:
- Total Principal Advanced
- Total Accrued Interest
- Payments Received
- **Total Outstanding Balance** = Principal + Accrued Interest - Payments

---

### Report 7: Mortgage Amortization Schedule (New)
**Endpoint:** `GET /reports/mortgage-amort?mortgage_id=&fiscal_year=`

Logic:
- Standard amortization formula: `M = P × [r(1+r)^n] / [(1+r)^n - 1]`
- Compute all payments from origination date forward
- Filter to fiscal year requested
- Output: 12-row monthly table (Payment #, Date, Payment, Principal, Interest, Balance)
- Summary: Balance at Year Start, Balance at Year End, Principal Paid This Year, **Annual Interest Paid (tax-deductible)**
- Annual interest feeds Tax Prep Schedule E Line 12

---

### Report 8: Balance Sheet (New)
**Endpoint:** `GET /reports/balance-sheet?entity_id=&as_of_year=`

Structure:
```
ASSETS
  Current Assets
    Cash & Bank Accounts          → sum of known bank balances (or manual starting balance + net income)
    Accounts Receivable — Tenants → sum of RentPayment status=expected but not received
    Intercompany Receivable       → LoanLedger total outstanding balance
    Prepaid Expenses              → from BalanceSheetSnapshot (manual)
    Total Current Assets

  Fixed Assets
    Land                          → SUM of property.land_value
    Buildings (at cost)           → SUM of property.cost_basis_building
    + Capital Improvements        → SUM of MaintenanceItem where type=capex and status=completed
    − Accumulated Depreciation    → Prior accum (snapshot) + current year depreciation
    Net Fixed Assets

  TOTAL ASSETS

LIABILITIES
  Current Liabilities
    Accounts Payable              → manual (snapshot)
    Credit Card Balance           → manual (snapshot)
    Security Deposits Held        → SUM of lease.security_deposit where active
    Other Current Liabilities     → manual
    Total Current Liabilities

  Long-Term Liabilities
    Mortgage — [Property Name]    → MortgagePayment.remaining_balance at year-end (one row per property)
    Other Long-Term Debt          → manual
    Total Long-Term Liabilities

  TOTAL LIABILITIES

OWNER'S EQUITY
  Owner's Capital Contributions   → SUM of property.down_payment
  Additional Paid-In Capital      → manual
  Retained Earnings (Prior Years) → manual (snapshot) — rolls forward at year-end
  Current Year Net Income         → from Annual P&L report
  Owner's Draws                   → manual
  TOTAL OWNER'S EQUITY

BALANCE CHECK: Total Assets − Total Liabilities − Total Equity (should = 0)
```

Two-column output: `Prior Year End` (from BalanceSheetSnapshot) | `Current Year End` (computed) | `Change`

---

### Report 9: NOI & Cash Flow Statement (New)
**Endpoint:** `GET /reports/noi-cashflow?entity_id=&property_id=&year=`

Structure:
```
GROSS RENTAL INCOME
  Scheduled Rental Income        → SUM(RentPayment.amount where status=received)
  Other Income / Refunds
  Gross Potential Income
  Less: Vacancy & Credit Loss    → Expected - Actual (from Rent Roll)
  EFFECTIVE GROSS INCOME (EGI)

OPERATING EXPENSES (with % of EGI column)
  [All operating expense categories]
  Total Operating Expenses
  OER = Total OpEx / EGI

NET OPERATING INCOME (NOI)
  NOI Margin = NOI / EGI

DEBT SERVICE
  Mortgage / Loan Payments
  Credit Card Payments
  Total Debt Service

CASH FLOW AFTER DEBT SERVICE

KEY METRICS
  DSCR = NOI / Total Debt Service
  Break-Even Occupancy = Total Expenses / Gross Potential Income
  Cash-on-Cash Return = Cash Flow / (SUM of down payments) — requires Config data
  Cap Rate = NOI / (SUM of purchase prices) — memo item

CAPITAL EXPENDITURES (Memo)
  CapEx Spend (Completed)        → from MaintenanceItem
  Cash Flow After CapEx

MORTGAGE BREAKDOWN
  Annual Interest (tax-deductible) → from MortgageAmort
  Annual Principal (not deductible)
```

---

### Report 10: Dashboard KPIs (New)
**Endpoint:** `GET /reports/dashboard?entity_id=&property_id=&year=`

Returns JSON for dashboard UI:
```json
{
  "kpis": {
    "egi": 40038,
    "noi": 18000,
    "cash_flow_after_debt": 12000,
    "noi_margin": 0.45,
    "dscr": 2.16,
    "magnet_labs_receivable": 9684
  },
  "monthly_summary": [
    {"month": "Jan", "income": 3336, "expenses": 698, "noi": 2638, "debt_svc": 1000, "cash_flow": 1638},
    ...
  ],
  "expense_breakdown": [
    {"category": "Property Taxes", "amount": 1046},
    ...
  ]
}
```

---

### Report 11: Maintenance & CapEx Log (New)
**Endpoint:** `GET /reports/maint-capex?property_id=&year=`

Sections:
1. Detail log (all items, sortable by date/property/type/status)
2. Repairs summary: count, total cost → feeds Schedule E Line 14
3. CapEx summary: count, total spend, total annual depreciation → feeds Schedule E Line 18
4. Total maintenance spend

---

## 6. Full API Endpoint Reference

```
# Existing from v1 — unchanged
GET/POST/PUT/DELETE  /entities
GET/POST/PUT/DELETE  /properties
GET/POST/PUT/DELETE  /bank-accounts
GET/POST/PUT/DELETE  /transactions
POST                 /transactions/bulk-categorize
POST                 /transactions/re-categorize
POST                 /imports/upload
GET                  /imports, /imports/{id}
GET/POST/PUT/DELETE  /categorization-rules
GET/POST/PUT/DELETE  /tenants
GET/POST/PUT/DELETE  /leases
GET/POST/PUT/DELETE  /rent-payments
POST                 /rent-payments/match-transaction
GET/POST/PUT/DELETE  /loans
GET                  /loans/{id}/advances

# New / Extended in v2

## Transactions (extended)
POST  /transactions/bulk-tag-property    # [{id, property_id}] — bulk assign property

## Mortgages (new)
GET/POST             /mortgages
GET/PUT/DELETE       /mortgages/{id}
GET                  /mortgages/{id}/amortization?year=   # full schedule for year
POST                 /mortgages/{id}/recalculate           # regenerate MortgagePayment rows

## Budgets (new)
GET/POST             /budgets
GET/PUT/DELETE       /budgets/{id}
GET                  /budgets?entity_id=&year=             # all budget lines for a year
POST                 /budgets/bulk-upsert                  # [{pl_line_name, monthly_amount}]

## Maintenance & CapEx (new)
GET/POST             /maintenance
GET/PUT/DELETE       /maintenance/{id}
POST                 /maintenance/{id}/link-transaction     # link to source transaction
GET                  /maintenance/summary?property_id=&year=  # feeds Tax Prep

## Balance Sheet Snapshots (new)
GET/POST             /balance-sheet-snapshots
GET/PUT              /balance-sheet-snapshots/{id}
POST                 /balance-sheet-snapshots/year-end-rollover  # copies year-end to new prior-year snapshot

## Reports (new/extended)
GET  /reports/pl?entity_id=&property_id=&month=&year=          # Extended: +budget columns
GET  /reports/pl/annual?entity_id=&property_id=&year=
GET  /reports/yoy?entity_id=&property_id=&current_year=&prior_year=  # NEW
GET  /reports/rent-roll?property_id=&fiscal_year=              # Extended: +monthly grid
GET  /reports/tax-prep?entity_id=&property_id=&year=           # Extended: +depreciation
GET  /reports/loan?loan_id=&as_of_date=                        # Extended: +interest
GET  /reports/mortgage-amort?mortgage_id=&fiscal_year=         # NEW
GET  /reports/balance-sheet?entity_id=&as_of_year=             # NEW
GET  /reports/noi-cashflow?entity_id=&property_id=&year=       # NEW
GET  /reports/dashboard?entity_id=&property_id=&year=          # NEW
GET  /reports/maint-capex?property_id=&year=                   # NEW

# All reports support ?format=json (default) or ?format=xlsx for Excel download
```

---

## 7. Depreciation Engine

All depreciation math lives in `services/depreciation.py`.

### Building Depreciation (Straight-Line, 27.5 years)
```python
def annual_building_depreciation(property: Property) -> Decimal:
    """
    IRS MACRS residential real property: cost basis / 27.5
    cost basis = purchase_price - land_value
    """
    return Decimal(property.cost_basis_building) / Decimal('27.5')

def accumulated_depreciation(property: Property, years_held: int) -> Decimal:
    """Prior years' depreciation for Balance Sheet."""
    return annual_building_depreciation(property) * years_held
```

### CapEx Depreciation
```python
def capex_annual_depreciation(item: MaintenanceItem) -> Decimal:
    """Capital improvements depreciated over their useful life."""
    if item.item_type != 'capex' or not item.useful_life_years:
        return Decimal('0')
    return Decimal(item.cost) / Decimal(str(item.useful_life_years))
```

### Schedule E Line 18 (Total Depreciation)
```python
def schedule_e_depreciation(property_id, fiscal_year) -> Decimal:
    building_depr = annual_building_depreciation(property)
    capex_depr = sum(capex_annual_depreciation(item) for item in completed_capex_items)
    return building_depr + capex_depr
```

---

## 8. Mortgage Amortization Engine

Lives in `services/mortgage_amort.py`.

```python
def compute_monthly_payment(principal: Decimal, annual_rate: Decimal, term_months: int) -> Decimal:
    """Standard mortgage payment formula: M = P × [r(1+r)^n] / [(1+r)^n - 1]"""
    r = annual_rate / 12
    return principal * (r * (1 + r)**term_months) / ((1 + r)**term_months - 1)

def generate_amortization_schedule(mortgage: Mortgage) -> list[MortgagePayment]:
    """
    Generate all payment rows from origination to term end.
    Returns full schedule; filter to fiscal_year in the router.
    Stores results in MortgagePayment table for performance.
    """
    ...

def fiscal_year_summary(mortgage_id, fiscal_year) -> dict:
    """Returns: balance_start, balance_end, principal_paid, interest_paid"""
    ...
```

---

## 9. Budget vs. Actual Engine

Lives in `services/report_pl.py` (extended).

```python
def monthly_pl_with_budget(entity_id, property_id, month, year) -> dict:
    """
    Returns per-line-item dict with:
    - actual: SUM of transactions in period
    - budget: from Budget table
    - variance_dollars: actual - budget
    - variance_pct: variance / budget (None if budget=0)
    - favorable: bool (income: actual > budget; expense: actual < budget)
    """
```

Budget lookup: `Budget` table keyed by `(entity_id, property_id, fiscal_year, pl_line_name)`.
If no budget exists for a line, variance fields return `None` — not zero.

---

## 10. Loan Interest Accrual Engine

**Simple interest model** (matching spreadsheet):
```python
def accrue_interest(advance: LoanAdvance, as_of_date: date, annual_rate: Decimal) -> Decimal:
    """
    Per-advance simple interest accrual from disbursement date.
    interest = principal × (rate / 365) × days_outstanding
    """
    days = (as_of_date - advance.advance_date).days
    return advance.amount * (annual_rate / 365) * days

def loan_outstanding_balance(loan: Loan, as_of_date: date) -> dict:
    """Returns: principal, accrued_interest, payments_received, total_outstanding"""
    principal = sum(a.amount for a in loan.advances)
    interest = sum(accrue_interest(a, as_of_date, loan.interest_rate) for a in loan.advances)
    return {
        "principal": principal,
        "accrued_interest": interest,
        "payments_received": loan.payments_received,
        "total_outstanding": principal + interest - loan.payments_received
    }
```

The Magnet Labs receivable on the Dashboard and Balance Sheet both call `loan_outstanding_balance()`.

---

## 11. Updated Seed Data

On `make seed`, populate everything from v1 PLUS:

**Mortgages:**
- Prop-1: $0 loan (paid off / no active mortgage per spreadsheet)
- Prop-2 (Weideman): $60,000 original, 7% rate, 10-year term, origination 2022-03-07 → monthly payment $696.65

**Property Asset Details:**
- Prop-1 (Strathcona): land $10,000, building cost basis $133,000
- Prop-2 (Weideman): purchase $156,000, land $10,000, building cost basis $146,000

**Budgets (2026 monthly):**
- Rental Income: $1,710
- Repairs & Maintenance: $100
- Insurance: $90
- Bank Fees: $15
- Office Expense: $50
- Meals & Entertainment: $50
- Licenses & Fees: $25
- Property Taxes: $200
- Other: $25
- Loan Payments: $400
- Credit Card Payments: $75

**BalanceSheetSnapshot (Dec 31, 2025 starting balances):**
- Accumulated Depreciation Prior: computed from 2021–2025 building depreciation
- Retained Earnings Prior: 0 (first year of tracking)
- Cash: 0 (simplified starting position)

**FiscalConfig:**
- Fiscal Year: 2026
- Entity: Strathcona 1 LLC

---

## 12. Updated Frontend Pages

### Dashboard (New — Landing Page)
- Six KPI tiles: EGI, NOI, Cash Flow After Debt, NOI Margin, DSCR, Magnet Labs Receivable
- Monthly summary table (12 rows: Jan–Dec) with Income, Expenses, NOI, Debt Service, Cash Flow
- Expense breakdown donut chart (top 8 categories by spend)
- Property selector (All or specific property)
- "as of" indicator showing data through latest imported month

### Transactions (Extended)
- Add "Property" column with inline dropdown (Prop-1, Prop-2, All, Untagged)
- Bulk property tagger: select rows → assign property
- Filter by property
- Visual indicator for untagged transactions (needed for per-property reports to be accurate)

### Properties (Extended)
- Asset details tab: purchase price, land value, cost basis, acquisition date
- Mortgage summary card: current balance, next payment, months remaining
- Quick link to full amortization schedule

### Tenants (Extended)
- Monthly payment grid (12-month × tenant matrix)
- Click cell to mark paid/unpaid/partial
- Collection rate per month shown at bottom
- Days to lease expiry badge (amber <60 days, red <30 days)

### Maintenance & CapEx (New Page)
- Add item form: Property, Date, Description, Cost, Type (Repair/CapEx/De Minimis), Vendor, Status, Useful Life
- Table with filter by property, type, status, date range
- Link to transaction button (opens transaction picker modal)
- Summary cards: Total Repairs YTD, Total CapEx YTD, Annual Depreciation from CapEx

### Mortgage Amortization (New Page)
- Mortgage selector per property
- Summary: Original Loan, Rate, Term, Monthly Payment, Balance at Year Start/End
- Annual summary: Principal Paid, Interest Paid (tax-deductible)
- Full 12-month table for selected fiscal year
- Note: "Annual interest figure appears on Tax Prep Schedule E Line 12"

### Balance Sheet (New Page)
- Two-column layout: Prior Year End | Current Year End | Change
- Expandable sections: Current Assets, Fixed Assets, Liabilities, Owner's Equity
- Balance check indicator (green ✓ Balanced / red ⚠ Out of Balance with difference amount)
- Editable prior-year starting balances (blue cells in spreadsheet → inline edit)
- Year-End Rollover button: copies current year computed values to next year's starting balances

### Year-over-Year Comparison (New Page)
- Side-by-side P&L: Prior Year | Current Year | $ Change | % Change
- Year pickers for both columns
- Color coding: green = improvement, red = deterioration
- Key Metrics block below table

### NOI & Cash Flow (New Page)
- Single-column statement layout (income statement style)
- EGI waterfall: Gross Income → vacancy loss → EGI
- Operating Expenses with % of EGI column
- NOI highlighted prominently
- Key Metrics panel: DSCR, OER, Break-Even Occupancy, Cash-on-Cash Return
- CapEx memo section
- Mortgage P/I breakdown

### Tax Prep (New Page — replaces basic Schedule E)
- Property selector (Schedule E is per-property by IRS rules)
- Property Identification section (address, type, fair rental days)
- Income section (auto-pulled)
- Expenses section mapped to Schedule E lines (auto-pulled)
- Depreciation section (auto-computed from Config, editable prior accum)
- Schedule E Net Income/Loss (prominently displayed)
- Tax Impact Estimate panel: editable marginal rate, computed tax/savings
- Imputed Interest section (Magnet Loan)
- Print/Export to Excel button

---

## 13. Development Phases (Extended from v1)

### Phase 1 — Foundation (Unchanged from v1)
Backend scaffold, all models migrated, seed data, FastAPI skeleton, `make dev` command.

### Phase 2 — Transaction Import Engine (Unchanged from v1)
CSV parsers, duplicate detection, auto-categorizer, Plaid stub. Upload Chase CSVs, transactions appear and auto-categorize.

**Phase 2 addition:** Property column parsing — the 2026 bank CSVs have a `Property` column. Parser must extract and persist it.

### Phase 3 — Core Reporting (Extended)
Implement all report services:
- Monthly P&L (with budget/variance)
- Annual P&L rollup
- Rent Roll (with monthly grid)
- Tax Prep / Schedule E (with depreciation)
- Loan Ledger (with interest accrual)
- Dashboard KPIs

Verify outputs against spreadsheet figures for Jan 2026 and Feb 2026.

### Phase 4 — New Financial Modules
Build the three new financial engines:
- Mortgage Amortization: model, service, endpoint, amortization schedule generation
- Depreciation Engine: building + capex
- Balance Sheet: model, snapshot management, computed report
- YoY Comparison: report service + endpoint
- NOI/Cash Flow: report service + endpoint

Verify mortgage balance ($36,639.49 end of 2026), depreciation ($10,145.45/year), and balance sheet totals against spreadsheet.

### Phase 5 — Frontend (Extended)
All pages from v1 PLUS:
- Dashboard (new landing page)
- Maintenance/CapEx page
- Mortgage Amortization page
- Balance Sheet page
- YoY Comparison page
- NOI/Cash Flow page
- Tax Prep page (full version)
- Extended Transactions page (property tag column)
- Extended Tenants page (monthly grid)

### Phase 6 — Budget System
- Budget CRUD API and frontend
- Config page: set monthly budgets per line item
- Mo-P&L page: actual vs. budget with variance columns
- Seed 2026 budgets from Config tab values

### Phase 7 — Historical Data Migration & Reconciliation
- Import all 2025 data (Chase5271, Chase9268, Chase0505 from prior year CSVs)
- Import all 2026 data through current month
- Tag transactions to properties (bulk tagger for historical cleanup)
- Enter CapEx items from Maint-CapEx log
- Set prior-year balance sheet snapshot values
- Verify all report outputs against both spreadsheets

---

## 14. Key Engineering Notes

### Property Tagging is Load-Bearing
The entire per-property reporting layer depends on `transaction.property_id`. Transactions without a property tag are excluded from property-specific P&L, Rent Roll collection matching, and Balance Sheet. The UI should make untagged transaction counts visible and easy to resolve.

### Depreciation Math Is Deterministic — No Storage Needed
Building depreciation (`cost_basis / 27.5`) and CapEx depreciation (`cost / useful_life`) are pure functions with no state. Compute them fresh at report time from `Property` and `MaintenanceItem` tables. The only stored value is `BalanceSheetSnapshot.accumulated_depreciation_prior` for the prior-year column.

### Mortgage Schedule Storage Pattern
Mortgage amortization rows are computationally cheap but queried frequently. Generate and store all `MortgagePayment` rows when a mortgage is created or when `POST /mortgages/{id}/recalculate` is called. This makes balance sheet and tax prep queries simple joins rather than re-computing the schedule on every request.

### Balance Sheet Relies on Snapshot Pattern
The prior-year Balance Sheet column cannot be computed from transactions alone (it requires starting balances that predate the system). The `BalanceSheetSnapshot` table stores these manual "blue cell" values. At fiscal year-end, `POST /balance-sheet-snapshots/year-end-rollover` copies the current year's computed ending balances into the next year's snapshot, automating the annual rollover.

### Interest Accrual Is Date-Sensitive
Loan interest changes every day. Never cache accrued interest — always compute it fresh with `as_of_date = today` (or a user-specified date for historical reports). The loan balance shown on the Dashboard, Balance Sheet, and Tax Prep must all call the same `loan_outstanding_balance()` function to stay consistent.

### Amounts — Integer Cents Throughout
All dollar values: stored as integers (cents) in the database, converted to/from decimal on API boundaries. All financial math uses Python's `Decimal` type with `ROUND_HALF_UP` mode. No `float` arithmetic anywhere in the financial layer.

### No Hardcoded Fiscal Logic
All reports accept `year` as a parameter. The system can report on 2024, 2025, or 2026 from the same codebase. The only fiscal-year-specific data is `Budget` (entered per year) and `BalanceSheetSnapshot` (stored per year-end date).

---

## 15. Questions for Claude Code Before Starting

1. Confirm Python 3.11+ and `uv` as package manager
2. Confirm OS (macOS assumed — affects Makefile defaults)
3. Confirm TypeScript for frontend (recommended yes)
4. Confirm no auth layer for v1 local use
5. Confirm whether to start Phase 1 fresh or incorporate the v1 plan's existing scaffold (if any code was already written)
6. Confirm which report to implement first in Phase 3 — recommend Dashboard → Mo-P&L (highest daily use value)
7. Ask whether the Weideman property mortgage origination date of 2022-03-07 should be used to pre-seed historical amortization rows back to 2022, or only generate rows from the current fiscal year forward

---

## 16. Scope Boundary — What Is NOT in v2

- Multi-user / role-based access (single user, local app)
- Double-entry bookkeeping / journal entries
- Invoice or bill management
- Document/lease file storage (metadata only)
- Payroll
- Cash flow forecasting / projections
- Plaid live sync (interface stubbed, not wired)
- Mobile app

These are v3+ scope items.

---

*End of Architecture Plan v2.0. Feed this document to Claude Code with the prompt:*
*"Read this architecture plan in full. Compare it to the v1 plan (PropLedger-Architecture-Plan.md) if available, then generate a detailed sprint-by-sprint development plan with specific tasks, file names, and acceptance criteria for each phase. Start with any clarifying questions before producing the plan."*
