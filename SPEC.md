# Snowball Samurai — Product Specification

> **Version:** 0.1.0-draft
> **Last Updated:** 2026-10-11
> **Status:** Architecture Phase

---

## 1. Vision & Purpose

Snowball Samurai is a **self-hosted, privacy-first personal finance application** built in Rust that helps users aggressively pay down debt using proven strategies. Users upload their financial statements (PDF bank statements, credit card bills, paychecks), and the application:

1. **Extracts** key financial data from those documents automatically.
2. **Models** their complete debt picture (balances, APRs, minimums, due dates).
3. **Calculates** optimal payoff plans using Snowball, Avalanche, or Velocity Banking strategies.
4. **Instructs** the user on exactly how much to pay, to which account, and when.
5. **Nudges** the user when surplus cash is detected ("You have ~$340 extra this cycle — apply it to Visa ending 4821").

The application runs as a Docker container (ideal for Unraid) and is accessed via any browser on the local network. **No financial data ever leaves the user's network.**

---

## 2. Target User Profile

- Individuals with multiple debts (credit cards, auto loans, student loans, personal loans, medical bills).
- Comfortable self-hosting Docker containers (e.g., Unraid, Synology, bare Linux server).
- Privacy-conscious: unwilling to share financial data with cloud services.
- No coding experience; relies on the application's UI for all interactions.

---

## 3. Functional Requirements

### 3.1 Document Ingestion

| ID | Requirement | Priority |
|----|-------------|----------|
| ING-01 | Accept PDF file uploads via browser drag-and-drop or file picker. | P0 |
| ING-02 | Extract embedded text from digital PDFs (not scanned images) using a pure-Rust PDF text extraction library. | P0 |
| ING-03 | Apply configurable regex-based "parser profiles" to extract structured fields from the raw text: Statement Date, Due Date, Current Balance, Minimum Payment Due, APR/Interest Rate, Account Number (last 4 digits), and Institution Name. | P0 |
| ING-04 | Ship with built-in parser profiles for common US statement formats (Chase, Capital One, Discover, Amex, generic). Users can define additional profiles via a simple pattern-matching UI or TOML config file. | P1 |
| ING-05 | Display the raw extracted text alongside the parsed fields so the user can verify and correct any misreads before saving. | P0 |
| ING-06 | Store the original uploaded PDF in a configurable data directory for audit trail purposes. | P2 |
| ING-07 | Accept CSV import for bulk data entry (debts, income, expenses). | P1 |

### 3.2 Debt Management

| ID | Requirement | Priority |
|----|-------------|----------|
| DEB-01 | Maintain a registry of all active debts. Each debt record contains: Nickname, Institution, Account (last 4), Type (credit card, auto loan, student loan, personal loan, mortgage, medical, HELOC/PLOC, other), Current Balance, APR, Minimum Payment, Due Date (day of month), and Status (active, paid off). | P0 |
| DEB-02 | Support manual entry/editing of debt records in addition to PDF-extracted data. | P0 |
| DEB-03 | Track payment history: every payment made is logged with Date, Amount, Debt ID, and whether it was a minimum or extra payment. | P0 |
| DEB-04 | Mark debts as "Paid Off" when balance reaches $0. Archive them but keep history. | P0 |
| DEB-05 | Support recurring debts that are not "payoff-able" (e.g., rent, utilities, subscriptions) as fixed expenses in the cash flow model. | P0 |

### 3.3 Income & Cash Flow

| ID | Requirement | Priority |
|----|-------------|----------|
| CFW-01 | Track recurring income sources: Name, Amount (net take-home), Frequency (weekly, biweekly, semi-monthly, monthly), and Next Pay Date. | P0 |
| CFW-02 | Track recurring fixed expenses (rent, utilities, insurance, subscriptions, groceries estimate, fuel estimate). | P0 |
| CFW-03 | Compute "Disposable Surplus" per pay cycle: `Income - Fixed Expenses - Sum(Debt Minimums) = Surplus available for extra debt payments`. | P0 |
| CFW-04 | Allow one-time income events (tax refund, bonus, side hustle payout) and one-time expenses (car repair, medical bill) to be entered and factored into the current cycle. | P1 |
| CFW-05 | Maintain a configurable "Safety Buffer" amount the user wants to keep as a cash cushion (default: $500). Surplus calculations never recommend paying more than would breach this buffer. | P0 |

### 3.4 Payoff Strategy Engines

#### 3.4.1 Debt Snowball

| ID | Requirement | Priority |
|----|-------------|----------|
| SNO-01 | Order all active debts by **ascending current balance** (lowest first). Ties broken by highest APR. | P0 |
| SNO-02 | Allocate minimum payments to all debts. Route the entire remaining surplus to the top-priority (lowest balance) debt. | P0 |
| SNO-03 | When the top-priority debt is paid off, "roll" its minimum payment + the surplus into the next debt in the ordered list. | P0 |
| SNO-04 | Generate a month-by-month amortization schedule showing: each debt's projected balance, payment amount, interest accrued, and the projected payoff date. | P0 |
| SNO-05 | Display total interest paid across the life of the plan and the final debt-free date. | P0 |

#### 3.4.2 Debt Avalanche

| ID | Requirement | Priority |
|----|-------------|----------|
| AVA-01 | Order all active debts by **descending APR** (highest first). Ties broken by lowest balance. | P0 |
| AVA-02 | Same allocation, rollover, and reporting mechanics as Snowball (SNO-02 through SNO-05), but with Avalanche ordering. | P0 |

#### 3.4.3 Velocity Banking

| ID | Requirement | Priority |
|----|-------------|----------|
| VEL-01 | Require the user to designate one debt as a "Line of Credit" (HELOC or PLOC) with its credit limit, current balance, and APR. This is the "velocity account." | P0 |
| VEL-02 | Model the "chunk" transfer: a lump sum is drawn from the LOC and applied to a target debt's principal. The LOC balance increases by the chunk amount. | P0 |
| VEL-03 | Model income parking: all income is deposited into (applied against) the LOC balance during the pay cycle, reducing average daily balance and therefore daily interest. | P0 |
| VEL-04 | Model expense outflow: living expenses are drawn from the LOC, increasing its balance. | P0 |
| VEL-05 | Calculate the optimal chunk size: `LOC Credit Limit - Safety Reserve - Projected Expenses Until Next Income = Max Chunk`. | P0 |
| VEL-06 | Simulate day-by-day LOC balance fluctuations over a full payoff timeline to show the user exactly how income deposits reduce average daily interest vs. a traditional monthly payment. | P1 |
| VEL-07 | Generate a cycle-by-cycle plan: "On [date], transfer $X from LOC to [Target Debt]. Your LOC balance will be $Y. Over the next [N] days, your income deposits will bring it down to $Z." | P0 |
| VEL-08 | Warn the user if velocity banking is not advantageous (e.g., LOC APR >= target debt APR, or surplus is too small to benefit from daily interest reduction). | P0 |

#### 3.4.4 Strategy Comparison

| ID | Requirement | Priority |
|----|-------------|----------|
| CMP-01 | Present a side-by-side comparison of all three strategies showing: Total Interest Paid, Debt-Free Date, and Monthly Cash Flow Impact. | P0 |
| CMP-02 | Highlight the recommended strategy based on user's debt profile (with explanation of why). | P1 |

### 3.5 Payment Schedule & Reminders

| ID | Requirement | Priority |
|----|-------------|----------|
| REM-01 | Generate a concrete payment calendar: "On [date], pay $[amount] to [debt name]." | P0 |
| REM-02 | Detect when the user records a paycheck or income event and compute whether surplus cash is available above the safety buffer. If so, display a prominent nudge: "You should have ~$X extra. Go ahead and pay this to [highest-priority debt]." | P0 |
| REM-03 | Support push notifications via configurable channels: Discord webhook, Telegram bot, Gotify, Ntfy, or email (SMTP). At least one channel must be implemented in v0.1. | P1 |
| REM-04 | Run a background scheduler (Tokio-based cron) that fires reminders N days before each debt's due date (configurable, default: 3 days). | P1 |
| REM-05 | After a payment is recorded, recalculate the entire payoff timeline and update the dashboard projections. | P0 |

### 3.6 Dashboard & Reporting

| ID | Requirement | Priority |
|----|-------------|----------|
| DSH-01 | Display a summary dashboard showing: Total Debt, Total Minimum Payments, Monthly Surplus, Projected Debt-Free Date, Current Strategy, and Next Recommended Action. | P0 |
| DSH-02 | Show a visual debt payoff progress chart (stacked bar or line chart showing each debt's balance over time). | P1 |
| DSH-03 | Show a "Debt Thermometer" or progress indicator for each individual debt (% paid off). | P1 |
| DSH-04 | Provide a detailed amortization table view for each debt. | P0 |
| DSH-05 | All monetary values default to USD but support a configurable currency symbol. | P2 |

---

## 4. Non-Functional Requirements

### 4.1 Privacy & Security

| ID | Requirement | Priority |
|----|-------------|----------|
| SEC-01 | The application makes **zero outbound network connections** by default. All processing is local. | P0 |
| SEC-02 | If notification channels are enabled (Discord, Telegram, email), the app connects **only** to the user-configured endpoint. The notification payload contains only the reminder text, never raw financial data (no balances, account numbers, or amounts in notification bodies unless explicitly opted in). | P0 |
| SEC-03 | The SQLite database file may optionally be encrypted at rest using SQLCipher (or Rust equivalent). | P2 |
| SEC-04 | The web UI is served on `0.0.0.0:<port>` with no authentication by default (LAN-only trust model). Optional basic auth can be enabled via config. | P1 |

### 4.2 Deployment

| ID | Requirement | Priority |
|----|-------------|----------|
| DEP-01 | Ship as a single Docker image. Multi-stage build: compile in a Rust builder image, run in a minimal `scratch` or `alpine` image. | P0 |
| DEP-02 | All persistent data (SQLite DB, uploaded PDFs, config) stored under a single volume mount: `/data`. | P0 |
| DEP-03 | Configurable via environment variables and/or a `config.toml` file in the `/data` volume. | P0 |
| DEP-04 | The Docker image should support `linux/amd64` and `linux/arm64` architectures. | P1 |
| DEP-05 | Provide an Unraid Community Applications XML template for one-click install. | P2 |

### 4.3 Performance

| ID | Requirement | Priority |
|----|-------------|----------|
| PRF-01 | The compiled binary should be under 20 MB. | P1 |
| PRF-02 | Idle memory usage should be under 30 MB. | P1 |
| PRF-03 | PDF text extraction should complete in under 2 seconds for a typical 3-page statement. | P0 |
| PRF-04 | Payoff simulations for up to 30 debts over 30 years should complete in under 1 second. | P0 |

---

## 5. Out of Scope (v0.1)

The following are explicitly **not** in scope for the initial release:

- Live bank account connections (Plaid, MX, Finicity).
- Mobile native app (use the responsive web UI from a phone browser).
- Multi-user / multi-household support.
- Investment tracking or net worth calculations.
- Tax optimization or tax-related features.
- AI-powered natural language chat interface.
- Scanned/photographed document OCR (only digital PDF text extraction).

---

## 6. Glossary

| Term | Definition |
|------|-----------|
| **Snowball Method** | Debt repayment strategy that prioritizes paying off the smallest balance first to build psychological momentum, while making minimum payments on all other debts. |
| **Avalanche Method** | Debt repayment strategy that prioritizes paying off the highest APR debt first to minimize total interest paid. |
| **Velocity Banking** | Strategy using a Line of Credit (HELOC/PLOC) as a "float account." Lump sums are drawn from the LOC to pay down high-interest debts, while income is parked in the LOC to reduce its average daily balance and therefore daily interest charges. |
| **Chunk** | In velocity banking, a lump-sum transfer from the LOC to a target debt. |
| **Surplus / Disposable Surplus** | The amount of cash remaining after all minimum debt payments and fixed living expenses are subtracted from income for a given pay cycle. |
| **Safety Buffer** | A user-defined minimum cash reserve that the system will never recommend spending. |
| **Parser Profile** | A named set of regex patterns and field mappings that instruct the PDF ingestion engine how to extract structured data from a specific institution's statement format. |
