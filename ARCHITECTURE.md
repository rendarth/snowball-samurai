# Snowball Samurai — Architecture Document

> **Version:** 0.1.0-draft
> **Last Updated:** 2026-10-11
> **Spec Reference:** [SPEC.md](./SPEC.md)

---

## 1. System Overview

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        DOCKER CONTAINER                                │
│                                                                         │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │                    RUST BINARY (snowball-samurai)                  │  │
│  │                                                                   │  │
│  │  ┌─────────────┐  ┌──────────────┐  ┌─────────────────────────┐  │  │
│  │  │  Actix Web   │  │  Background  │  │     Notification        │  │  │
│  │  │  Server      │  │  Scheduler   │  │     Dispatcher          │  │  │
│  │  │  (HTTP)      │  │  (Tokio)     │  │  (Discord/Ntfy/SMTP)   │  │  │
│  │  └──────┬───────┘  └──────┬───────┘  └────────────┬────────────┘  │  │
│  │         │                 │                       │               │  │
│  │  ┌──────▼─────────────────▼───────────────────────▼────────────┐  │  │
│  │  │                    APPLICATION CORE                          │  │  │
│  │  │                                                             │  │  │
│  │  │  ┌─────────────┐  ┌──────────────┐  ┌───────────────────┐  │  │  │
│  │  │  │   Payoff     │  │  Cash Flow   │  │   PDF Ingestion   │  │  │  │
│  │  │  │   Engines    │  │  Calculator  │  │   Pipeline        │  │  │  │
│  │  │  │             │  │              │  │                   │  │  │  │
│  │  │  │ • Snowball  │  │ • Income     │  │ • Text Extract    │  │  │  │
│  │  │  │ • Avalanche │  │ • Expenses   │  │ • Parser Profiles │  │  │  │
│  │  │  │ • Velocity  │  │ • Surplus    │  │ • Field Mapping   │  │  │  │
│  │  │  └──────┬──────┘  └──────┬───────┘  └────────┬──────────┘  │  │  │
│  │  │         │                │                   │             │  │  │
│  │  │  ┌──────▼────────────────▼───────────────────▼──────────┐  │  │  │
│  │  │  │                  DATA ACCESS LAYER                    │  │  │  │
│  │  │  │                  (rusqlite)                           │  │  │  │
│  │  │  └──────────────────────┬────────────────────────────────┘  │  │  │
│  │  └─────────────────────────┼───────────────────────────────────┘  │  │
│  └─────────────────────────────┼─────────────────────────────────────┘  │
│                                │                                        │
│  ┌─────────────────────────────▼─────────────────────────────────────┐  │
│  │              /data (Docker Volume — Unraid Appdata)                │  │
│  │                                                                   │  │
│  │  finance.db          config.toml          uploads/                │  │
│  │  (SQLite)            (User Config)        (Stored PDFs)           │  │
│  └───────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Technology Stack

| Layer | Choice | Rationale |
|-------|--------|-----------|
| **Language** | Rust (2021 edition, stable) | Single compiled binary, memory safety, tiny Docker images, no runtime dependencies. |
| **Web Framework** | Actix Web 4 | Mature, macro-based routing (`#[get]`/`#[post]`), excellent multipart file upload support, runs on Tokio. |
| **Database** | SQLite via `rusqlite` | Single-file persistence, zero-config, perfect for single-user self-hosted apps. |
| **PDF Text Extraction** | `pdf-extract` crate | Pure Rust, extracts embedded text from digital PDFs, no system dependencies. |
| **HTML Templating** | `askama` (Jinja2-style, compile-time) | Type-safe templates compiled into the binary. Zero runtime template parsing overhead. |
| **Frontend Interactivity** | HTMX + Alpine.js | Server-rendered HTML with lightweight interactivity. No Node.js, no npm, no JS build pipeline. |
| **CSS** | Tailwind CSS (CDN or pre-built) | Utility-first CSS. Can be loaded from a single vendored CSS file (no build step). |
| **Background Tasks** | `tokio-cron-scheduler` | Async cron jobs for payment reminders and cash flow nudges. |
| **Serialization** | `serde` + `serde_json` + `toml` | JSON for API responses, TOML for configuration files. |
| **Date/Time** | `chrono` | Date arithmetic for payment schedules, amortization, and due date tracking. |
| **Logging** | `tracing` + `tracing-subscriber` | Structured async-aware logging. |
| **Notifications** | `reqwest` (HTTP client) | Used only for outbound notification webhooks (Discord, Ntfy, Gotify). Compiled as optional feature. |
| **CLI Args / Config** | `clap` + `config` | CLI flags and layered config (env vars → config.toml → defaults). |
| **Containerization** | Multi-stage Dockerfile | Builder: `rust:1.82-bookworm`. Runtime: `debian:bookworm-slim` (for OpenSSL if notifications enabled) or `scratch`. |

---

## 3. Project Structure

```
snowball-samurai/
├── Cargo.toml
├── Cargo.lock
├── Dockerfile
├── docker-compose.yml
├── config.example.toml
├── SPEC.md
├── ARCHITECTURE.md
├── TASKS.md
├── README.md
├── LICENSE (MIT)
│
├── src/
│   ├── main.rs                     # Entry point: config loading, Actix server startup
│   │
│   ├── config.rs                   # Configuration struct, defaults, TOML/env loading
│   │
│   ├── db/
│   │   ├── mod.rs                  # Database module root, connection pool setup
│   │   ├── migrations.rs           # Schema creation and migration logic
│   │   ├── models.rs               # Rust structs mirroring DB tables
│   │   ├── debts.rs                # CRUD operations for debts table
│   │   ├── cashflow.rs             # CRUD for income_sources and fixed_expenses
│   │   ├── payments.rs             # CRUD for payment_ledger
│   │   └── settings.rs            # CRUD for user_settings
│   │
│   ├── engine/
│   │   ├── mod.rs                  # Engine module root, Strategy enum
│   │   ├── types.rs                # Shared types: PayoffSchedule, MonthEntry, etc.
│   │   ├── snowball.rs             # Snowball algorithm implementation
│   │   ├── avalanche.rs            # Avalanche algorithm implementation
│   │   ├── velocity.rs             # Velocity Banking simulation
│   │   └── comparison.rs           # Side-by-side strategy comparison
│   │
│   ├── ingestion/
│   │   ├── mod.rs                  # Ingestion module root
│   │   ├── pdf.rs                  # PDF text extraction via pdf-extract
│   │   ├── parser.rs               # Regex-based parser profile engine
│   │   └── profiles/
│   │       ├── mod.rs              # Profile registry and auto-detection
│   │       ├── chase.rs            # Chase credit card statement patterns
│   │       ├── capital_one.rs      # Capital One patterns
│   │       ├── discover.rs         # Discover patterns
│   │       ├── amex.rs             # American Express patterns
│   │       └── generic.rs          # Fallback generic patterns
│   │
│   ├── notifications/
│   │   ├── mod.rs                  # Notification dispatcher
│   │   ├── discord.rs              # Discord webhook
│   │   ├── ntfy.rs                 # Ntfy push
│   │   └── smtp.rs                 # Email via SMTP (P2)
│   │
│   ├── scheduler/
│   │   ├── mod.rs                  # Background task scheduler setup
│   │   └── jobs.rs                 # Cron job definitions (due date reminders, surplus nudges)
│   │
│   └── web/
│       ├── mod.rs                  # Actix app factory, route registration
│       ├── handlers/
│       │   ├── dashboard.rs        # GET /  — Main dashboard
│       │   ├── debts.rs            # GET/POST /debts — Debt list and CRUD
│       │   ├── cashflow.rs         # GET/POST /cashflow — Income and expenses
│       │   ├── upload.rs           # POST /upload — PDF file upload and parsing
│       │   ├── strategy.rs         # GET /strategy — Payoff plan comparison
│       │   ├── payments.rs         # POST /payments — Record a payment
│       │   └── settings.rs         # GET/POST /settings — App configuration
│       └── templates/
│           ├── base.html           # Base layout (nav, head, footer)
│           ├── dashboard.html      # Dashboard view
│           ├── debts.html          # Debt management view
│           ├── cashflow.html       # Cash flow management view
│           ├── upload.html         # PDF upload + verification view
│           ├── strategy.html       # Strategy comparison view
│           ├── payments.html       # Payment recording view
│           ├── settings.html       # Settings view
│           └── components/
│               ├── debt_card.html      # Individual debt card partial
│               ├── schedule_table.html # Amortization table partial
│               ├── nudge_banner.html   # "Pay extra" suggestion banner
│               └── progress_bar.html   # Debt payoff progress bar
│
├── static/
│   ├── css/
│   │   └── tailwind.min.css        # Vendored Tailwind CSS
│   ├── js/
│   │   ├── htmx.min.js            # Vendored HTMX
│   │   └── alpine.min.js          # Vendored Alpine.js
│   └── img/
│       └── logo.svg                # App logo
│
├── tests/
│   ├── engine_tests.rs             # Integration tests for payoff engines
│   ├── ingestion_tests.rs          # Tests for PDF parsing
│   ├── db_tests.rs                 # Database CRUD tests
│   └── fixtures/
│       ├── sample_chase.pdf        # Test PDF fixture
│       └── sample_generic.pdf      # Test PDF fixture
│
└── data/                           # Docker volume mount point (not in git)
    ├── finance.db
    ├── config.toml
    └── uploads/
```

---

## 4. Database Schema

All tables use SQLite. Integer primary keys with `AUTOINCREMENT`. Monetary values stored as **integer cents** (i.e., `$1,234.56` is stored as `123456`) to avoid floating-point rounding errors. APR stored as basis points (i.e., `24.99%` is stored as `2499`).

### 4.1 `debts`

```sql
CREATE TABLE IF NOT EXISTS debts (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    nickname        TEXT NOT NULL,                          -- "Chase Sapphire"
    institution     TEXT NOT NULL DEFAULT '',               -- "Chase"
    account_last4   TEXT NOT NULL DEFAULT '',               -- "4821"
    debt_type       TEXT NOT NULL DEFAULT 'credit_card',    -- enum: credit_card, auto_loan, student_loan,
                                                           --       personal_loan, mortgage, medical,
                                                           --       heloc_ploc, other
    current_balance_cents   INTEGER NOT NULL,               -- Balance in cents
    apr_basis_points        INTEGER NOT NULL,               -- APR in basis points (2499 = 24.99%)
    minimum_payment_cents   INTEGER NOT NULL,               -- Minimum monthly payment in cents
    due_day_of_month        INTEGER NOT NULL,               -- 1-28 (day of month payment is due)
    credit_limit_cents      INTEGER,                        -- Only for revolving credit / LOC
    is_velocity_loc         INTEGER NOT NULL DEFAULT 0,     -- 1 if this is the velocity banking LOC
    status                  TEXT NOT NULL DEFAULT 'active',  -- active, paid_off
    created_at              TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at              TEXT NOT NULL DEFAULT (datetime('now'))
);
```

### 4.2 `income_sources`

```sql
CREATE TABLE IF NOT EXISTS income_sources (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    name            TEXT NOT NULL,                          -- "Primary Salary"
    amount_cents    INTEGER NOT NULL,                       -- Net take-home per occurrence
    frequency       TEXT NOT NULL DEFAULT 'biweekly',       -- weekly, biweekly, semi_monthly, monthly
    next_pay_date   TEXT NOT NULL,                          -- ISO 8601 date (YYYY-MM-DD)
    is_active       INTEGER NOT NULL DEFAULT 1,
    created_at      TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at      TEXT NOT NULL DEFAULT (datetime('now'))
);
```

### 4.3 `fixed_expenses`

```sql
CREATE TABLE IF NOT EXISTS fixed_expenses (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    name            TEXT NOT NULL,                          -- "Rent", "Car Insurance"
    amount_cents    INTEGER NOT NULL,                       -- Monthly amount in cents
    frequency       TEXT NOT NULL DEFAULT 'monthly',        -- weekly, biweekly, monthly
    due_day_of_month INTEGER,                               -- Optional: day of month
    category        TEXT NOT NULL DEFAULT 'essential',      -- essential, discretionary
    is_active       INTEGER NOT NULL DEFAULT 1,
    created_at      TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at      TEXT NOT NULL DEFAULT (datetime('now'))
);
```

### 4.4 `payment_ledger`

```sql
CREATE TABLE IF NOT EXISTS payment_ledger (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    debt_id         INTEGER NOT NULL REFERENCES debts(id),
    payment_date    TEXT NOT NULL,                          -- ISO 8601 date
    amount_cents    INTEGER NOT NULL,
    payment_type    TEXT NOT NULL DEFAULT 'minimum',        -- minimum, extra, snowball, velocity_chunk
    note            TEXT DEFAULT '',
    created_at      TEXT NOT NULL DEFAULT (datetime('now'))
);
```

### 4.5 `one_time_events`

```sql
CREATE TABLE IF NOT EXISTS one_time_events (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    event_type      TEXT NOT NULL,                          -- income, expense
    name            TEXT NOT NULL,                          -- "Tax Refund", "Car Repair"
    amount_cents    INTEGER NOT NULL,
    event_date      TEXT NOT NULL,                          -- ISO 8601 date
    is_applied      INTEGER NOT NULL DEFAULT 0,            -- Has it been factored in?
    created_at      TEXT NOT NULL DEFAULT (datetime('now'))
);
```

### 4.6 `user_settings`

```sql
CREATE TABLE IF NOT EXISTS user_settings (
    key             TEXT PRIMARY KEY,
    value           TEXT NOT NULL
);

-- Default settings inserted on first run:
-- ('safety_buffer_cents', '50000')           -- $500.00
-- ('active_strategy', 'snowball')            -- snowball, avalanche, velocity
-- ('currency_symbol', '$')
-- ('reminder_days_before_due', '3')
-- ('notification_channel', 'none')           -- none, discord, ntfy, smtp
-- ('notification_endpoint', '')              -- webhook URL or SMTP config JSON
-- ('notification_include_amounts', '0')      -- 0 = safe/private, 1 = include $ amounts
```

### 4.7 `parser_profiles`

```sql
CREATE TABLE IF NOT EXISTS parser_profiles (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    name            TEXT NOT NULL UNIQUE,                   -- "Chase Credit Card"
    institution     TEXT NOT NULL DEFAULT '',
    patterns_json   TEXT NOT NULL,                          -- JSON object of field → regex pattern
    is_builtin      INTEGER NOT NULL DEFAULT 0,
    created_at      TEXT NOT NULL DEFAULT (datetime('now'))
);
```

The `patterns_json` structure:

```json
{
  "balance": "New Balance\\s+\\$([\\d,]+\\.\\d{2})",
  "minimum_payment": "Minimum Payment Due\\s+\\$([\\d,]+\\.\\d{2})",
  "apr": "Purchase APR\\s+([\\d.]+)%",
  "due_date": "Payment Due Date\\s+(\\d{2}/\\d{2}/\\d{4})",
  "statement_date": "Statement Closing Date\\s+(\\d{2}/\\d{2}/\\d{4})",
  "account_last4": "Account Number.*?(\\d{4})$"
}
```

---

## 5. Core Algorithms

### 5.1 Snowball Method

```
FUNCTION snowball_payoff(debts: [Debt], monthly_surplus_cents: i64) -> PayoffSchedule:
    sorted = debts.sort_by(|a, b| a.balance ASC, b.apr DESC)
    schedule = []
    month = 0

    WHILE any debt has balance > 0:
        month += 1
        remaining_surplus = monthly_surplus_cents

        FOR each debt in sorted:
            IF debt.balance <= 0: CONTINUE

            // Calculate monthly interest
            monthly_interest = debt.balance * debt.apr / 10000 / 12
            debt.balance += monthly_interest

            // Determine payment
            IF debt == first_nonzero(sorted):
                payment = debt.minimum + remaining_surplus
                remaining_surplus = 0
            ELSE:
                payment = debt.minimum

            // Apply payment
            actual_payment = min(payment, debt.balance)
            debt.balance -= actual_payment

            // If paid off, add freed minimum to surplus for next debt
            IF debt.balance <= 0:
                debt.balance = 0
                remaining_surplus += debt.minimum  // Snowball roll

            record(month, debt.id, actual_payment, debt.balance, monthly_interest)

    RETURN schedule
```

### 5.2 Avalanche Method

Identical to Snowball except:
```
sorted = debts.sort_by(|a, b| a.apr DESC, b.balance ASC)
```

### 5.3 Velocity Banking Simulation

```
FUNCTION velocity_simulate(
    debts: [Debt],
    loc: Debt,          // The designated Line of Credit
    income_events: [IncomeEvent],    // Dated income deposits
    expense_events: [ExpenseEvent],  // Dated expense withdrawals
    safety_reserve_cents: i64,
    start_date: Date
) -> VelocitySchedule:

    timeline = []
    current_date = start_date

    WHILE target_debt.balance > 0 OR loc.balance > 0:

        // === CHUNK PHASE (beginning of each cycle) ===
        // Calculate max chunk: available LOC headroom minus safety
        available_headroom = loc.credit_limit - loc.balance - safety_reserve_cents
        projected_expenses = sum_expenses_until_next_income(current_date, expense_events)
        max_chunk = available_headroom - projected_expenses

        IF max_chunk > 0 AND target_debt.balance > 0:
            chunk = min(max_chunk, target_debt.balance)
            target_debt.balance -= chunk
            loc.balance += chunk
            record_chunk(current_date, chunk, target_debt, loc)

        // === DAILY SIMULATION (until next chunk opportunity) ===
        FOR each day in current cycle:
            // LOC daily interest (calculated on average daily balance)
            loc_daily_interest = loc.balance * loc.apr / 10000 / 365
            loc.accrued_interest += loc_daily_interest

            // Apply income deposits (reduce LOC balance)
            IF income_event on this day:
                loc.balance -= income_event.amount

            // Apply expense withdrawals (increase LOC balance)
            IF expense_event on this day:
                loc.balance += expense_event.amount

            record_daily(day, loc.balance, loc_daily_interest)

        // End of cycle: apply accrued LOC interest
        loc.balance += loc.accrued_interest
        loc.accrued_interest = 0

        // If target debt is paid off, select next target
        IF target_debt.balance <= 0:
            target_debt = next_debt_by_priority()

    RETURN timeline
```

### 5.4 Surplus Detection

```
FUNCTION compute_surplus(
    income_sources: [IncomeSource],
    fixed_expenses: [FixedExpense],
    debt_minimums_total: i64,
    safety_buffer: i64,
    one_time_events: [OneTimeEvent]
) -> SurplusResult:

    monthly_income = normalize_to_monthly(income_sources)
    monthly_expenses = normalize_to_monthly(fixed_expenses)

    one_time_net = sum(one_time_events where type=income) - sum(one_time_events where type=expense)

    gross_surplus = monthly_income - monthly_expenses - debt_minimums_total
    net_surplus = gross_surplus + one_time_net - safety_buffer

    RETURN SurplusResult {
        gross_surplus,
        net_surplus: max(0, net_surplus),
        is_actionable: net_surplus > 0,
        recommended_extra_payment: max(0, net_surplus),
    }
```

---

## 6. API Routes

All routes serve HTML (server-side rendered via Askama). HTMX attributes enable partial-page updates without full reloads.

| Method | Path | Handler | Description |
|--------|------|---------|-------------|
| `GET` | `/` | `dashboard::index` | Main dashboard with summary stats and nudge banner |
| `GET` | `/debts` | `debts::list` | List all debts |
| `GET` | `/debts/new` | `debts::new_form` | Form to add a new debt |
| `POST` | `/debts` | `debts::create` | Create a new debt record |
| `GET` | `/debts/{id}/edit` | `debts::edit_form` | Edit form for a specific debt |
| `PUT` | `/debts/{id}` | `debts::update` | Update a debt record |
| `DELETE` | `/debts/{id}` | `debts::delete` | Delete a debt record |
| `GET` | `/cashflow` | `cashflow::index` | View income sources and fixed expenses |
| `POST` | `/cashflow/income` | `cashflow::create_income` | Add an income source |
| `POST` | `/cashflow/expense` | `cashflow::create_expense` | Add a fixed expense |
| `PUT` | `/cashflow/income/{id}` | `cashflow::update_income` | Update income source |
| `PUT` | `/cashflow/expense/{id}` | `cashflow::update_expense` | Update expense |
| `DELETE` | `/cashflow/income/{id}` | `cashflow::delete_income` | Delete income source |
| `DELETE` | `/cashflow/expense/{id}` | `cashflow::delete_expense` | Delete expense |
| `GET` | `/upload` | `upload::form` | PDF upload page |
| `POST` | `/upload` | `upload::process` | Upload PDF, extract text, parse fields |
| `POST` | `/upload/confirm` | `upload::confirm` | Confirm parsed data and save to DB |
| `GET` | `/strategy` | `strategy::compare` | Strategy comparison view (Snowball vs Avalanche vs Velocity) |
| `GET` | `/strategy/{name}` | `strategy::detail` | Detailed amortization view for a specific strategy |
| `GET` | `/payments` | `payments::list` | Payment history |
| `POST` | `/payments` | `payments::record` | Record a new payment (triggers recalculation) |
| `GET` | `/settings` | `settings::index` | App settings page |
| `POST` | `/settings` | `settings::update` | Save settings |
| `GET` | `/api/health` | (inline) | Health check endpoint for Docker/monitoring |

---

## 7. Configuration

Configuration is loaded in priority order (highest wins):

1. **Environment Variables** (`SS_PORT`, `SS_DATA_DIR`, `SS_LOG_LEVEL`, etc.)
2. **`/data/config.toml`** (persistent user config in the Docker volume)
3. **Compiled Defaults**

### `config.example.toml`

```toml
[server]
host = "0.0.0.0"
port = 8080

[data]
# Directory for SQLite database, uploaded PDFs, and config
# In Docker, this is the volume mount point
dir = "/data"

[logging]
# trace, debug, info, warn, error
level = "info"

[security]
# Enable basic auth for the web UI
basic_auth_enabled = false
basic_auth_username = ""
basic_auth_password_hash = ""   # bcrypt hash

[notifications]
# Channel: none, discord, ntfy, gotify, smtp
channel = "none"
# Endpoint URL (webhook URL, ntfy topic URL, or SMTP JSON config)
endpoint = ""
# Include dollar amounts in notification messages?
include_amounts = false

[scheduler]
# How many days before a due date to send a reminder
reminder_days_before = 3
# Cron expression for surplus nudge check (default: daily at 8 AM)
surplus_check_cron = "0 8 * * *"
```

---

## 8. Docker Deployment

### `Dockerfile`

```dockerfile
# ---- Builder Stage ----
FROM rust:1.82-bookworm AS builder

WORKDIR /build
COPY Cargo.toml Cargo.lock ./
COPY src/ src/
COPY static/ static/
COPY templates/ templates/

RUN cargo build --release

# ---- Runtime Stage ----
FROM debian:bookworm-slim AS runtime

RUN apt-get update && apt-get install -y --no-install-recommends \
    ca-certificates \
    && rm -rf /var/lib/apt/lists/*

COPY --from=builder /build/target/release/snowball-samurai /usr/local/bin/snowball-samurai
COPY --from=builder /build/static /app/static

ENV SS_DATA_DIR=/data
ENV SS_PORT=8080

EXPOSE 8080
VOLUME ["/data"]

ENTRYPOINT ["snowball-samurai"]
```

### `docker-compose.yml`

```yaml
version: "3.8"
services:
  snowball-samurai:
    image: snowball-samurai:latest
    build: .
    container_name: snowball-samurai
    ports:
      - "8080:8080"
    volumes:
      - ./data:/data
    environment:
      - SS_DATA_DIR=/data
      - SS_PORT=8080
      - SS_LOG_LEVEL=info
    restart: unless-stopped
```

### Unraid Notes

On Unraid, the volume mapping would typically be:
```
/mnt/user/appdata/snowball-samurai:/data
```

---

## 9. Sub-Agent Task Decomposition

The project is divided into **5 independent work packages** that can be developed and tested in isolation. Each package has clear inputs, outputs, and test criteria.

### Task 1: Engine — Payoff Calculators
- **Files:** `src/engine/*`
- **Dependencies:** None (pure functions, no DB, no web)
- **Input:** Vec of Debt structs + monthly surplus amount
- **Output:** PayoffSchedule struct
- **Tests:** Unit tests with known debt scenarios verifying month-by-month balances, total interest, and payoff dates against hand-calculated expected values.
- **Estimated Complexity:** Medium

### Task 2: Database — Schema & CRUD
- **Files:** `src/db/*`
- **Dependencies:** None
- **Input:** rusqlite connection
- **Output:** Type-safe CRUD functions
- **Tests:** Integration tests creating an in-memory SQLite DB, inserting records, querying, updating, deleting.
- **Estimated Complexity:** Low-Medium

### Task 3: Ingestion — PDF Parsing
- **Files:** `src/ingestion/*`
- **Dependencies:** None (operates on byte slices)
- **Input:** PDF file bytes
- **Output:** ParsedStatement struct (or extraction failure with raw text)
- **Tests:** Unit tests against embedded test PDF byte arrays.
- **Estimated Complexity:** Medium

### Task 4: Web — Actix Handlers & Templates
- **Files:** `src/web/*`, `static/*`, templates in `src/web/templates/*`
- **Dependencies:** Tasks 1, 2, and 3 (imports their types and functions)
- **Input:** HTTP requests
- **Output:** Rendered HTML responses
- **Tests:** Integration tests using Actix test server.
- **Estimated Complexity:** Medium-High

### Task 5: Infrastructure — Notifications, Scheduler, Docker
- **Files:** `src/notifications/*`, `src/scheduler/*`, `Dockerfile`, `docker-compose.yml`
- **Dependencies:** Tasks 1 and 2 (scheduler calls engine functions and queries DB)
- **Input:** Cron schedule + DB state
- **Output:** Notification dispatches
- **Tests:** Unit tests for notification payload formatting. Manual Docker build & run test.
- **Estimated Complexity:** Medium

### Dependency Graph

```
Task 1 (Engine) ────────────────┐
                                 │
Task 2 (Database) ──────────────┼──→ Task 4 (Web / UI)
                                 │
Task 3 (Ingestion) ─────────────┘
                                 │
Task 1 (Engine) ────────────────┤
                                 └──→ Task 5 (Notifications / Docker)
Task 2 (Database) ──────────────┘
```

**Tasks 1, 2, and 3 can be developed in parallel.**
Task 4 depends on all three being complete.
Task 5 depends on Tasks 1 and 2.

---

## 10. Development Workflow

### Phase 1: Foundation (Tasks 1 + 2 + 3 in parallel)
All sub-agents work simultaneously on independent modules. Each produces a self-contained Rust module with passing `cargo test`.

### Phase 2: Integration (Task 4)
The web layer is built by importing modules from Phase 1. The Actix server boots, serves the dashboard, and wires up form submissions to the database and engine.

### Phase 3: Deployment (Task 5)
Background scheduler and notifications are integrated. The Dockerfile is created and tested. The app is verified running in Docker with persistent `/data` volume.

### Phase 4: Polish & Iterate
User testing with real statements. Parser profile refinement. UI polish. Additional notification channels.
