# PropLedger — Sprint Development Plan
## Strathcona Accounting System Foundation

**Based on:** PropertyLedger-Architecture-Plan-v2.md (supersedes v1)
**Architecture Principles:**
- **No hardcoded options or lists** — all type classifications are database-driven lookup tables
- **All settings configurable** — centralized SystemSetting model + FiscalConfig
- **Expandable foundation** — schema-ready for double-entry bookkeeping, multi-currency, multi-entity
- **Sprint cadence:** 2-week sprints

---

## Architectural Enhancements Beyond v2 Plan

Before detailing sprints, these foundational design decisions affect every layer and differ from what the v2 plan specifies:

### 1. Database-Driven Lookup Tables (replaces all enums)

Instead of hardcoded Python enums or string validation for type fields, ALL classification values live in the database:

```
LookupType
├── id (PK)
├── code (unique)          # e.g. "property_type", "account_type", "transaction_type"
├── name                   # Human-readable: "Property Type"
├── description
├── is_system (bool)       # System-managed types can't be deleted
└── created_at

LookupValue
├── id (PK)
├── lookup_type_id (FK → LookupType)
├── code (unique per type)  # e.g. "residential", "commercial", "multi_unit"
├── name                    # Display name: "Residential"
├── display_order (int)     # Controls dropdown ordering
├── is_active (bool)        # Soft-disable without deletion
├── is_default (bool)       # Default selection in UI
├── is_system (bool)        # Protects seed values from deletion
├── metadata (JSON)         # Extensible per-type metadata
├── parent_value_id (FK → self, nullable)  # Hierarchical values
└── created_at
```

**Lookup types seeded at install:**

| LookupType code | Seed values |
|---|---|
| `entity_type` | LLC, Corp, Individual, Partnership, Trust |
| `property_type` | residential, commercial, multi_unit, mixed_use |
| `bank_account_type` | checking, credit_card, savings, money_market |
| `transaction_type` | debit, credit, payment, fee, transfer, adjustment |
| `category_type` | income, operating_expense, debt_service, capital, transfer, contra, equity, liability |
| `lease_type` | fixed, month_to_month, commercial_net, commercial_gross |
| `rent_payment_status` | expected, received, partial, late, waived |
| `maintenance_item_type` | repair, capex, de_minimis |
| `maintenance_status` | pending, in_progress, completed, cancelled |
| `loan_compounding` | simple, compound |
| `loan_repayment_type` | balloon, monthly, quarterly, annual |
| `import_source` | chase_checking_csv, chase_credit_csv, citizens_csv, plaid, manual, other |
| `schedule_e_line` | line_3, line_5, line_6, ... line_22 |
| `depreciation_method` | straight_line, declining_balance, sum_of_years |
| `fiscal_period` | calendar_year, fiscal_q1_start_apr, fiscal_q1_start_jul, fiscal_q1_start_oct |

**Impact:** Every dropdown, filter, and type check in the system queries LookupValue. Adding a new property type, account type, or expense category requires zero code changes — just an API call or UI interaction.

### 2. Configurable System Settings

```
SystemSetting
├── id (PK)
├── key (unique)            # Dot-notation: "fiscal.depreciation_years_residential"
├── value (text)            # Stored as text, cast by value_type
├── value_type              # string | integer | decimal | boolean | json
├── category                # general | fiscal | display | import | tax
├── label                   # UI display label
├── description             # Help text
├── is_readonly (bool)      # Some settings are display-only
├── validation_rule (JSON)  # Optional: {"min": 0, "max": 100} or {"options": [...]}
└── updated_at
```

**Seed settings:**

| Key | Default | Purpose |
|---|---|---|
| `general.default_entity_id` | (seed entity id) | Pre-select entity in forms |
| `general.application_name` | "PropLedger" | Branding |
| `fiscal.current_year` | 2026 | Active fiscal year |
| `fiscal.year_start_month` | 1 | Calendar year = Jan; configurable for non-calendar fiscal years |
| `fiscal.depreciation_years_residential` | 27.5 | IRS MACRS residential |
| `fiscal.depreciation_years_commercial` | 39 | IRS MACRS commercial |
| `fiscal.de_minimis_threshold_cents` | 250000 | $2,500 safe harbor (stored as cents) |
| `fiscal.default_depreciation_method` | "straight_line" | References LookupValue code |
| `tax.default_marginal_rate` | 0.24 | For tax impact estimates |
| `tax.afr_rate` | 0.05 | Applicable Federal Rate for imputed interest |
| `import.duplicate_detection_enabled` | true | Toggle duplicate checking |
| `import.auto_categorize_on_import` | true | Auto-run categorizer |
| `import.default_property_id` | null | Auto-assign property on import |
| `display.date_format` | "MM/DD/YYYY" | UI date display |
| `display.currency_symbol` | "$" | For future multi-currency |
| `display.currency_code` | "USD" | ISO 4217 |
| `display.decimal_places` | 2 | Display precision |

### 3. Schema-Ready for Double-Entry (not implemented)

The Transaction model gets a nullable `journal_entry_id` FK. Two placeholder models are created but unused in v2:

```
JournalEntry (placeholder — no business logic in v2)
├── id (PK)
├── entry_date
├── reference_number
├── description
├── entity_id (FK)
├── is_posted (bool)
├── is_adjusting (bool)
├── source (manual | auto | import | closing)
├── created_at

JournalLine (placeholder — no business logic in v2)
├── id (PK)
├── journal_entry_id (FK)
├── account_id (FK → Category/Account)
├── debit_amount (integer, cents)
├── credit_amount (integer, cents)
├── description
├── property_id (FK, nullable)
```

This means: when double-entry is implemented in a future version, the schema already supports it. No migration needed — just activate the models and add business logic.

### 4. Hierarchical Chart of Accounts

The v2 `Category` model is renamed to `Account` and enhanced:

```
Account (replaces Category)
├── id (PK)
├── code (unique)                    # "4000", "5100", etc.
├── name                             # "Rental Income"
├── parent_id (FK → self, nullable)  # Supports unlimited nesting
├── account_type_id (FK → LookupValue where type='category_type')
├── normal_balance                   # 'debit' | 'credit' — for future double-entry
├── schedule_e_line                  # nullable, maps to IRS line
├── pl_line_name                     # P&L display grouping
├── is_system (bool)                 # Protects built-in accounts
├── is_active (bool)                 # Soft-disable
├── display_order (int)              # Controls report ordering
├── description
├── created_at
```

**Standard CoA seed (numbered for expandability):**

| Code | Name | Type | Sched E |
|---|---|---|---|
| 1000 | Assets | — | — |
| 1100 | Cash & Bank Accounts | asset | — |
| 1200 | Accounts Receivable - Tenants | asset | — |
| 1300 | Intercompany Receivable | asset | — |
| 1400 | Prepaid Expenses | asset | — |
| 1500 | Land | asset | — |
| 1600 | Buildings (at cost) | asset | — |
| 1610 | Capital Improvements | asset | — |
| 1650 | Accumulated Depreciation | contra | — |
| 2000 | Liabilities | — | — |
| 2100 | Accounts Payable | liability | — |
| 2200 | Credit Card Balance | liability | — |
| 2300 | Security Deposits Held | liability | — |
| 2400 | Mortgage Payable | liability | — |
| 2500 | Other Long-Term Debt | liability | — |
| 3000 | Owner's Equity | — | — |
| 3100 | Owner's Capital Contributions | equity | — |
| 3200 | Additional Paid-In Capital | equity | — |
| 3300 | Retained Earnings | equity | — |
| 3400 | Owner's Draws | equity | — |
| 4000 | Income | — | — |
| 4100 | Rental Income | income | line_3 |
| 4200 | Refunds & Credits | income | — |
| 4300 | Other Income | income | — |
| 5000 | Operating Expenses | — | — |
| 5100 | Repairs & Maintenance | operating_expense | line_14 |
| 5200 | Utilities | operating_expense | line_17 |
| 5300 | Insurance | operating_expense | line_9 |
| 5400 | Bank Fees | operating_expense | line_22 |
| 5500 | Office Expense | operating_expense | line_22 |
| 5600 | Meals & Entertainment | operating_expense | — |
| 5700 | Licenses & Fees | operating_expense | line_22 |
| 5800 | Property Taxes | operating_expense | line_16 |
| 5900 | Professional Fees | operating_expense | line_11 |
| 5950 | Other Expense | operating_expense | line_22 |
| 6000 | Debt Service | — | — |
| 6100 | Loan Payments | debt_service | line_12 |
| 6200 | Credit Card Payments | debt_service | — |
| 7000 | Intercompany | — | — |
| 7100 | Magnet Labs - R&D | transfer | — |
| 7200 | Magnet Labs - Travel | transfer | — |
| 7300 | Magnet Labs - SaaS | transfer | — |
| 8000 | Capital | — | — |
| 8100 | Capital Purchase | capital | — |
| 9000 | Depreciation | — | — |
| 9100 | Depreciation - Buildings | operating_expense | line_18 |
| 9200 | Depreciation - Improvements | operating_expense | line_18 |

### 5. Plugin-Ready Report Registry

Reports are registered in a service registry rather than hardcoded in routes:

```python
# services/report_registry.py
class ReportDefinition:
    code: str           # "monthly_pl", "balance_sheet", etc.
    name: str           # "Monthly P&L"
    category: str       # "financial", "tax", "operational"
    service_class: type  # The service class that generates the report
    parameters: list    # Required/optional query parameters
    supports_excel: bool
    supports_pdf: bool  # Future

REPORT_REGISTRY: dict[str, ReportDefinition] = {}

def register_report(definition: ReportDefinition):
    REPORT_REGISTRY[definition.code] = definition
```

This allows future reports to be added by registering a new service — no router changes needed. The `GET /reports` endpoint returns available reports dynamically from the registry.

### 6. Abstract CSV Parser Pattern

```python
# services/parsers/base.py
class BaseCSVParser(ABC):
    """All CSV parsers implement this interface."""

    @abstractmethod
    def can_parse(self, headers: list[str], filename: str) -> bool:
        """Return True if this parser handles the given CSV format."""

    @abstractmethod
    def parse(self, file_content: IO, bank_account_id: int) -> list[ParsedTransaction]:
        """Parse CSV content into standardized transactions."""

    @abstractmethod
    def get_format_name(self) -> str:
        """Human-readable format name for UI display."""

# Parser registry — auto-detects format
PARSER_REGISTRY: list[BaseCSVParser] = []
```

New bank formats require only a new parser class + registration. No changes to import pipeline.

---

## Sprint Plan

### Sprint 0: Pre-Development (Days 1-2)
> **Goal:** Environment setup and project scaffold validation

**Tasks:**

| # | Task | Files | Done When |
|---|---|---|---|
| 0.1 | Verify Python 3.11+ installed, install `uv` | — | `python --version` ≥ 3.11, `uv --version` works |
| 0.2 | Verify Node 18+ installed, npm available | — | `node --version` ≥ 18 |
| 0.3 | Create monorepo directory structure | `Makefile`, `.env.example`, `.gitignore`, `backend/pyproject.toml`, `frontend/package.json` | `ls` shows full tree from v2 §3 |
| 0.4 | Initialize uv project with core dependencies | `backend/pyproject.toml` | `uv sync` succeeds |
| 0.5 | Initialize Vite + React + TypeScript + Tailwind | `frontend/` scaffold | `npm run dev` serves empty app |
| 0.6 | Create Makefile with `make dev`, `make backend`, `make frontend`, `make seed`, `make migrate` | `Makefile` | `make dev` starts both servers |
| 0.7 | Create CLAUDE.md with project conventions | `CLAUDE.md` | Conventions documented |

**Dependencies (backend/pyproject.toml):**
```
fastapi, uvicorn[standard], sqlalchemy[asyncio], alembic,
pydantic, pydantic-settings, pandas, openpyxl,
python-multipart, aiofiles, httpx (testing)
```

**Acceptance Criteria:**
- [ ] `make dev` starts FastAPI on :8000 and Vite on :5173
- [ ] FastAPI docs available at localhost:8000/docs
- [ ] Empty React app renders at localhost:5173

---

### Sprint 1: Data Model Foundation (Weeks 1-2)
> **Goal:** All database models migrated, lookup tables seeded, basic CRUD for core entities

**Tasks:**

| # | Task | Files |
|---|---|---|
| 1.1 | SQLAlchemy base, engine, session factory | `backend/app/database.py` |
| 1.2 | App config via pydantic-settings | `backend/app/config.py` |
| 1.3 | FastAPI app with CORS, lifespan, router registration | `backend/app/main.py` |
| 1.4 | LookupType + LookupValue models | `backend/app/models/lookup.py` |
| 1.5 | SystemSetting model | `backend/app/models/system_setting.py` |
| 1.6 | Entity model | `backend/app/models/entity.py` |
| 1.7 | Property model (with v2 extended asset fields) | `backend/app/models/property.py` |
| 1.8 | BankAccount model | `backend/app/models/bank_account.py` |
| 1.9 | Account model (hierarchical CoA, replaces Category) | `backend/app/models/account.py` |
| 1.10 | Transaction model (with property_id, journal_entry_id placeholder) | `backend/app/models/transaction.py` |
| 1.11 | ImportBatch model | `backend/app/models/import_batch.py` |
| 1.12 | CategorizationRule model | `backend/app/models/categorization_rule.py` |
| 1.13 | Tenant model | `backend/app/models/tenant.py` |
| 1.14 | Lease model | `backend/app/models/lease.py` |
| 1.15 | RentPayment model | `backend/app/models/rent_payment.py` |
| 1.16 | Loan + LoanAdvance models | `backend/app/models/loan.py` |
| 1.17 | Mortgage + MortgagePayment models | `backend/app/models/mortgage.py` |
| 1.18 | MaintenanceItem model | `backend/app/models/maintenance_item.py` |
| 1.19 | FiscalConfig model | `backend/app/models/fiscal_config.py` |
| 1.20 | Budget model | `backend/app/models/budget.py` |
| 1.21 | BalanceSheetSnapshot model | `backend/app/models/balance_sheet_snapshot.py` |
| 1.22 | JournalEntry + JournalLine placeholder models | `backend/app/models/journal_entry.py` |
| 1.23 | Models `__init__.py` — export all models | `backend/app/models/__init__.py` |
| 1.24 | Alembic init + initial migration | `backend/alembic.ini`, `backend/migrations/` |
| 1.25 | Seed script: lookup types + values | `backend/app/seeds/lookups.py` |
| 1.26 | Seed script: system settings | `backend/app/seeds/settings.py` |
| 1.27 | Seed script: Chart of Accounts | `backend/app/seeds/accounts.py` |
| 1.28 | Seed script: entities, properties, bank accounts | `backend/app/seeds/entities.py` |
| 1.29 | Seed script: tenants, leases | `backend/app/seeds/tenants.py` |
| 1.30 | Seed script: categorization rules | `backend/app/seeds/rules.py` |
| 1.31 | Seed script: mortgages, fiscal config, budgets | `backend/app/seeds/financial.py` |
| 1.32 | Seed orchestrator + `make seed` | `backend/app/seeds/__init__.py` |
| 1.33 | Pydantic schemas for LookupType/Value, SystemSetting | `backend/app/schemas/lookup.py`, `backend/app/schemas/system_setting.py` |
| 1.34 | Pydantic schemas for Entity, Property, BankAccount | `backend/app/schemas/entity.py`, `backend/app/schemas/property.py`, `backend/app/schemas/bank_account.py` |
| 1.35 | Pydantic schemas for Account | `backend/app/schemas/account.py` |
| 1.36 | Router: lookups CRUD | `backend/app/routers/lookups.py` |
| 1.37 | Router: settings CRUD | `backend/app/routers/settings.py` |
| 1.38 | Router: entities CRUD | `backend/app/routers/entities.py` |
| 1.39 | Router: properties CRUD | `backend/app/routers/properties.py` |
| 1.40 | Router: bank_accounts CRUD | `backend/app/routers/bank_accounts.py` |
| 1.41 | Router: accounts CRUD (hierarchical) | `backend/app/routers/accounts.py` |
| 1.42 | Unit tests: models, seeds, CRUD endpoints | `backend/tests/` |

**Key Model Design Decisions:**
- Every model with a "type" field uses `lookup_value_id` (FK) instead of a string enum
- All money fields: `Integer` (cents) in DB, `Decimal` in Python, formatted on API boundary
- All models inherit from a `TimestampMixin` (created_at, updated_at)
- Property model includes: `units`, `purchase_price`, `land_value`, `cost_basis_building`, `acquisition_date`, `down_payment`, `property_type_id` (FK → LookupValue)

**Acceptance Criteria:**
- [ ] `make seed` creates database with all lookup tables populated
- [ ] `GET /lookups` returns all lookup types with their values
- [ ] `GET /lookups/property_type/values` returns ["residential", "commercial", "multi_unit", "mixed_use"]
- [ ] `POST /lookups/property_type/values` with `{"code": "industrial", "name": "Industrial"}` adds a new property type — **zero code changes**
- [ ] `GET /settings` returns all system settings
- [ ] `PUT /settings/fiscal.current_year` updates the fiscal year
- [ ] `GET /entities` returns Strathcona 1 LLC and Magnet Labs
- [ ] `GET /accounts` returns full hierarchical Chart of Accounts
- [ ] `POST /accounts` creates a new account under any parent — **no code change needed for new account types**
- [ ] `GET /properties` returns properties with extended asset fields
- [ ] All CRUD endpoints return proper 404/422 responses
- [ ] Database schema supports forward migration to PostgreSQL (no SQLite-specific types)
- [ ] Alembic can generate migration from current models: `alembic revision --autogenerate`

---

### Sprint 2: Transaction Import Engine (Weeks 3-4)
> **Goal:** Upload Chase CSVs, transactions auto-categorized and stored, property tagging works

**Tasks:**

| # | Task | Files |
|---|---|---|
| 2.1 | ParsedTransaction dataclass | `backend/app/services/parsers/base.py` |
| 2.2 | Abstract BaseCSVParser + parser registry | `backend/app/services/parsers/base.py` |
| 2.3 | Chase Checking CSV parser | `backend/app/services/parsers/chase_checking.py` |
| 2.4 | Chase Credit Card CSV parser | `backend/app/services/parsers/chase_credit.py` |
| 2.5 | Generic/auto-detect CSV parser (fallback) | `backend/app/services/parsers/generic.py` |
| 2.6 | Parser auto-detection: inspect headers → select parser | `backend/app/services/parsers/__init__.py` |
| 2.7 | Duplicate detection service | `backend/app/services/duplicate_detector.py` |
| 2.8 | Rule-based auto-categorizer service | `backend/app/services/categorizer.py` |
| 2.9 | Import orchestrator: parse → dedupe → categorize → persist | `backend/app/services/import_service.py` |
| 2.10 | Plaid stub service | `backend/app/services/plaid_stub.py` |
| 2.11 | Pydantic schemas for Transaction, ImportBatch | `backend/app/schemas/transaction.py`, `backend/app/schemas/import_batch.py` |
| 2.12 | Pydantic schemas for CategorizationRule | `backend/app/schemas/categorization_rule.py` |
| 2.13 | Router: imports (upload, history, detail) | `backend/app/routers/imports.py` |
| 2.14 | Router: transactions CRUD + filters | `backend/app/routers/transactions.py` |
| 2.15 | Transaction bulk-categorize endpoint | `backend/app/routers/transactions.py` |
| 2.16 | Transaction re-categorize endpoint | `backend/app/routers/transactions.py` |
| 2.17 | Transaction bulk-tag-property endpoint | `backend/app/routers/transactions.py` |
| 2.18 | Router: categorization-rules CRUD | `backend/app/routers/categorization_rules.py` |
| 2.19 | Property column parsing from 2026 CSV format | Chase parsers |
| 2.20 | Unit tests: each parser, categorizer, duplicate detection | `backend/tests/test_parsers.py`, `backend/tests/test_categorizer.py` |
| 2.21 | Integration test: full import pipeline with sample CSV | `backend/tests/test_import_pipeline.py` |

**Key Design Decisions:**
- Parser registry auto-detects CSV format from headers — no user selection needed
- `can_parse()` method on each parser checks if headers match its expected format
- New bank formats: create a parser class, register it. Zero changes to import endpoint
- Categorization rules stored in DB, editable via API — no hardcoded rules in code
- Property column: if CSV has a "Property" column, parser extracts and maps to `property_id`
- Duplicate hash: `sha256(bank_account_id + transaction_date + amount + description_raw)`

**Acceptance Criteria:**
- [ ] `POST /imports/upload` with Chase5271 CSV creates transactions, ~80% auto-categorized
- [ ] `POST /imports/upload` with Chase9268 CSV works (credit card format)
- [ ] Uploading the same CSV twice flags duplicates (count shown, not blocked)
- [ ] `GET /transactions?uncategorized=true` returns only uncategorized transactions
- [ ] `GET /transactions?property_id=1` filters by property
- [ ] `POST /transactions/bulk-tag-property` assigns property to multiple transactions
- [ ] `POST /transactions/bulk-categorize` categorizes multiple transactions at once
- [ ] `GET /categorization-rules` returns all rules; `POST` creates new rules
- [ ] Adding a new categorization rule and re-running categorizer on existing transactions works
- [ ] Parser auto-detection: uploading any supported CSV format auto-selects the correct parser
- [ ] Unsupported CSV format returns clear error message

---

### Sprint 3: Core Reporting Engine (Weeks 5-6)
> **Goal:** All 5 baseline reports working + 2 new reports. Numbers verified against spreadsheet.

**Tasks:**

| # | Task | Files |
|---|---|---|
| 3.1 | Report registry + base report service | `backend/app/services/report_registry.py` |
| 3.2 | Monthly P&L report service (with budget/variance) | `backend/app/services/reports/report_pl.py` |
| 3.3 | Annual P&L rollup service | `backend/app/services/reports/report_pl.py` |
| 3.4 | Rent Roll report service (with monthly collection grid) | `backend/app/services/reports/report_rent_roll.py` |
| 3.5 | Loan Ledger report service (with interest accrual) | `backend/app/services/reports/report_loan.py` |
| 3.6 | Loan interest accrual engine | `backend/app/services/engines/interest_accrual.py` |
| 3.7 | Schedule E / Tax Prep report service (basic) | `backend/app/services/reports/report_schedule_e.py` |
| 3.8 | Dashboard KPI report service | `backend/app/services/reports/report_dashboard.py` |
| 3.9 | YoY Comparison report service | `backend/app/services/reports/report_yoy.py` |
| 3.10 | Excel exporter service (openpyxl) | `backend/app/services/exporter.py` |
| 3.11 | Pydantic schemas for all report responses | `backend/app/schemas/reports.py` |
| 3.12 | Router: reports (all endpoints, JSON + XLSX) | `backend/app/routers/reports.py` |
| 3.13 | Register all reports in registry | `backend/app/services/reports/__init__.py` |
| 3.14 | Pydantic schemas for Tenant, Lease, RentPayment | `backend/app/schemas/tenant.py`, `backend/app/schemas/lease.py`, `backend/app/schemas/rent_payment.py` |
| 3.15 | Router: tenants CRUD | `backend/app/routers/tenants.py` |
| 3.16 | Router: leases CRUD | `backend/app/routers/leases.py` |
| 3.17 | Router: rent-payments CRUD + match-transaction | `backend/app/routers/rent_payments.py` |
| 3.18 | Pydantic schemas for Loan, LoanAdvance | `backend/app/schemas/loan.py` |
| 3.19 | Router: loans CRUD + advances + interest summary | `backend/app/routers/loans.py` |
| 3.20 | Unit tests: each report service against known data | `backend/tests/test_reports/` |

**Key Design Decisions:**
- All reports registered in `REPORT_REGISTRY` — `GET /reports` returns available reports dynamically
- Report services are pure functions: input (entity_id, property_id, dates) → output (structured data)
- P&L line ordering driven by `Account.display_order`, not hardcoded
- Budget lookup: `Budget` table keyed by `(entity_id, property_id, fiscal_year, pl_line_name)`. Missing budget → `None` variance, not zero
- Interest accrual always computed fresh — never cached
- All reports support `?format=json` (default) and `?format=xlsx`

**Acceptance Criteria:**
- [ ] `GET /reports/pl?entity_id=1&month=1&year=2026` returns Monthly P&L matching spreadsheet Jan 2026
- [ ] P&L includes Budget, Variance $, Variance % columns when budgets exist
- [ ] `GET /reports/pl/annual?entity_id=1&year=2026` returns 12-column rollup
- [ ] `GET /reports/rent-roll?property_id=1&fiscal_year=2026` returns tenant grid with monthly ✓/✗
- [ ] `GET /reports/loan?loan_id=1&as_of_date=2026-03-01` returns advances with per-advance interest
- [ ] `GET /reports/tax-prep?entity_id=1&property_id=1&year=2026` returns Schedule E mapping
- [ ] `GET /reports/dashboard?entity_id=1&year=2026` returns KPIs: EGI, NOI, DSCR, etc.
- [ ] `GET /reports/yoy?entity_id=1&current_year=2026&prior_year=2025` returns side-by-side P&L
- [ ] `GET /reports` returns list of all available reports with their parameters
- [ ] All report endpoints support `?format=xlsx` for Excel download
- [ ] Adding a new P&L line (new Account) automatically appears in P&L reports — **no code change**

---

### Sprint 4: Financial Engines (Weeks 7-8)
> **Goal:** Depreciation, mortgage amortization, and advanced reports fully operational

**Tasks:**

| # | Task | Files |
|---|---|---|
| 4.1 | Depreciation engine (building straight-line) | `backend/app/services/engines/depreciation.py` |
| 4.2 | Depreciation engine (CapEx per-item) | `backend/app/services/engines/depreciation.py` |
| 4.3 | Depreciation method lookup (configurable via SystemSetting) | `backend/app/services/engines/depreciation.py` |
| 4.4 | Mortgage amortization engine | `backend/app/services/engines/mortgage_amort.py` |
| 4.5 | Mortgage payment schedule generator + storage | `backend/app/services/engines/mortgage_amort.py` |
| 4.6 | Pydantic schemas for Mortgage, MortgagePayment | `backend/app/schemas/mortgage.py` |
| 4.7 | Router: mortgages CRUD + amortization + recalculate | `backend/app/routers/mortgages.py` |
| 4.8 | Pydantic schemas for MaintenanceItem | `backend/app/schemas/maintenance.py` |
| 4.9 | Router: maintenance CRUD + link-transaction + summary | `backend/app/routers/maintenance.py` |
| 4.10 | Pydantic schemas for Budget | `backend/app/schemas/budget.py` |
| 4.11 | Router: budgets CRUD + bulk-upsert | `backend/app/routers/budgets.py` |
| 4.12 | Pydantic schemas for FiscalConfig | `backend/app/schemas/fiscal_config.py` |
| 4.13 | Pydantic schemas for BalanceSheetSnapshot | `backend/app/schemas/balance_sheet_snapshot.py` |
| 4.14 | Router: balance-sheet-snapshots CRUD + year-end-rollover | `backend/app/routers/balance_sheet_snapshots.py` |
| 4.15 | Balance Sheet report service | `backend/app/services/reports/report_balance_sheet.py` |
| 4.16 | NOI/Cash Flow report service | `backend/app/services/reports/report_noi_cashflow.py` |
| 4.17 | Maintenance/CapEx summary report service | `backend/app/services/reports/report_maint_capex.py` |
| 4.18 | Tax Prep enhanced (depreciation section, tax impact, imputed interest) | `backend/app/services/reports/report_schedule_e.py` |
| 4.19 | Register new reports in registry | `backend/app/services/reports/__init__.py` |
| 4.20 | Seed: mortgage data (Weideman property) | `backend/app/seeds/financial.py` |
| 4.21 | Seed: balance sheet snapshot (Dec 31, 2025) | `backend/app/seeds/financial.py` |
| 4.22 | Unit tests: depreciation, amortization, balance sheet | `backend/tests/test_engines/` |

**Key Design Decisions:**
- Depreciation years configurable via SystemSetting (`fiscal.depreciation_years_residential` = 27.5)
- Depreciation method extensible via LookupValue — straight_line now, declining_balance later
- Mortgage schedule stored in MortgagePayment rows for query performance, regenerated on recalculate
- Balance Sheet prior-year column from BalanceSheetSnapshot; current-year column fully computed
- Year-end rollover copies computed values → new snapshot (automating annual close)
- NOI/Cash Flow: all line items derived from Account hierarchy, not hardcoded

**Acceptance Criteria:**
- [ ] `GET /mortgages/1/amortization?year=2026` returns 12 monthly rows matching spreadsheet
- [ ] Weideman mortgage year-end balance = $36,639.49 (or within rounding)
- [ ] Annual building depreciation per property matches `cost_basis / 27.5`
- [ ] Combined Strathcona+Weideman depreciation = ~$10,145.45/year
- [ ] `GET /reports/balance-sheet?entity_id=1&as_of_year=2026` returns full balance sheet
- [ ] Balance check (Assets - Liabilities - Equity) = 0
- [ ] `GET /reports/noi-cashflow?entity_id=1&year=2026` returns NOI, DSCR, cash-on-cash
- [ ] `GET /reports/maint-capex?property_id=1&year=2026` returns categorized maintenance log
- [ ] `GET /reports/tax-prep` now includes depreciation section + tax impact estimate
- [ ] `POST /balance-sheet-snapshots/year-end-rollover` copies current year to next year's prior
- [ ] Changing `fiscal.depreciation_years_residential` setting changes depreciation calculations — **no code change**
- [ ] Adding a new maintenance item type via lookup API works without code changes

---

### Sprint 5: Frontend Foundation (Weeks 9-10)
> **Goal:** React app with navigation, Dashboard, Transactions page, and Import page

**Tasks:**

| # | Task | Files |
|---|---|---|
| 5.1 | Project structure: pages, components, api, hooks, types, utils | `frontend/src/` |
| 5.2 | API client layer (Axios, base URL config, error handling) | `frontend/src/api/client.ts` |
| 5.3 | API modules: entities, properties, transactions, imports | `frontend/src/api/` |
| 5.4 | API modules: lookups, settings, accounts | `frontend/src/api/` |
| 5.5 | TypeScript type definitions (mirroring Pydantic schemas) | `frontend/src/types/` |
| 5.6 | App layout: sidebar navigation, header, content area | `frontend/src/components/layout/` |
| 5.7 | React Router setup with all page routes | `frontend/src/App.tsx` |
| 5.8 | Reusable: DataTable component (sortable, filterable) | `frontend/src/components/ui/DataTable.tsx` |
| 5.9 | Reusable: PropertySelector component | `frontend/src/components/ui/PropertySelector.tsx` |
| 5.10 | Reusable: KPICard component | `frontend/src/components/ui/KPICard.tsx` |
| 5.11 | Reusable: MonthlyGrid component | `frontend/src/components/ui/MonthlyGrid.tsx` |
| 5.12 | Reusable: CurrencyDisplay component (cents → formatted) | `frontend/src/components/ui/CurrencyDisplay.tsx` |
| 5.13 | Reusable: LookupSelect component (fetches options from API) | `frontend/src/components/ui/LookupSelect.tsx` |
| 5.14 | Dashboard page: KPI tiles + monthly chart + expense donut | `frontend/src/pages/Dashboard.tsx` |
| 5.15 | Transactions page: table, filters, property tag, bulk ops | `frontend/src/pages/Transactions.tsx` |
| 5.16 | Import page: CSV drag-drop, format auto-detect, history | `frontend/src/pages/Import.tsx` |
| 5.17 | Notification/toast system | `frontend/src/components/ui/Toast.tsx` |
| 5.18 | Loading states + error boundaries | `frontend/src/components/ui/` |

**Key Design Decisions:**
- `LookupSelect` component: pass `lookupTypeCode` prop → fetches values from `/lookups/{code}/values` → renders dropdown. Used everywhere type selection is needed. Adding a new lookup value in the DB instantly appears in all dropdowns — **zero frontend changes**.
- `PropertySelector`: fetches from `/properties`, used across all report pages
- `DataTable`: generic, sortable, filterable — used for transactions, maintenance, amortization tables
- All money display goes through `CurrencyDisplay` component (reads `display.currency_symbol` from settings)
- Chart library: Recharts (lightweight, React-native)

**Acceptance Criteria:**
- [ ] App loads at localhost:5173 with sidebar navigation
- [ ] Dashboard shows 6 KPI tiles with live data from API
- [ ] Dashboard monthly summary table shows Jan–Dec with income/expenses/NOI
- [ ] Dashboard expense breakdown chart renders
- [ ] Property selector on Dashboard filters all data by property
- [ ] Transactions page shows paginated, sortable, filterable transaction table
- [ ] Transactions page: inline property dropdown per row (populated from DB, not hardcoded)
- [ ] Transactions page: bulk select → assign property works
- [ ] Transactions page: untagged transaction count badge visible
- [ ] Import page: drag-drop CSV → upload → shows import results with counts
- [ ] Import page: shows import history with date, file, row count, duplicates
- [ ] All dropdowns for types (property type, account type, etc.) populated from `/lookups` API
- [ ] Adding a new lookup value via API → refreshing page shows new option in dropdown

---

### Sprint 6: Frontend — Management Pages (Weeks 11-12)
> **Goal:** All CRUD management pages + report viewer pages

**Tasks:**

| # | Task | Files |
|---|---|---|
| 6.1 | Properties page: list + detail + asset info + mortgage summary | `frontend/src/pages/Properties.tsx` |
| 6.2 | Tenants page: tenant list + monthly payment grid | `frontend/src/pages/Tenants.tsx` |
| 6.3 | Tenants: click cell to mark paid/unpaid/partial | `frontend/src/pages/Tenants.tsx` |
| 6.4 | Tenants: lease expiry badges (amber <60d, red <30d) | `frontend/src/pages/Tenants.tsx` |
| 6.5 | Loans page: advance table + interest accrual display | `frontend/src/pages/Loans.tsx` |
| 6.6 | Maintenance/CapEx page: add form + table + summary cards | `frontend/src/pages/Maintenance.tsx` |
| 6.7 | Maintenance: link-to-transaction modal | `frontend/src/pages/Maintenance.tsx` |
| 6.8 | Mortgage Amortization page: selector + schedule + summary | `frontend/src/pages/MortgageAmort.tsx` |
| 6.9 | Settings page: system settings editor | `frontend/src/pages/Settings.tsx` |
| 6.10 | Settings page: lookup table manager (add/edit/reorder values) | `frontend/src/pages/Settings.tsx` |
| 6.11 | Settings page: Chart of Accounts manager (tree view, CRUD) | `frontend/src/pages/Settings.tsx` |
| 6.12 | Settings page: categorization rules manager | `frontend/src/pages/Settings.tsx` |
| 6.13 | Budget management page: per-line-item monthly budget editor | `frontend/src/pages/Budgets.tsx` |
| 6.14 | BudgetVarianceRow component | `frontend/src/components/ui/BudgetVarianceRow.tsx` |
| 6.15 | DepreciationSummary component | `frontend/src/components/ui/DepreciationSummary.tsx` |

**Key Design Decisions:**
- Settings page is the **control center**: lookup management, CoA editor, categorization rules, system settings
- Lookup table manager: generic CRUD for any LookupType — add/edit/reorder/deactivate values
- CoA manager: tree view of accounts with drag-to-reorder, add child, edit, deactivate
- Budget editor: grid layout matching P&L lines, monthly amounts editable inline
- All forms dynamically render fields based on lookup values — no hardcoded form options

**Acceptance Criteria:**
- [ ] Properties page shows asset details (purchase price, land value, cost basis)
- [ ] Properties page shows mortgage summary card with current balance
- [ ] Tenants page shows 12-month payment grid per tenant
- [ ] Clicking a cell in payment grid toggles paid/unpaid/partial status
- [ ] Lease expiry badges show correct colors based on days remaining
- [ ] Loans page shows per-advance interest accrual + total outstanding
- [ ] Maintenance page: add repair/capex item with all fields
- [ ] Maintenance page: link item to existing transaction
- [ ] Maintenance summary cards show YTD totals by type
- [ ] Mortgage Amort page shows correct 12-month schedule for selected year
- [ ] Settings page: can add a new property type → immediately available in Property form
- [ ] Settings page: can add a new account to CoA → immediately appears in reports
- [ ] Settings page: can edit system settings (fiscal year, depreciation years, etc.)
- [ ] Settings page: can manage categorization rules (add/edit/delete/reorder)
- [ ] Budget page: can set monthly budgets per P&L line item
- [ ] Budget page: bulk save works

---

### Sprint 7: Frontend — Report Pages (Weeks 13-14)
> **Goal:** All financial report pages with print/export functionality

**Tasks:**

| # | Task | Files |
|---|---|---|
| 7.1 | Reports index page: available reports list from registry | `frontend/src/pages/Reports.tsx` |
| 7.2 | Monthly P&L page with budget/variance columns | `frontend/src/pages/reports/MonthlyPL.tsx` |
| 7.3 | Annual P&L rollup page (12-column) | `frontend/src/pages/reports/AnnualPL.tsx` |
| 7.4 | Balance Sheet page: two-column prior/current + balance check | `frontend/src/pages/reports/BalanceSheet.tsx` |
| 7.5 | Balance Sheet: editable prior-year starting balances | `frontend/src/pages/reports/BalanceSheet.tsx` |
| 7.6 | Balance Sheet: year-end rollover button | `frontend/src/pages/reports/BalanceSheet.tsx` |
| 7.7 | YoY Comparison page: side-by-side P&L + color coding | `frontend/src/pages/reports/YoYComparison.tsx` |
| 7.8 | NOI/Cash Flow page: statement layout + key metrics | `frontend/src/pages/reports/NOICashFlow.tsx` |
| 7.9 | Tax Prep page: Schedule E mapping + depreciation + tax impact | `frontend/src/pages/reports/TaxPrep.tsx` |
| 7.10 | Tax Prep: editable marginal rate for tax impact estimate | `frontend/src/pages/reports/TaxPrep.tsx` |
| 7.11 | Rent Roll report page (full grid view) | `frontend/src/pages/reports/RentRoll.tsx` |
| 7.12 | Maintenance/CapEx report page | `frontend/src/pages/reports/MaintCapEx.tsx` |
| 7.13 | Export button component (XLSX download for all reports) | `frontend/src/components/ui/ExportButton.tsx` |
| 7.14 | Print-friendly CSS for all report pages | `frontend/src/styles/print.css` |
| 7.15 | Report parameter persistence (URL query params) | Each report page |

**Key Design Decisions:**
- Reports index page reads from `/reports` endpoint — shows only registered reports
- All report pages share common parameter bar (entity, property, year/month selectors)
- Report parameters stored in URL query params for bookmarking/sharing
- Balance sheet editable cells use inline-edit pattern with save-on-blur
- Color coding: green = favorable variance/improvement, red = unfavorable
- Print CSS: hide nav, full-width content, proper page breaks

**Acceptance Criteria:**
- [ ] Reports index shows all available reports with descriptions
- [ ] Monthly P&L renders with Actual | Budget | Var $ | Var % columns
- [ ] Annual P&L shows 12 monthly columns + YTD total
- [ ] Balance Sheet shows Prior Year | Current Year | Change columns
- [ ] Balance Sheet balance check indicator: green ✓ when balanced
- [ ] Balance Sheet: clicking prior-year values makes them editable
- [ ] Balance Sheet: year-end rollover button works with confirmation dialog
- [ ] YoY Comparison: green/red color coding for improvements/deterioration
- [ ] NOI/Cash Flow: full waterfall from Gross Income → Cash Flow After Debt
- [ ] NOI/Cash Flow: key metrics panel shows DSCR, OER, break-even occupancy, cash-on-cash
- [ ] Tax Prep: depreciation section auto-computed from property data
- [ ] Tax Prep: editable marginal rate → computed tax impact updates in real-time
- [ ] Tax Prep: imputed interest section for Magnet Loan
- [ ] All reports: XLSX download works
- [ ] All reports: print view renders correctly (Ctrl+P)
- [ ] Navigating to a report with URL params pre-fills the parameter bar

---

### Sprint 8: Integration, Data Migration & Verification (Weeks 15-16)
> **Goal:** Real data loaded, all numbers verified against source spreadsheets, system is production-ready

**Tasks:**

| # | Task | Files |
|---|---|---|
| 8.1 | Import 2025 Chase5271 CSV data | Data task |
| 8.2 | Import 2025 Chase9268 CSV data | Data task |
| 8.3 | Import 2025 Chase0505 CSV data | Data task |
| 8.4 | Import 2026 CSV data through current month | Data task |
| 8.5 | Bulk property-tag all historical transactions | Data task |
| 8.6 | Enter all Magnet Labs loan advances to loan ledger | Data task |
| 8.7 | Enter CapEx items from Maint-CapEx log | Data task |
| 8.8 | Set 2025 balance sheet snapshot starting values | Data task |
| 8.9 | Enter 2026 monthly budgets from Config tab | Data task |
| 8.10 | Verify Monthly P&L Jan 2026 against spreadsheet | Verification |
| 8.11 | Verify Monthly P&L Feb 2026 against spreadsheet | Verification |
| 8.12 | Verify Annual P&L rollup against spreadsheet | Verification |
| 8.13 | Verify Rent Roll against spreadsheet | Verification |
| 8.14 | Verify Loan Ledger totals against spreadsheet | Verification |
| 8.15 | Verify Mortgage amort balance ($36,639.49 EOY) | Verification |
| 8.16 | Verify Depreciation totals (~$10,145.45/year) | Verification |
| 8.17 | Verify Balance Sheet totals and balance check | Verification |
| 8.18 | Verify NOI/Cash Flow metrics against spreadsheet | Verification |
| 8.19 | Verify Dashboard KPIs against spreadsheet | Verification |
| 8.20 | Fix any discrepancies found during verification | Bug fixes |
| 8.21 | End-to-end smoke test: full user workflow | `backend/tests/test_e2e.py` |
| 8.22 | Performance check: all reports <2s response time | Performance |
| 8.23 | Add Citizens Bank CSV parser (if format available) | `backend/app/services/parsers/citizens.py` |
| 8.24 | Data backup/restore documentation | `README.md` |

**Acceptance Criteria:**
- [ ] All 2025 and 2026 transaction data imported and categorized
- [ ] All transactions tagged to correct property
- [ ] Monthly P&L Jan 2026 matches spreadsheet values within $1 rounding
- [ ] Monthly P&L Feb 2026 matches spreadsheet values within $1 rounding
- [ ] Rent Roll collection grid matches spreadsheet exactly
- [ ] Loan ledger total outstanding balance matches spreadsheet
- [ ] Mortgage year-end balance = $36,639.49 (±$1 rounding)
- [ ] Annual depreciation = ~$10,145.45 (±$1 rounding)
- [ ] Balance sheet balances (Assets = Liabilities + Equity)
- [ ] Dashboard KPIs match spreadsheet Dashboard tab
- [ ] All reports generate in <2 seconds
- [ ] No hardcoded values anywhere in the codebase (verified by code review)
- [ ] System replaces spreadsheet as source of truth for Strathcona 1 LLC

---

## Sprint Summary

| Sprint | Weeks | Focus | Key Deliverable |
|---|---|---|---|
| 0 | Pre | Scaffold | `make dev` starts both servers |
| 1 | 1-2 | Data Model + CRUD | All models, seeds, lookup APIs, CoA |
| 2 | 3-4 | Transaction Import | CSV upload → auto-categorize pipeline |
| 3 | 5-6 | Core Reports | 9 report endpoints, Excel export |
| 4 | 7-8 | Financial Engines | Depreciation, mortgage, balance sheet |
| 5 | 9-10 | Frontend Foundation | Dashboard, Transactions, Import pages |
| 6 | 11-12 | Frontend Management | Properties, Tenants, Settings, Maintenance |
| 7 | 13-14 | Frontend Reports | All report pages + export + print |
| 8 | 15-16 | Data Migration + QA | Real data verified against spreadsheets |

**Total: 8 sprints × 2 weeks = 16 weeks**
**First usable deliverable:** End of Sprint 4 (week 8) — full backend with all APIs working
**Full system:** End of Sprint 8 (week 16) — complete replacement for spreadsheet

---

## File Inventory (Complete)

### Backend: Models (22 files)
```
backend/app/models/__init__.py
backend/app/models/lookup.py                  # LookupType + LookupValue
backend/app/models/system_setting.py           # SystemSetting
backend/app/models/entity.py                   # Entity
backend/app/models/property.py                 # Property (extended)
backend/app/models/bank_account.py             # BankAccount
backend/app/models/account.py                  # Account (hierarchical CoA)
backend/app/models/transaction.py              # Transaction
backend/app/models/import_batch.py             # ImportBatch
backend/app/models/categorization_rule.py      # CategorizationRule
backend/app/models/tenant.py                   # Tenant
backend/app/models/lease.py                    # Lease
backend/app/models/rent_payment.py             # RentPayment
backend/app/models/loan.py                     # Loan + LoanAdvance
backend/app/models/mortgage.py                 # Mortgage + MortgagePayment
backend/app/models/maintenance_item.py         # MaintenanceItem
backend/app/models/fiscal_config.py            # FiscalConfig
backend/app/models/budget.py                   # Budget
backend/app/models/balance_sheet_snapshot.py    # BalanceSheetSnapshot
backend/app/models/journal_entry.py            # JournalEntry + JournalLine (placeholder)
backend/app/models/mixins.py                   # TimestampMixin, CentsMixin
```

### Backend: Schemas (16 files)
```
backend/app/schemas/lookup.py
backend/app/schemas/system_setting.py
backend/app/schemas/entity.py
backend/app/schemas/property.py
backend/app/schemas/bank_account.py
backend/app/schemas/account.py
backend/app/schemas/transaction.py
backend/app/schemas/import_batch.py
backend/app/schemas/categorization_rule.py
backend/app/schemas/tenant.py
backend/app/schemas/lease.py
backend/app/schemas/rent_payment.py
backend/app/schemas/loan.py
backend/app/schemas/mortgage.py
backend/app/schemas/maintenance.py
backend/app/schemas/budget.py
backend/app/schemas/fiscal_config.py
backend/app/schemas/balance_sheet_snapshot.py
backend/app/schemas/reports.py
```

### Backend: Routers (16 files)
```
backend/app/routers/__init__.py
backend/app/routers/lookups.py
backend/app/routers/settings.py
backend/app/routers/entities.py
backend/app/routers/properties.py
backend/app/routers/bank_accounts.py
backend/app/routers/accounts.py
backend/app/routers/transactions.py
backend/app/routers/imports.py
backend/app/routers/categorization_rules.py
backend/app/routers/tenants.py
backend/app/routers/leases.py
backend/app/routers/rent_payments.py
backend/app/routers/loans.py
backend/app/routers/mortgages.py
backend/app/routers/maintenance.py
backend/app/routers/budgets.py
backend/app/routers/balance_sheet_snapshots.py
backend/app/routers/reports.py
```

### Backend: Services (18 files)
```
backend/app/services/parsers/__init__.py       # Parser registry + auto-detect
backend/app/services/parsers/base.py           # BaseCSVParser + ParsedTransaction
backend/app/services/parsers/chase_checking.py
backend/app/services/parsers/chase_credit.py
backend/app/services/parsers/generic.py
backend/app/services/import_service.py         # Import orchestrator
backend/app/services/duplicate_detector.py
backend/app/services/categorizer.py
backend/app/services/plaid_stub.py
backend/app/services/exporter.py               # Excel export
backend/app/services/report_registry.py         # Report registry pattern
backend/app/services/reports/__init__.py
backend/app/services/reports/report_pl.py
backend/app/services/reports/report_rent_roll.py
backend/app/services/reports/report_loan.py
backend/app/services/reports/report_schedule_e.py
backend/app/services/reports/report_dashboard.py
backend/app/services/reports/report_yoy.py
backend/app/services/reports/report_balance_sheet.py
backend/app/services/reports/report_noi_cashflow.py
backend/app/services/reports/report_maint_capex.py
backend/app/services/engines/__init__.py
backend/app/services/engines/depreciation.py
backend/app/services/engines/mortgage_amort.py
backend/app/services/engines/interest_accrual.py
```

### Backend: Seeds (8 files)
```
backend/app/seeds/__init__.py                  # Orchestrator
backend/app/seeds/lookups.py                   # All LookupType + LookupValue
backend/app/seeds/settings.py                  # All SystemSetting
backend/app/seeds/accounts.py                  # Chart of Accounts
backend/app/seeds/entities.py                  # Entities, Properties, BankAccounts
backend/app/seeds/tenants.py                   # Tenants, Leases
backend/app/seeds/rules.py                     # Categorization Rules
backend/app/seeds/financial.py                 # Mortgages, FiscalConfig, Budgets, BS Snapshot
```

### Frontend: Pages (18 files)
```
frontend/src/pages/Dashboard.tsx
frontend/src/pages/Transactions.tsx
frontend/src/pages/Import.tsx
frontend/src/pages/Properties.tsx
frontend/src/pages/Tenants.tsx
frontend/src/pages/Loans.tsx
frontend/src/pages/Maintenance.tsx
frontend/src/pages/MortgageAmort.tsx
frontend/src/pages/Budgets.tsx
frontend/src/pages/Settings.tsx
frontend/src/pages/reports/Reports.tsx          # Index
frontend/src/pages/reports/MonthlyPL.tsx
frontend/src/pages/reports/AnnualPL.tsx
frontend/src/pages/reports/BalanceSheet.tsx
frontend/src/pages/reports/YoYComparison.tsx
frontend/src/pages/reports/NOICashFlow.tsx
frontend/src/pages/reports/TaxPrep.tsx
frontend/src/pages/reports/RentRoll.tsx
frontend/src/pages/reports/MaintCapEx.tsx
```

### Frontend: Components (14 files)
```
frontend/src/components/layout/AppLayout.tsx
frontend/src/components/layout/Sidebar.tsx
frontend/src/components/layout/Header.tsx
frontend/src/components/ui/DataTable.tsx
frontend/src/components/ui/PropertySelector.tsx
frontend/src/components/ui/KPICard.tsx
frontend/src/components/ui/MonthlyGrid.tsx
frontend/src/components/ui/CurrencyDisplay.tsx
frontend/src/components/ui/LookupSelect.tsx
frontend/src/components/ui/BudgetVarianceRow.tsx
frontend/src/components/ui/DepreciationSummary.tsx
frontend/src/components/ui/ExportButton.tsx
frontend/src/components/ui/Toast.tsx
frontend/src/components/ui/LoadingSpinner.tsx
```

---

## Expansion Path: v3+ Features Enabled by This Architecture

This foundation explicitly supports these future features **without schema migration**:

| Future Feature | Enabled By |
|---|---|
| **Double-entry bookkeeping** | JournalEntry + JournalLine placeholder models; Account.normal_balance field |
| **New property types** | `POST /lookups/property_type/values` — zero code changes |
| **New expense categories** | `POST /accounts` — appears in reports automatically |
| **New bank CSV formats** | Create parser class implementing BaseCSVParser, register it |
| **New report types** | Create report service, register in REPORT_REGISTRY |
| **Multi-currency** | SystemSetting `display.currency_code` + `display.currency_symbol`; amounts already integer cents |
| **Non-calendar fiscal year** | SystemSetting `fiscal.year_start_month` |
| **Commercial lease types** | LookupValue additions to `lease_type` |
| **Multiple depreciation methods** | LookupValue + SystemSetting; depreciation engine already reads method from config |
| **Invoice/bill management** | New models + router + report — no existing schema changes needed |
| **Document storage** | Add `document` model with FK to any entity — polymorphic via `content_type` + `content_id` |
| **Multi-user / auth** | Add `User` model + JWT middleware — no business logic changes |
| **PostgreSQL migration** | Change SQLAlchemy connection string — no model changes (no SQLite-specific types used) |
| **Plaid live sync** | Implement `PlaidService` methods — import pipeline already uses the interface |
| **Cash flow forecasting** | New report service using existing budget + mortgage + lease data |
| **API integrations** | Webhook/event system can be added to import and transaction services |

---

## Technical Standards (Enforced Across All Sprints)

1. **Money:** Integer cents in DB, `Decimal` in Python, `ROUND_HALF_UP`. No `float` anywhere in financial code.
2. **Types:** All type fields reference `LookupValue.id` via FK. No string enums in models.
3. **Settings:** All configurable parameters in `SystemSetting`. No magic numbers in business logic.
4. **Reports:** All reports registered in `REPORT_REGISTRY`. Report services are pure functions.
5. **Parsers:** All CSV parsers implement `BaseCSVParser`. Auto-detection via `can_parse()`.
6. **Accounts:** Hierarchical, numbered CoA. New accounts added via API without code changes.
7. **Dates:** ISO 8601. All reports accept year/month as parameters — no hardcoded fiscal logic.
8. **Tests:** Each sprint includes unit tests for new services and integration tests for endpoints.
9. **API:** RESTful, consistent error responses (422 validation, 404 not found, 409 conflict).
10. **Frontend:** All dropdowns populated from API (lookups/accounts). No hardcoded option lists in JSX.
