# Adeel Traders — Implementation Roadmap v2 (draft for review)

> **Status:** DRAFT. Nothing in this roadmap is started until it is approved.
> **Date:** 11 Nov 2026 · **Checked against:** SRS v1.0 (28 Oct 2026, all 22 pages), our code in
> `apps/adeel_traders` and `web/`, the live dev site `adeel.localhost`, and the installed
> Frappe 16.56 / ERPNext 16.54 / HRMS 16.53 / Lending 16.8 / Webshop / Payments source.
> **Replaces:** Part 6 (phases) and 19.3 (checklist) of `IMPLEMENTATION_PLAN.md` once approved.
> The other parts of that plan (architecture, security, testing, migration) stay valid.

---

## 0. How to read this

| Icon | Meaning |
|---|---|
| ✅ | **Built-in**: Frappe/ERPNext already does it. We only use it. |
| ⚙️ | **Configuration**: built-in, but we must set it up (settings, masters, workflow) through a patch or fixture in our app. |
| 🔧 | **Customization**: small change on a standard doctype (custom field, hook, property setter, print format). No core change. |
| 🔴 | **Custom development**: new doctype, new API, new business logic, new integration. |
| 🟩 | Next.js screen (staff app or storefront). |
| 🟦 | Frappe Desk screen (standard ERPNext UI). |

**Three layers for every step** (requirement 10):

| Layer | Where | Who owns it |
|---|---|---|
| **ERPNext** | Standard doctypes, reports, ledgers, stock, workflow engine | Frappe (never changed) |
| **Our backend** | `apps/adeel_traders` (business rules, APIs, setup patches, tests) and `apps/pk_integrations` (PostEx, gateway, attendance machine) | Us |
| **Next.js** | `web/apps/staff` (staff portal) and `web/apps/storefront` (website) | Us; shows what the API returns, no business logic |

**Rule:** if a value affects money, stock, permission or approval, it is decided in the backend
and covered by a Python test. Next.js only displays it.

---

## 1. Why the old plan kept "missing" things (root causes)

| # | Root cause | How this roadmap fixes it |
|---|---|---|
| R1 | Phases were grouped by **module**, not by **dependency**. Shared things (approvals, audit, export, email, print) were left to Phase 9, but every module needs them. | Section 5 lists all shared infrastructure. It is built in **Phase 1**, before any module. |
| R2 | Accounting setup was too thin (one cash account, no wallets, no tax decision, channel not filled), but purchasing and sales post to it. | **Phase 2** finishes the accounting foundation before the first transaction. |
| R3 | The plan was written for Frappe Desk, but we build screens in Next.js. Steps had no "Next.js work". | Every step has 3 columns: ERPNext / our backend / Next.js. |
| R4 | "Done" meant "the screen exists". | Section 3: a **phase gate** with 12 checks (backend, permissions, ledgers, reports, tests, UI, docs). |
| R5 | The SRS checklist was not linked to steps. | Section 8: **traceability matrix** — every SRS line → phase → step ID. |
| R6 | Some setup was coded but never applied to the existing site (e.g. default warehouse). | Gate check G2: every setting must apply through a **patch** on an existing site **and** on a fresh install. |
| R7 | Decisions were left open while building on top of them. | Section 4: decisions list. Each one names the phase it blocks. A phase does not start while a blocking decision is open. |

---

## 2. Audit of what exists today (real code and real site, not the plan)

### 2.1 Built and verified ✅

| Area | What exists | Evidence |
|---|---|---|
| Platform | Frappe + ERPNext + HRMS + Lending + Webshop + Payments v16 installed; apps `adeel_traders`, `pk_integrations` (empty) | `bench list-apps` |
| Setup as code | Units, item groups, price lists (Wholesale, Online, Self-delivery), customer groups → price list, 5 warehouses, Sales Channel doctype + accounting dimension, stock settings (reservation on, negative stock off, FEFO, FIFO, auto reorder request, reorder email), batch tracking on, back-dated rule, item attributes (Shade, Size, Fragrance), stock entry types with reason | `setup/data.py`, `setup/core_setup.py`, patches v0_1–v0_4 |
| Roles | Roles "Stock Counter", "Online Order Team"; 7 role profiles "AT Owner … AT Online Order Team" | `fixtures/role.json`, `role_profile.json` |
| Company | Adeel Traders, PKR, Pakistan, fiscal year Jul–Jun 2026-27, perpetual inventory on | live site |
| Products (API + 🟩) | List with status tabs, search, filters (category, brand), hero stock value by location, restock card; product page with prices + margin, units, variants (any attribute), brand/description/photo, gift sets (builder, sets you can pack, limiting part, cost), stock by location, move/adjust stock with reason, stock history, barcode generate + label print | `api/v1/products.py`, `app/(app)/products/*` |
| Catalogue (API + 🟩) | Categories and brands: add, rename, delete brand if unused | `api/v1/catalogue.py`, `settings/catalogue` |
| Stock count (🔴 doctype + API + 🟩) | Scan to count, blind count enforced on server, difference report, full count (unscanned = 0), export/print, post → ERPNext Stock Reconciliation, cancel | `doctype/stock_count`, `api/v1/stock_count.py` |
| Staff app shell | Login, session + CSRF proxy, sidebar, ⌘K palette, design system (DESIGN.md), form kit | `web/apps/staff` |
| Tests | 44 backend tests pass; pre-commit clean | `bench run-tests --app adeel_traders` |

### 2.2 Partly built 🟡

| Area | What is missing |
|---|---|
| Excel bulk upload/download (SRS 3.1) | Only CSV templates and a guide (`docs/DATA_IMPORT.md`). No screen, no download of current products and prices. |
| Minimum stock levels (SRS 3.3 "managed in Settings") | Set one product at a time on the product page. No Settings screen, no bulk edit. |
| Low-stock alerts (SRS 3.12) | ERPNext reorder email is on, but **no outgoing email account exists**, so nothing is sent. No in-app alert. |
| Stock value "for each branch" | Shown per **location** (warehouse). There is no "branch" grouping yet (branches not set up). |
| Label printing | Browser print works. Not tested on the client's label printer (Q7). |
| Gift sets | "Stock is deducted from each item" can only be proven after a sale exists (Phase 6). |
| Buy and sell in any unit | Units exist on items. Only proven after purchase (Phase 5) and sale (Phase 6). |
| Permission in the UI | `session.me` returns only `can_view_cost`. Pages use their own `can` flags. There is no shared permission map for menus and buttons. |
| Sales Channel | Dimension exists, but **nothing fills it** (spec 7.16 not built). Must work before the first sale. |
| Storefront | Only the empty Next.js starter page. |

### 2.3 Not built ❌ (by SRS area)

3.1 website flags, Excel import screen · 3.2 discounts and role limits, dealer prices, coupons · 3.3 opening stock screen, stock reports ·
3.5 purchasing (all) · 3.6 customers and dealers (all) · 3.7 sales (all) · 3.8 online orders (all) · 3.9 accounts (all) ·
3.10 HR and payroll (all) · 3.11 website (all) · 3.12 communication (all) · 3.13 users/roles/permissions screens, change history, branches, numbering, tax, payment methods, delivery charges, login security screens · 3.14 dashboards (all) · 4 reports (all) · 5 integrations (all).

### 2.4 Wrong or inconsistent today ⚠️ (must be fixed in Phase 0)

| # | Problem | Why it matters |
|---|---|---|
| I1 | Company **default warehouse is still "Stores - AT"**. The fix (`ensure_default_warehouse`) was added to `run_setup`, but no new patch runs it, so the existing site never got it. | New receipts and entries default to a warehouse Adeel Traders does not use. Same bug can happen to any setting. |
| I2 | ERPNext's own warehouses (**Stores, Finished Goods, Work In Progress, Goods In Transit**) are active and **show in our Move stock and Count pickers** (`form_options` lists all warehouses). | Staff can move stock into a warehouse nobody looks at. |
| I3 | Modes of payment are ERPNext samples (Cheque, Cash, Credit Card, Wire Transfer, Bank Draft). **No JazzCash, Easypaisa, Bank Transfer, PostEx COD.** Only one account ("Cash - AT"). No bank or wallet accounts, no default bank account. | Sales, purchasing and POS cannot record payments the SRS asks for. |
| I4 | A tax template "Pakistan Tax - AT" was created by the setup wizard. Nobody reviewed it. GST decision (Q5) is open. | Wrong tax on the first invoice. |
| I5 | Sales Channel dimension is not filled on any document (7.16 missing). | Channel reports (SRS 4) will be empty for every sale made before the fix. ERPNext does not fill old documents. |
| I6 | Change tracking is **off** for Brand, Item Group, Product Bundle, Pricing Rule, Stock Reconciliation. | "Record of every change" (SRS 3.13) is incomplete. |
| I7 | Session expiry is **170 hours** (7 days). 2FA is off. "Login with email link" is on. | SRS 3.13 asks for automatic logout and login security. |
| I8 | No outgoing email account. | Reorder emails, alerts, statements and order emails cannot be sent. |
| I9 | Selling Settings default price list = "Standard Selling" (a 4th, unused price list). Default customer group is empty. | A sale with no customer type picks the wrong price. |
| I10 | Letter heads are ERPNext samples. No print formats of our own. Naming series are ERPNext defaults (`ACC-SINV-.YYYY.-`). | Changing numbering after real documents exist is painful. |
| I11 | `ARCHITECTURE_REVIEW.md` says store, accounts, HR and POS use Desk (hybrid). We built store screens (stock count) in Next.js, and the working rule is "the user tests only in our UI". | The plan and the code disagree about which UI each role uses (decision D1). |
| I12 | Plan lists the **Builder** and **offsite_backups** apps. **They are not installed.** | Website home sections (SRS 3.11) and daily off-site backups (SRS 6) have no tool yet. |
| I13 | Role "Stock Counter" still has a custom permission on **Stock Reconciliation** from before we built the Stock Count doctype. | A leftover permission that bypasses our blind count if used from Desk. |
| I14 | All our code is **uncommitted** (only the "Initialize App" commit). No CI. | One bad command loses weeks of work. No automatic test run. |
| I15 | `IMPLEMENTATION_PLAN.md` Phase 2 note says "Done" for items that are partial (see 2.2). | Status reports are not trustworthy. |
| I16 | Demo data on the site: items `temp`, `test` and `1` (unit "Nos"), posted test count STC-2026-00001, extra "Finished Goods" warehouse. | DESIGN.md rule 10: no dummy data in a demo. |
| I17 | ✅ **Partly fixed 12 Nov 2026:** Settings › Units now keeps the usual size of each pack in Frappe's own UOM Conversion Factor (a **Dozen = 12 Pieces for every product**, "Carton = 12 Boxes" is worked out from the sizes; ERPNext reads it when a product has no size of its own, and a product's own size wins). Still open (P3.11): the product page's "Add unit" only asks for pieces, and Box/Carton sizes per product. Was: Units: only **Box = 12** is set (5 items). **Dozen and Carton have no conversion on any item**, and "Add unit" only asks for pieces (no "Carton = 12 Boxes"). I wrongly marked SRS 3.2-1 as built. | Buying by carton (Phase 5) would fail or need manual setup per item. |
| I18 | Import templates (`docs/import-templates`) have **no Brand column**, and the site has **0 brands**. | Brands could not be imported with the catalogue. |

---

## 3. Phase gate — when a phase is "complete"

A phase is complete **only when all 12 checks pass**. Screens alone are never enough.

| # | Check | How we prove it |
|---|---|---|
| G1 | Every SRS line assigned to the phase (section 8) is ✅ | Traceability table updated with the test or screen that proves it |
| G2 | All setup is code: doctypes, custom fields, permissions, settings, workflows come from our app (fixtures or patches) | Run on a **fresh site** and **migrate the existing site**; both give the same result |
| G3 | Backend tests pass: every API, every business rule, and **permission-denied cases** for each role | `bench run-tests --app adeel_traders` green, pre-commit clean |
| G4 | Ledgers are right: every transaction type in the phase posts the correct stock ledger and general ledger (with Sales Channel and branch) | A written "posting table" per transaction, checked by a test |
| G5 | Roles: each SRS role tried (allowed and denied), including branch restriction | Permission test matrix |
| G6 | Shared infrastructure is used, not rebuilt (approvals, audit, export, print, email, scan) | Code review checklist |
| G7 | The reports for this phase exist, filter by date/branch/channel/category/brand where the data has them, and export to Excel/PDF | Each report opened and exported once |
| G8 | UI: 1440 px and 390 px, light and dark, loading/empty/error states, DESIGN.md checklist | Headless browser run with screenshots |
| G9 | Integrations of the phase work against sandbox (with retries and logs) | Integration test + log check |
| G10 | One **end-to-end business flow** per phase passes (named in the phase) | Script run, test data removed after |
| G11 | Docs updated: this roadmap, DESIGN.md patterns, the user guide section for each role in the phase | Diff reviewed |
| G12 | Code committed, CI green, no blocking decision open (or a written assumption accepted by the owner) | Git log, CI, decisions table |

**Before a phase starts (entry check):** all its prerequisites (listed per phase) are ✅, its blocking decisions are closed,
and we list what later phases need from it, so we build it once.

---

## 4. Decisions to close (each blocks a phase)

| # | Decision | Options | My recommendation | Blocks |
|---|---|---|---|---|
| D1 | **Which UI each role uses** ✅ **CLOSED (11 Nov 2026): (b)** | (a) Next.js for every staff screen · (b) hybrid: Next.js for daily work, Desk for expert setup (chart of accounts tree, print designer, workflow designer, deep reports) | (b) with **every SRS action available in Next.js**; Desk only for Nesway admins and the accountant's expert tools. Needs a written update of `ARCHITECTURE_REVIEW.md` | Phase 1 |
| D2 | **POS** | (a) ERPNext POS, branded · (b) our Next.js POS on ERPNext's POS invoice and closing APIs | Decide after a 1-day test of ERPNext POS with the client's scanner and printer (Phase 0 PoC) | Phase 6 |
| D3 | Client approval of technology change (Q17) and accountant approval of the accounting design (Q18) | — | Get both in writing | Phase 2 |
| D4 | **Branches** (Q1) | — | ✅ **CLOSED (11 Nov 2026):** one company, **2 branches to start: Main Warehouse and Shop Counter**. Each has a Branch, a Cost Center and its warehouse(s). Online, wholesale and self-delivery orders belong to Main. A new branch later = 1 Branch + 1 Cost Center + 1 warehouse + users, no redesign. Client can correct the number later | Phase 1 (step P1.4) |
| D5 | **GST** (Q5): tax on invoices or not, NTN/STRN | — | Must be answered; "no tax" is also a valid answer | Phase 2 |
| D6 | **Numbering** for products (Q22) and documents (invoice, PO, receipt…) | — | Short prefixes per document and branch, e.g. `SI-SHOP-26-00001` | Phase 2 (before any real document) |
| D7 | Valuation **FIFO** and fiscal year **Jul–Jun** (Q12, Q26) | FIFO / Moving average | FIFO (already set). Hard to change later | Phase 2 |
| D8 | **Damaged stock** account (Q23) | Loss when moved to Damaged / only when written off | Loss only when written off (current) | Phase 2 |
| D9 | **Below-cost sale**: SRS says "warning", ERPNext setting **blocks** it | Block / warn and ask a manager approval | Warn + manager approval (uses approvals engine) | Phase 6 (rule recorded in Phase 2) |
| D10 | **City** model: Territory = city, or city on Address only | — | Territory = city (needed by delivery charges, sales by city, PostEx city list) | Phase 2 |
| D11 | **Expense entry** model (SRS 3.9) | (a) Journal Entry with a template · (b) small custom "Expense" doctype that posts a Journal Entry after approval · (c) Purchase Invoice | (b): simple for staff, bill photo and approval built in | Phase 2 (accounts list) / Phase 7 (screen) |
| D12 | Self-delivery price for walk-in customers (Q2) | — | — | Phase 2 |
| D13 | Discount above the limit at the counter (Q19) | Manager PIN on the till / manager approves from inbox | Manager PIN for the counter, inbox for orders | Phase 6 |
| D14 | Website home sections editor (Builder not installed) | Install Builder / our own "Home Section" doctype + Next.js | Own doctype: we render the storefront in Next.js anyway | Phase 9 |
| D15 | Daily off-site backups (offsite_backups not installed) | Install app / server-level backup to S3-compatible storage | Server-level backup + restore test | Phase 13 (hosting decided in Phase 0) |
| D16 | Email account for sending (client) | Gmail/Workspace, own domain SMTP | ⏳ **PENDING by owner's choice (11 Nov 2026), do later.** Phase 1 built the email templates, the bell and the Email settings page (P1.7). Frappe refuses to send or queue any email without an account, so every Email button stays hidden until the account is added. Real sending is switched on when the account is ready (before the first real alert or customer email, latest Phase 3 low-stock alert test) | Phase 1 (build) / go-live of real sending |
| D17 | Label size and printer (Q7), receipt printer | — | — | Phase 3 (labels), Phase 6 (receipts) |
| D18 | Slow / dead stock days (Q20) | 60 / 180 | 60 / 180 | Phase 3 |
| D19 | Guest checkout (Q14), payment provider (Q4), PostEx account (Q11), attendance machine (Q9), HR policies (Q10) | — | — | Phases 9, 10, 8, 11, 11 |
| D20 | **Staff and roles** (Q8) | — | ✅ **CLOSED (11 Nov 2026):** the **7 staff roles of SRS 2.1** (Owner, Manager, Counter Staff, Store, Accountant, HR, Online Order Team; the 7 role profiles already exist) + Customer on the website. Riders are **not** logins: they are Driver records (cash settlement is per rider, P6.5); a Rider login can be added later if wanted. User count is open: the owner adds real users any time on the Users page (no per-user fee), and can make custom roles (P1.2) | Phase 1 (P1.1, P1.3) |
| D21 | **Dealer credit** (Q3): do dealers buy on credit; credit limit and credit days per dealer | — | Credit limit + payment terms per dealer | Phase 2 (P2.9 payment terms), Phase 4 (P4.2) |
| D22 | **Number of products** (Q6) incl. shades and sizes | — | Needed for import batch size and the performance target | Phase 3 (P3.1), Phase 13 (P13.4) |
| D24 | **Hosting** (Q13): own server or Frappe Cloud | — | — | Phase 0 (decide), Phase 13 (set up) |
| D25 | **Old WooCommerce orders** (Q15): keep history or not | — | Keep customers; old orders as read-only archive if needed | Phase 13 (P13.1) |
| D26 | **Brand guidelines** (Q16): logo files, colours, fonts | — | — | Phase 1 (P1.4, P1.7, P1.8: logo on prints and emails), Phase 9 |
| D27 | Confirm **back-dated entries** only by Stock Manager (Q25, already set) and "any other business rules" (SRS 8) | — | — | Phase 0 |
| D23 | **Unit sizes** per product: pieces in a box, boxes in a carton (do they differ by product?) | Same for all / per product | Per product (current design); Dozen = 12 for all | Phase 3 (P3.11) |

**Choices Nesway made while the owner was away (12 Nov 2026), all easy to change:**
Change history is for Owner and Accountant · approval limits come with each phase (none invented) ·
WhatsApp share links last 7 days · exports stop at 10,000 rows · numbering is view-only until D6 ·
managers may run stock reports · counters may print but not email a count.
**Sync 12 Nov 2026 (SRS 2.1):** every page added in P1.5–P1.12 is a tick or switch on the Roles page, not fixed in code (areas Reports, Units, Payment methods, Email, Integrations, Numbering; switches Change history, Approvals). Raw read access to Frappe's change records, given to two roles in the first P1.5, was removed. The owner's "Try again" on a failed integration call needed a missing permission: fixed.

**Rule while a client answer is late:** we may continue with the recommended default only if the owner accepts it in writing
and the rework risk is written next to the step.

---

## 5. Shared infrastructure (built once, used by many)

| ID | Shared piece | ERPNext gives | We build (backend) | We build (Next.js) | Built in | Used by |
|---|---|---|---|---|---|---|
| S1 | **Permission model** | Roles, Role Profiles, Module Profiles, DocPerm, User Permissions (branch, warehouse) ✅ | Complete role set as fixtures; `session.me` returns roles + a **capability map** ("can create PO", "can approve discount", "can see cost"…); `get_list` only (never `get_all`) for user data | `useCan()` hook; menu, buttons and pages hidden by capability; "no access" page | P1.1 | All |
| S2 | **Custom roles screen** (SRS 2.1) | Role + Custom DocPerm ✅ | API to create a role and set view/create/edit/delete/submit/cancel/approve/export per area; guard rails (cannot lock out the owner, cannot edit system roles) | Roles & permissions page | P1.2 | Owner |
| S3 | **Users and login security** | User, password policy, session expiry, login attempts, 2FA, Activity Log ✅ | API: invite, assign role profile + branch, disable, reset password; settings patch | Users page, login history page | P1.3 | All |
| S4 | **Company, branches, branding** | Company, Branch, Cost Center, Warehouse, Letter Head ✅ | Branch → cost center + warehouses + user permission; settings patch | Company & branches settings | P1.4 | All |
| S5 | **Change history (audit)** | Version (track changes), Activity Log, Access Log ✅ | Turn on tracking for every business doctype; `get_changes(doctype, name)` and a global log API (who, what, when, old → new) | Reusable "History" panel + Audit log page | P1.5 | All |
| S6 | **Approvals engine** | Workflow, Workflow Action ✅ | One API: my pending approvals, approve/reject with comment, approval history; rule helpers (amount/discount/credit thresholds) | Approvals inbox + "Waiting for approval" banner component | P1.6 | Discounts, below-cost, credit, PO, expenses, payroll |
| S7 | **Email and notifications** | Email Account, Email Template, Notification, Notification Log, Email Queue ✅ | Branded base email template with logo; notification helper; email queue health check | Bell with in-app notifications; email template editor in Settings | P1.7 | Alerts, statements, orders, payslips, daily summary |
| S8 | **Print and PDF** | Print Format (Jinja), Letter Head, PDF download ✅ | Base A4 layout and base 80 mm layout with logo; PDF endpoint with permission check; public signed link for "send on WhatsApp" | "Print / PDF / Email / Share" action component | P1.8 | Labels, invoices, challans, receipts, PO, statements, payslips |
| S9 | **Export any list** (SRS 6) | Data Export, report export ✅ | One export API (list → Excel), same filters and permissions as the screen, column whitelist | "Export" button in the table kit | P1.9 | Every list and report |
| S10 | **List and report kit** | Report API, prepared reports ✅ | Report wrapper pattern: standard ERPNext report → our API with filters date/branch/channel/category/brand | DataTable (sticky header, paging, saved filters), report page layout, charts | P1.10 | Every list and report |
| S11 | **Integration framework** | Background jobs, Scheduled Job Type, Integration Request log ✅ | Base for `pk_integrations`: job with retry, idempotency key, request/response log, error alert | Integration status page | P1.11 | PostEx, gateway, attendance machine |
| S12 | **Settings and masters kit** | UOM, Mode of Payment, Naming Series ✅ | Generic "simple master" API (list/add/rename/disable when unused) | Settings pages following the Catalogue pattern | P1.12 | Units, payment methods, expense types, leave types, cities |
| S13 | **Scan service** | Item Barcode lookup ✅ | One `lookup_code(code)` (barcode / item code / batch), used by count, receiving, counter, packing | Reusable scan box (scanner + phone keyboard, Add button) | P3.8 | Phases 3, 5, 6, 8 |
| S14 | **Receive / pay money** | Payment Entry, allocation to invoices ✅ | API: receive from customer, pay supplier, pay other; cheque details; allocation | Payment dialog reused everywhere | P4.5 | Purchasing, sales, accounts, riders |
| S15 | **Sales Channel auto-fill** | Accounting Dimension ✅ | 🔴 hooks: fill channel from POS Profile / customer group / order source; block submit if empty | Channel chip on documents | P2.5 | All sales and payments, channel reports |
| S16 | **Delivery charge by city** | Shipping Rule (country only, not city) | 🔴 "City Delivery Charge" doctype + apply on quotation/order/cart | Settings page; shown on order and cart | P6.6 | Self-delivery, website |

---

## 6. Dependency map

```
P0 Stabilise + decide
 │
 ▼
P1 Platform (S1–S12) ──────────────────────────────────────────────┐
 │                                                                 │
 ▼                                                                 │
P2 Accounting + commercial foundation (S15)                        │
 │                                                                 │
 ├──► P3 Products & stock — finish (S13) ──► P9A Website catalogue │
 │        │                                                        │
 │        ▼                                                        │
 ├──► P4 Customers, dealers, suppliers, payments, ledgers (S14)    │
 │        │                                                        │
 │        ▼                                                        │
 │    P5 Purchasing                                                │
 │        │                                                        │
 │        ▼                                                        │
 │    P6 Sales: pricing rules, POS, wholesale, riders (S16)        │
 │        │                    │                                   │
 │        ▼                    ▼                                   │
 │    P7 Accounts module    P8 Online orders + PostEx              │
 │                             │                                   │
 │                             ▼                                   │
 │                  P9B Website account + checkout ──► P10 Payment gateway
 │                             │
 │                             ▼
 │                  P9C Website tracking + order emails
 │
 └──► P11 HR & payroll (needs P1 + P2 only; runs in parallel from P4 on)

P12 Dashboards, alerts, owner summary, report check  (needs P3–P11)
P13 Migration, UAT, training, go-live, support        (needs everything)
```

Why this order:
- **P1 before everything:** every screen needs permissions, audit, export, print, email and approvals.
- **P2 before any transaction:** purchases, sales, stock write-offs and payments post to accounts, taxes and the channel dimension.
- **P4 before P5 and P6:** suppliers and customers, their payment terms, credit limits and the payment service are used by both.
- **P5 before P6:** purchases set the real cost (with freight), and margin and below-cost checks depend on cost.
- **P6 before P8:** online orders are Sales Orders with stages; reservation, returns and payment flows come from P6.
- **P7 after P5 and P6:** the accounts module shows receipts, payments and ledgers that those phases create. Its **foundations** are in P2.
- **P11 is independent** of sales and stock. It only needs users, branches (P1) and payroll accounts (P2), so it can run in parallel.

---

## 7. Phases and steps

Column "Type": ✅ built-in · ⚙️ configuration · 🔧 customization · 🔴 custom development.

### Phase 0 — Stabilise and decide

**Needs before start:** nothing. **Later phases rely on it for:** a safe codebase, a clean site, closed decisions.

| ID | Step | SRS | ERPNext gives | Type | Our backend | Next.js |
|---|---|---|---|---|---|---|
| P0.1 | Commit all code to Git (branches per feature), add CI: pre-commit, backend tests, typecheck, lint | 6 | — | 🔴 | GitHub Actions | CI for `web/` |
| P0.2 | Fix I1: new patch that runs `ensure_default_warehouse`; add a "fresh site vs migrated site" check to tests | 3.3 | Company default warehouse | ⚙️ | Patch v0_5 + test | — |
| P0.3 | Fix I2: disable unused ERPNext warehouses (keep Goods In Transit only if branches transfer by courier); pickers show only active Adeel warehouses | 3.3 | Warehouse disable | ⚙️ | Patch + `form_options` filter + test | Pickers |
| P0.4 | Fix I13: remove old Stock Counter permission on Stock Reconciliation | 3.3 | Custom DocPerm | ⚙️ | Patch + test | — |
| P0.5 | Fix I16: remove `temp`, `test`, `1` items and test documents from the demo site | 6 | — | — | Cleanup script | — |
| P0.6 | Decisions are closed **when the phase that needs them starts** (see section 4). Closed so far: D1 = B, D4, D20. D16 pending. Update `ARCHITECTURE_REVIEW.md` for D1 | 7, 8 | — | — | — | — |
| P0.7 | Short PoC for risky foundations: (a) Workflow approve/reject through our API, **with role enforcement**: only the roles allowed for that approval type can approve or reject (Owner/Manager for PO, discount, credit, expense; **Owner only for payroll**), every other role is refused when it calls the API directly, and each approve/reject is recorded in the change history (who, when, action, reject comment); (b) realtime (socket.io) from Next.js; (c) Webshop cart → Sales Order from a non-Jinja client; (d) ERPNext POS with the real scanner and receipt printer (for D2) | 3.7, 3.11, 3.14 | Workflow, socket.io, Webshop APIs, POS | 🔴 PoC | Throw-away scripts, results written | Small test pages |
| P0.8 | Correct `IMPLEMENTATION_PLAN.md` Phase 2 status (I15) | — | — | — | — | — |

**Exit criteria:** code in Git with green CI · I1–I18 fixed or owned by a later step · D1, D4, D20 closed (done) and the rest closed before the phase that needs them · PoC report (pass/fail per item).
**Business flow:** none (stabilisation).

---

### Phase 1 — Platform (shared infrastructure S1–S12)

**Needs before start:** P0; D1, D4, D20 (all closed). D16 (email account) is pending: real sending waits for it. **Later phases rely on it for:** every screen, every approval, every export, print and email.

| ID | Step | SRS | ERPNext gives | Type | Our backend | Next.js |
|---|---|---|---|---|---|---|
| P1.1 | S1 Permission model: final role list (Owner, Manager, Counter Staff, Store, Accountant, HR, Online Order Team; riders are Driver records, not logins, D20), role profiles, module profiles, branch user permissions; capability map in `session.me` | 2.1, 3.13, 6 | Roles, profiles, User Permission ✅ | ⚙️🔴 | Fixtures; capability map; tests: each role allowed/denied | `useCan()`, menu by capability, no-access page; update existing product/count pages to use it |
| ↳ **P1.1 status 11 Nov 2026: ✅ built and checked** (capability map, Can View Cost and Can Export Lists roles, role profiles fixed, branch rule on stock moves and counts, `useCan()` and page guards, 7 demo users, 14 permission tests). Not yet done for P1.1: fresh-site install run (G2), Desk access closed for staff (moved to P1.3). | | | | | | |
| P1.2 | S2 Custom roles screen: owner creates a role and ticks view/create/edit/delete/approve/export per area | 2.1 | Role, Custom DocPerm ✅ | 🔴 | Role API with guard rails + tests | Roles & permissions page |
| ↳ **P1.2 status 11 Nov 2026: ✅ built and checked** (area list, Roles API with guard rails, Roles & permissions pages, 16 role tests incl. an HTTP-level test, browser check as owner and as a user holding a custom role). Giving the role to a person waits for the Users page (P1.3). Approve column waits for P1.6. | | | | | | |
| P1.3 | S3 Users and login security: invite, role profile, branch, disable, reset password; password policy, **auto logout** (fix I7), login attempts limit, 2FA for owner/manager/accountant, login history | 3.13, 6 | User, System Settings, Activity Log ✅ | ⚙️🔴 | Settings patch; users API; login history API | Users page; Login history page |
| ↳ **P1.3 status 12 Nov 2026: ✅ built and checked** (People list and person page, add a person with a temporary password shown once, change role and branch, disable/enable, reset password, forced password change at first login, login history from Frappe's Activity Log, 8-hour auto logout, lock for 5 minutes after 10 wrong tries (standard: Microsoft baseline, NIST 800-63B), Desk closed for staff (only Owner and Accountant may open it), 12 People tests; custom roles now get their own Role Profile so they survive saves). **2-step login dropped by the owner for now.** | | | | | | |
| P1.4 | S4 Company, logo, branches: **2 branches (Main Warehouse, Shop Counter, D4)**, each with Branch + Cost Center + warehouses; counter users limited to their branch | 3.13, 3.3 | Company, Branch, Cost Center, Letter Head ✅ | ⚙️🔴 | Setup patch from D4; branch API; tests for branch restriction | Company & branches settings page |
| ↳ **P1.4 status 12 Nov 2026: ✅ built and checked** (Company page: NTN, phone, email, website, address, logo; Branches: add a branch in one step (Branch + own stock location + own cost center), rename (links follow), never delete; shared locations listed; each branch has a cost center for Phase 2; People offers new branches; 7 Company tests). Letterhead from the logo comes in P1.8 (printing). | | | | | | |
| P1.5 | S5 Change history: tracking on every business doctype (fix I6); history API | 3.13 | Version, Activity Log ✅ | ⚙️🔴 | Property setter patch; `get_changes`, audit log API (permission-checked) | History panel on product page (first user), Audit log page |
| ↳ **P1.5 status 12 Nov 2026: ✅ built and checked** (track changes switched on for Branch, Stock Reconciliation, Brand, Item Group, Product Bundle, Pricing Rule, Address, UOM, Mode of Payment, Email Template, Letter Head, Role Profile (fixes I6); our own saves now go through `doc.save()` so they keep history; person notes for branches, password reset, log out everywhere, unlock; `api/v1/history.py` reads Frappe's Version/Comment/Deleted Document/Activity Log in plain words, cost hidden for non-cost viewers; History card on product, person, role and Company pages; **Settings › Change history** for owner and accountant (`audit.view`); 6 tests). | | | | | | |
| P1.6 | S6 Approvals engine: generic pending/approve/reject API on Workflow; helper for threshold rules | 3.14, 3.2, 3.5, 3.6, 3.9, 3.10 | Workflow ✅ | ⚙️🔴 | API + tests with a sample workflow | Approvals inbox, approval banner |
| ↳ **P1.6 status 12 Nov 2026: ✅ built and checked** (on Frappe Workflow + Workflow Action: `api/v1/approvals.py` inbox, approve/reject (reason required to reject), state for a banner, the server refuses anyone the workflow does not allow; approvers get a bell notification; **Approvals** page with a count in the sidebar, `ApprovalBanner` ready for document pages; 6 tests with a temporary sample workflow). **Amounts/limits (discount, credit, PO) are added by the phase that owns each rule** (no values invented). | | | | | | |
| P1.7 | S7 Email and notifications: outgoing account (D16), branded base email template with logo, notification helper, in-app bell | 3.12, 3.13, 5 | Email Account, Email Template, Notification ✅ | ⚙️🔴 | Settings patch; helper; email queue check | Bell; Email templates settings page |
| ↳ **P1.7 status 12 Nov 2026: ✅ built and checked, sending waits for D16** (🔔 bell on Frappe's notification log: unread count, list, mark read, opens our page; **Settings › Email**: account status, queue (waiting/failed/sent today), email templates with a preview carrying the logo; the company logo is also the email logo). Without an email account Frappe refuses to send or even queue an email, so the Email buttons appear automatically once the account (D16) is added (`session.email_ready`). | | | | | | |
| P1.8 | S8 Print and PDF: base A4 and 80 mm layouts with logo; PDF API; signed share link | 3.7, 3.13, 6 | Print Format, PDF ✅ | 🔧🔴 | Base Jinja layouts; PDF/share API | Print/PDF/Email/Share action |
| ↳ **P1.8 status 12 Nov 2026: ✅ built and checked** (Letter Head "Adeel Traders" built from the Company page and kept up to date, ERPNext's sample letterheads switched off (I10 part); base A4 and 80 mm Jinja layouts; first formats **Stock Count Sheet** (A4, blind count respected) and **Stock Move Slip** (80 mm); Print/PDF through Frappe's permission-checked `download_pdf`; WhatsApp share link valid 7 days (Document Share Key); Email with the PDF for roles that may (Stock Manager, owner); 5 tests). Real label/receipt printer test waits for D17. | | | | | | |
| P1.9 | S9 Export any list to Excel | 6, 4 | Export utilities ✅ | 🔴 | Export API (same filters + permissions) | Export button in table kit; add to Products list |
| ↳ **P1.9 status 12 Nov 2026: ✅ built and checked** (`api/v1/export.py`: Products, People, Stock counts and reports to Excel through the same list function as the screen, cost only for cost viewers, max 10,000 rows, every export written to Frappe's Access Log; one Export button replaces the old browser CSV; 5 tests). | | | | | | |
| P1.10 | S10 List and report kit: filters date/branch/channel/category/brand, report wrapper pattern | 4 | Script/Query reports ✅ | 🔴 | Report wrapper + test pattern | DataTable, report page layout |
| ↳ **P1.10 status 12 Nov 2026: ✅ built and checked** (`api/v1/reports.py` wraps ERPNext reports with our filters (dates, branch → its locations, category, brand, search, sort, paging) and hides cost columns; DataTable (sticky header, sort, totals, paging), FilterBar (filters in the link), ReportView; first report **Stock balance**; managers given the report with a Custom Role; 5 tests). Charts and the other reports come with their phases (P3.5 onwards). | | | | | | |
| P1.11 | S11 Integration framework for `pk_integrations` | 5 | Background jobs, Integration Request ✅ | 🔴 | Job base with retry, idempotency, log, alert | Integration status page |
| ↳ **P1.11 status 12 Nov 2026: ✅ built and checked** (`pk_integrations/base.py` on Frappe's Integration Request: one log row per try, a key that already succeeded is never sent twice, failed calls retried by the scheduler (5, 10, 20 minutes), owner alerted after the 3rd failure; **Settings › Integrations** with PostEx, payment gateway and attendance machine shown as "Not connected" until their phase; 4 tests). | | | | | | |
| P1.12 | S12 Settings and masters kit: units, payment methods list (accounts come in P2), naming series view | 3.13 | UOM, Mode of Payment, Naming Series ✅ | ⚙️🔴 | Generic master API | Settings › Units, Settings › Numbering |
| ↳ **P1.12 status 12 Nov 2026: ✅ built and checked** (`api/v1/masters.py`: one kit for simple lists (add, rename, switch off, delete only when unused); Catalogue uses the same helpers; **Settings › Units** (ERPNext's 229 unused sample units switched off once, can be switched back on), **Settings › Payment methods** (grouped by kind, on/off switch, rename/kind/delete in a menu; Cash is protected because ERPNext's till uses it; accounts and the business's real methods JazzCash, Easypaisa, Bank Transfer, PostEx COD come in P2.3), **Settings › Numbering** view-only until D6; 7 tests). | | | | | | |

**Exit criteria:** G1–G12 · every role logs in and sees only its menu · counter/store user of branch A cannot **act on** branch B stock (API test; reading quantities of other branches stays open by decision) · an owner-made role works end to end · a test approval goes through inbox → approve → history · **a role not allowed to approve is refused by the server (not only hidden in the screen), for every approval type** · a test email arrives with logo · export works on the Products list · auto logout works.
**Business flow F-P1:** owner creates a "Senior Cashier" role → invites a user → user logs in, sees only allowed pages → owner sees the user's login history and change history.

---

### Phase 2 — Accounting and commercial foundation

**Needs before start:** P1; D3, D5, D6, D7, D8, D9 (rule only), D10, D11, D12. **Later phases rely on it for:** every posting in P3–P11.

| ID | Step | SRS | ERPNext gives | Type | Our backend | Next.js |
|---|---|---|---|---|---|---|
| P2.1 | Chart of accounts reviewed by the accountant: bank accounts, **JazzCash, Easypaisa** wallets, **petty cash** per shop, **PostEx COD receivable**, rider cash in hand, GST accounts, freight/landed cost, discounts, damaged/expired loss (D8), **expense accounts per category** (rent, utilities, fuel, packaging, marketing…) | 3.9, 3.13 | Chart of Accounts ✅ | ⚙️ | Patch from `data.py` (accounts as code) + test | Accounts list (read-only) in Settings |
| P2.2 | Company default accounts and item-group defaults (income, COGS, stock, write-off, round off); fix I9 (default price list, default customer group) | 3.2, 3.9 | Company, Item Group defaults, Selling Settings ✅ | ⚙️ | Patch + test | — |
| P2.3 | Modes of payment linked to accounts: Cash, Card, Bank Transfer, JazzCash, Easypaisa, Cheque, PostEx COD (fix I3) | 3.7, 3.9, 3.13 | Mode of Payment ✅ | ⚙️ | Patch + test | Settings › Payment methods |
| P2.4 | Tax (D5): GST sales/purchase templates, item tax, tax category; or remove the unreviewed template (I4) | 3.13 | Tax templates ✅ | ⚙️ | Patch + test | Settings › Tax |
| P2.5 | S15 Sales Channel auto-fill on Quotation, Sales Order, Delivery Note, Sales Invoice, POS Invoice, Payment Entry, Journal Entry; cost center = branch | 1.2, 3.14, 4 | Accounting Dimension ✅ | 🔴 | Hooks + tests (each doc type) | Channel chip |
| P2.6 | Numbering (D6) for all documents and branches | 3.13 | Naming Series ✅ | ⚙️ | Patch + test | Settings › Numbering (from P1.12) |
| P2.7 | Fiscal year, FIFO (D7), accounting period lock, immutable ledger decision | 3.9 | Fiscal Year, Accounting Period ✅ | ⚙️ | Patch | — |
| P2.8 | Opening balance method (data comes in P13): temporary opening account; parties via Opening Invoice Creation Tool; cash/bank via Journal Entry; stock via Opening Stock reconciliation | 3.3, 3.5, 3.6, 3.9 | Opening tools ✅ | ⚙️ | Written procedure + test on sample data | — |
| P2.9 | Party foundations: customer groups → price list (exists), supplier groups, payment terms templates, **Territory = city** (D10) with the city list | 3.6, 3.5, 4 | Customer/Supplier Group, Payment Terms, Territory ✅ | ⚙️ | Patch + test | Settings › Cities |
| P2.10 | Price rules recorded: below-cost behaviour (D9), self-delivery for walk-ins (D12), Standard Selling hidden | 3.2 | Selling Settings, Price List ✅ | ⚙️ | Patch | — |

**Exit criteria:** G1–G12 · accountant signs the chart and the **posting table** for: purchase receipt, purchase invoice, sales invoice, POS invoice, payment in/out, stock write-off, stock transfer, landed cost, opening entries — each tested with a sample document showing the right accounts, tax, branch and channel.
**Business flow F-P2:** test purchase → test sale → test payment on a test company copy; trial balance matches by hand.

---

### Phase 3 — Products and stock (finish)

**Needs before start:** P1 (export, audit, settings kit, permissions), P2 (accounts for opening stock and write-offs). **Later phases rely on it for:** website flags (P9), scan service (P5, P6, P8), stock reports (P12).

| ID | Step | SRS | ERPNext gives | Type | Our backend | Next.js |
|---|---|---|---|---|---|---|
| P3.1 | Excel upload and download of products and prices, with preview and error list. Templates get **Brand**, **Category**, **units (box/carton size)** columns (today they have no Brand column) | 3.1 | Data Import ✅ | 🔴 | Import API on Data Import (our template, validation, log) + tests | Import page (upload → preview → import → result); Download catalogue |
| P3.1b | **Brand and category import first**: import brands and categories from a sheet, or the product import lists the new brands/categories in the preview and creates them after the owner confirms (a product import fails today if its brand does not exist; the site has 0 brands) | 3.1, 3.13 | Data Import ✅ | 🔴 | Brand/category import API + tests (duplicate names, spelling variants) | Settings › Catalogue › Import |
| P3.2 | Opening stock screen (with batch, expiry, cost) | 3.3 | Stock Reconciliation "Opening Stock" ✅ | 🔴 | API + tests (posts to temporary opening account) | Opening stock page (scan or type) |
| P3.3 | Minimum stock settings: bulk reorder level and usual order per product and location | 3.3, 3.13 | Item Reorder ✅ | 🔴 | Bulk API + tests | Settings › Minimum stock (table edit, import) |
| P3.4 | Low-stock alerts: in-app + email to Store/Manager (uses S7) | 3.3, 3.12 | Reorder, Notification ✅ | ⚙️🔴 | Daily job + tests | Bell + "Needs restocking" link |
| P3.5 | Stock reports: current stock, stock value (branch and total), low stock, stock movement, slow and dead stock (D18) | 4 | Stock Balance, Stock Ledger, Stock Ageing, Stock Projected Qty ✅ | 🔴 wrappers | Report APIs (S10) + tests | 5 report pages with export |
| P3.6 | Website flags: show on website, featured, new arrival, on sale; **skin type** field for website filter | 3.1, 3.11 | Website Item ✅ | 🔧🔴 | Custom fields + sync Item → Website Item + tests | Toggles on product page; filter on list |
| P3.7 | Stock value grouped by branch (after P1.4) | 3.3 | Bin ✅ | 🔴 | Extend list summary | Hero card by branch |
| P3.8 | S13 Scan service: move the count's lookup into one shared function | 3.4 | Item Barcode ✅ | 🔴 | `lookup_code` + tests | Shared scan box |
| P3.9 | Labels on the client's printer (D17): server print format with name, price, barcode | 3.4, 5 | Print Format ✅ | 🔧 | Jinja label format (S8) | Print labels (single and bulk) |
| P3.10 | Re-check existing product and count screens against P1 (capability map, history panel, export) | 3.1–3.4 | — | 🔴 | Tests | Updates |
| P3.11 | **Unit chain** (D23): Dozen = 12 pieces for every product (global); "Add unit" accepts "Carton = 12 Boxes" and saves 144 pieces; check that carton is a whole number of boxes; bulk set box/carton sizes by import (today only Box = 12 is set, on 5 items; Dozen and Carton on none) | 3.2 | UOM Conversion Factor, Item UOM table ✅ | ⚙️🔴 | Global Dozen conversion patch; `add_unit` accepts a base unit; tests | Add unit dialog shows "1 Carton = 12 Boxes = 144 Pieces" |

**Exit criteria:** G1–G12 · import 500 test products with prices, then download them unchanged · opening stock posts the right value · alert email received · stock reports match the product list numbers.
**Business flow F-P3:** import catalogue → opening stock → set minimum levels → stock count → alert for a low product → stock reports.

---

### Phase 4 — Customers, dealers, suppliers, payments and ledgers

**Needs before start:** P1, P2 (groups, terms, cities, payment methods). **Later phases rely on it for:** parties, credit limits and the payment service in P5–P8.

| ID | Step | SRS | ERPNext gives | Type | Our backend | Next.js |
|---|---|---|---|---|---|---|
| P4.1 | Customers: type → price list automatically, details, several addresses, purchase history | 3.6 | Customer, Address, Contact ✅ | ⚙️🔴 | Customer API + tests | Customers list + customer page |
| P4.2 | Dealer credit limits and overdue threshold (approval of over-limit sales is built in P6.3) | 3.6 | Customer Credit Limit ✅ | ⚙️🔴 | API + tests | Credit section on customer page |
| P4.3 | Suppliers: contacts, payment terms, groups | 3.5 | Supplier ✅ | 🔴 | Supplier API + tests | Suppliers list + supplier page |
| P4.4 | Opening balances for customers, dealers, suppliers (screen; real data in P13) | 3.5, 3.6 | Opening Invoice Creation Tool ✅ | 🔴 | API + tests | Opening balance import |
| P4.5 | S14 Receive / pay money: cash, card, bank, JazzCash, Easypaisa, cheque details; allocate to invoices | 3.5, 3.7, 3.9 | Payment Entry ✅ | 🔴 | Payment API + tests (posting table) | Payment dialog |
| P4.6 | Ledgers and statements: customer, supplier; print and email | 3.6, 3.5, 3.12 | General Ledger, Ledger Summary, Process Statement Of Accounts ✅ | 🔴 wrappers | Ledger API + statement PDF/email | Ledger tab, Statement button |
| P4.7 | Pending payments by age 30/60/90+ | 3.6, 4 | Accounts Receivable/Payable ✅ | 🔴 wrapper | Report API | Ageing page |
| P4.8 | Overdue reminders by email to dealers | 3.6, 3.12 | Dunning, auto statements ✅ | ⚙️ | Schedule + template | Reminder settings |
| P4.9 | Top customers and customers who stopped buying | 3.6, 4 | Sales Analytics, Inactive Customers ✅ | 🔴 wrapper | Report API | 2 report pages |

**Exit criteria:** G1–G12 · a customer's type sets the right price on a test quotation · receive and pay money post correctly · statement PDF and email arrive · ageing matches the ledger.
**Business flow F-P4:** create dealer with credit limit and opening balance → receive part payment by cheque → statement emailed → ageing shows the rest.

---

### Phase 5 — Suppliers and purchasing

**Needs before start:** P3 (scan service, stock), P4 (suppliers, payment service, ledger), P1.6 (approvals), P1.8 (print). **Later phases rely on it for:** real cost with freight (margin, below-cost check in P6).

| ID | Step | SRS | ERPNext gives | Type | Our backend | Next.js |
|---|---|---|---|---|---|---|
| P5.1 | Reorder list: products below level with suggested quantity → create PO | 3.5 | Material Request (auto), Recommended Reorder report ✅ | 🔴 | Suggestion API + tests | Reorder list; "Create purchase order" on restock card and product page |
| P5.2 | RFQ to suppliers and compare quotations | 3.5 | RFQ, Supplier Quotation, comparison report ✅ | ⚙️🔴 | API + tests | RFQ page, compare view |
| P5.3 | Purchase order with **approval** before sending | 3.5 | Purchase Order + Workflow ✅ | ⚙️🔴 | Workflow (S6) + rules + tests | PO list and page, approve in inbox |
| P5.4 | Email or print PO | 3.5 | Print Format, email ✅ | 🔧 | PO print format (S8) | Print/Email action |
| P5.5 | Last purchase price per supplier while ordering | 3.5 | Last purchase rate ✅ | 🔴 | API + test | Shown on PO lines |
| P5.6 | Receive stock against PO by scan, full or partial, with batch and expiry | 3.5, 3.4 | Purchase Receipt with scan ✅ | 🔴 | Receiving API (S13) + tests | Receiving screen (scan-first, phone friendly) |
| P5.7 | Purchase bill | 3.5 | Purchase Invoice ✅ | 🔴 | API + tests | Bill page |
| P5.8 | Freight and other costs added to product cost | 3.5 | Landed Cost Voucher ✅ | 🔴 | API + tests (cost changes) | "Add freight" on receipt |
| P5.9 | Purchase returns | 3.5 | Return against receipt/bill ✅ | 🔴 | API + tests | Return action |
| P5.10 | Supplier payments and ledger (uses P4.5, P4.6) | 3.5 | ✅ | — | Tests | Pay supplier on bill |
| P5.11 | Purchase reports: by supplier and product, pending POs, supplier balances | 4 | Purchase Analytics, PO Analysis, Accounts Payable ✅ | 🔴 wrappers | Report APIs | 3 report pages |
| P5.12 | Buying in box/carton converts stock to pieces | 3.2 | UOM conversion ✅ | — | Test | — |

**Exit criteria:** G1–G12 · PO cannot be sent before approval · partial receipt leaves the rest pending · freight changes the product cost on the product page · return reduces stock and supplier balance · reports match.
**Business flow F-P5:** reorder list → RFQ → compare → PO → approve → email → receive 2 of 3 lines by scan → bill → freight → pay → supplier ledger.

---

### Phase 6 — Sales: pricing rules, POS, wholesale, self-delivery, riders, returns

**Needs before start:** P2–P5; D2, D9, D13, D17. **Later phases rely on it for:** Sales Order flow, reservation, returns, documents, delivery charges (P8, P9).

| ID | Step | SRS | ERPNext gives | Type | Our backend | Next.js |
|---|---|---|---|---|---|---|
| P6.1 | Pricing rules: dealer special prices, item and bill discounts, **discount limit per role → approval** (7.3, 7.17), below-cost warning + approval (D9), coupon codes, website sale prices | 3.2 | Pricing Rule, Coupon Code, Party Specific Item ✅ | ⚙️🔴 | Validation hooks + approval rules + tests | Price rules page; discount approval in inbox |
| P6.2 | POS (D2): counter profile per branch, scan, Self-delivery price, split payment, hold/recall, receipt 80 mm, counter cannot change price, daily cash closing | 3.7, 3.4 | POS Invoice, POS Profile, Opening/Closing ✅ | ⚙️🔴 | Profiles, receipt format, closing checks + tests | POS screen (if D2 = Next.js) or branded ERPNext POS |
| P6.3 | Wholesale: quotation → sales order → delivery → invoice; credit sale goes to ledger; **over-limit sale waits for manager approval** (7.17) | 3.7, 3.6 | Quotation, SO, DN, SI, credit check ✅ | ⚙️🔴 | Approval hook + tests | Sales order and invoice pages |
| P6.4 | Salesperson on every sale (mandatory) and commission % | 3.7 | Sales Team ✅ | 🔧 | Property setter + test | Salesperson field |
| P6.5 | Self-delivery orders: rider assignment, cash collected per rider, **rider cash settlement** (7.4) | 3.7 | Driver, Delivery Trip ✅ | 🔴 | Rider Settlement doctype + API + tests | Rider assignment, settlement screen |
| P6.6 | S16 Delivery charge by city (7.7) | 3.11, 3.13 | Shipping Rule (country only) | 🔴 | City Delivery Charge doctype + apply hook + tests | Settings › Delivery charges |
| P6.7 | Returns and **exchanges**; returned goods go to "Returns - To Check", then sellable or Damaged | 3.7, 3.3 | Credit note / return ✅ | 🔧🔴 | Return warehouse rule; exchange API + tests | Return and exchange flow |
| P6.8 | Branded invoice, delivery challan, receipt, quotation: PDF, print, email, **WhatsApp share link**, reprint old ones | 3.7, 3.12, 3.13 | Print Format ✅ | 🔧 | Formats on S8 | Actions on every document |
| P6.9 | Payment by cash, card, transfer, JazzCash, Easypaisa, split | 3.7 | Mode of Payment ✅ | — | Tests | In POS and payment dialog |
| P6.10 | Reservation and gift-set proofs: order reserves, cancel releases, last unit cannot be sold twice; gift set sale deducts each part | 3.3, 3.1 | Stock Reservation, Product Bundle ✅ | — | Tests | — |
| P6.11 | Sales Channel filled on every sales document (S15) | 4 | ✅ | — | Tests | — |
| P6.12 | Sales reports: daily/monthly/yearly, by channel, product, category, brand, customer, salesperson, city; profit by product and invoice; cash closing; cash by rider | 4 | Sales Analytics, Gross Profit, Sales Person reports, POS register ✅ | 🔴 wrappers | Report APIs + custom rider report | Report pages |

**Exit criteria:** G1–G12 · counter staff cannot change price or give a discount over their limit without approval · over-limit credit sale waits in the inbox · cash closing equals payments · rider settlement balances · all sales reports filter by channel.
**Business flow F-P6:** 10 POS sales (one split payment, one return) → closing; dealer credit sale over limit → approval → delivery → invoice → payment; rider delivery → settlement.

---

### Phase 7 — Accounts module

**Needs before start:** P2 (accounts), P4 (payment service, ledgers), P5 and P6 (real documents to show). **Later phases rely on it for:** owner dashboard cash and profit (P12).

| ID | Step | SRS | ERPNext gives | Type | Our backend | Next.js |
|---|---|---|---|---|---|---|
| P7.1 | Cash, bank and wallet accounts with live balances | 3.9 | Account balances ✅ | 🔴 wrapper | API | Accounts page |
| P7.2 | Receipts and payments to others (non-party) | 3.9 | Payment Entry / Journal Entry ✅ | 🔴 | Reuse S14 | Receive / Pay buttons |
| P7.3 | Expenses by category with bill photo; **approval** (D11, Q21) | 3.9 | Journal Entry ✅ | 🔴 | Expense doctype → Journal Entry after approval + tests | Add expense (camera upload), expense list |
| P7.4 | Transfers between accounts | 3.9 | Internal Transfer ✅ | 🔴 | API + test | Transfer dialog |
| P7.5 | Daily cash book | 3.9, 4 | General Ledger ✅ | 🔴 wrapper | Report API | Cash book page |
| P7.6 | Cheques received and issued, clearing status | 3.9 | Payment Entry, Bank Clearance, Cheque Print Template ✅ | ⚙️🔴 | API + tests | Cheques page (clear / bounce) |
| P7.7 | Petty cash per shop: float, top-up, small expenses | 3.9 | Cash account ✅ | ⚙️ | Uses P7.3, P7.4 | Petty cash view |
| P7.8 | Opening balances for cash and bank accounts (screen; data in P13) | 3.9 | Journal Entry ✅ | 🔴 | API + test | Opening balances page |
| P7.9 | Simple monthly profit and loss | 3.9, 4 | Profit and Loss Statement ✅ | 🔴 wrapper | Report API | P&L page |
| P7.10 | Expense reports (category, branch, month); cash and bank book; ledgers | 4 | GL, P&L ✅ | 🔴 wrappers | Report APIs | Report pages |
| P7.11 | Month-end checklist and period lock | 3.9 | Accounting Period ✅ | ⚙️ | Procedure | — |

**Exit criteria:** G1–G12 · expense over the limit cannot be saved without a photo and waits for approval · bounced cheque reverses correctly · P&L equals the ERPNext report.
**Business flow F-P7:** petty cash top-up → 3 expenses (one needs approval) → cheque received and cleared → bank transfer → cash book → monthly P&L.

---

### Phase 8 — Online orders and PostEx

**Needs before start:** P6 (orders, reservation, returns, delivery charges), P4 (customers), P1.11 (integration framework), Q11 (PostEx account). **Later phases rely on it for:** website tracking (P9C).

| ID | Step | SRS | ERPNext gives | Type | Our backend | Next.js |
|---|---|---|---|---|---|---|
| P8.1 | Order stages: Pending → Confirmed → Packed → Booked → Dispatched → Delivered / Returned / Cancelled, with stage history (7.1) | 3.8 | Sales Order ✅ | 🔧🔴 | `order_stage` field + transition API + tests | — |
| P8.2 | Phone / WhatsApp / social orders entered by staff (order source) | 3.8 | Sales Order ✅ | 🔧🔴 | API + tests | New order form |
| P8.3 | All orders in one list / board | 3.8 | — | 🔴 | List API | Orders board (realtime) |
| P8.4 | Pick list for the day and **packing by scan** (wrong item or extra qty blocked) | 3.8, 3.4 | Pick List ✅ | 🔴 | Packing API (S13) + tests | Packing screen |
| P8.5 | Pick lists and packing slips print | 3.8 | Print Format ✅ | 🔧 | Formats | Print action |
| P8.6 | PostEx: settings, cities, booking, labels, tracking sync, **COD matching** (7.2) | 3.8, 5 | — | 🔴 `pk_integrations` | On S11 + sandbox tests | Book (bulk), labels, COD matching page |
| P8.7 | Returned parcels back into stock ("Returns - To Check") | 3.8 | Sales Return ✅ | 🔴 | API + tests | Return action |
| P8.8 | Cancel with reason; reservation released | 3.8 | Reservation release ✅ | 🔴 | API + tests | Cancel dialog |
| P8.9 | Flag customers who often return or refuse COD (7.9) | 3.8 | — | 🔴 | Rule + report | Flag on order and customer |
| P8.10 | Customer emails at each stage | 3.12 | Notification ✅ | ⚙️ | Templates on S7 | — |
| P8.11 | Online order reports: by stage, delivery success and return rate, COD received vs pending | 4 | — | 🔴 | Report APIs | Report pages |

**Exit criteria:** G1–G12 · 50 test orders from Pending to Delivered · PostEx sandbox booking, label and tracking work with retries · COD payout matched · wrong item blocked at packing.
**Business flow F-P8:** phone order → confirm → pick → pack by scan → book PostEx → tracking → delivered → COD matched; one returned parcel back in stock.

---

### Phase 9 — Website (replaces WordPress)

Three parts, each with its own prerequisites.

**9A — Catalogue** · Needs: P3 (website flags, skin type), P2 (prices), D14.
**9B — Account and checkout** · Needs: 9A, P4 (customers), P6 (orders, coupons, delivery charges), D19 (guest checkout).
**9C — Tracking** · Needs: 9B, P8.

| ID | Step | SRS | ERPNext gives | Type | Our backend | Next.js (storefront) |
|---|---|---|---|---|---|---|
| P9A.1 | Webshop settings, Website Items synced from products, live price and stock | 3.11 | Webshop ✅ | ⚙️ | Settings patch; cache refresh webhook | — |
| P9A.2 | Home, category, brand, search, filters (price, skin type, shade), product page with live stock per shade; featured/new/sale badges | 3.11, 3.1 | Webshop product APIs ✅ | 🔴 | Public API with rate limit | Pages |
| P9A.3 | Home banners and sections editable from the admin (D14) | 3.11 | Website Slideshow ✅ | 🔴 | Home Section doctype + API | Admin page in staff app; sections on storefront |
| P9A.4 | About, contact, delivery and return policy pages, editable | 3.11 | Web Page ✅ | ⚙️ | API | Pages + admin editor |
| P9A.5 | SEO (meta, sitemap, clean URLs, alt text), WhatsApp button, Google Analytics, Facebook Pixel | 3.11 | Website Settings ✅ | ⚙️ | Settings | Head tags, sitemap |
| P9B.1 | Sign-up, login, password reset, saved addresses | 3.11 | Website user, Address ✅ | ⚙️ | API wrappers | Account pages |
| P9B.2 | Cart, checkout with COD, delivery charge by city (S16), coupon codes, website sale prices | 3.11, 3.2 | Webshop cart APIs ✅ | 🔴 | Delivery charge hook | Cart and checkout |
| P9B.3 | Wishlist; product reviews shown **after approval** | 3.11 | Wishlist, Item Review ✅ | 🔧 | Approval flag on reviews | Wishlist, reviews |
| P9B.4 | "Notify me when back in stock" (7.8) | 3.11 | — | 🔴 | Doctype + job + email | Button on product |
| P9C.1 | Order history and **tracking with PostEx status** | 3.11 | Sales Order ✅ | 🔴 | API | Track page |
| P9C.2 | Order emails to customers (from P8.10) | 3.12 | ✅ | — | — | — |

**Exit criteria:** G1–G12 · Lighthouse mobile ≥ 80 · real test order from the website reserves stock and appears on the orders board.

---

### Phase 10 — Online payment gateway

**Needs before start:** P9B, D19 (provider, Q4), P1.11.

| ID | Step | SRS | ERPNext gives | Type | Our backend | Next.js |
|---|---|---|---|---|---|---|
| P10.1 | Gateway on the Payments app pattern (7.10) | 3.11, 5 | Payment Request, Payment Gateway Account ✅ | 🔴 `pk_integrations` | Gateway + callbacks + tests (success, fail, cancel, duplicate, refund) | Checkout payment step |

**Exit criteria:** 100 sandbox payments reconcile to Payment Entries with zero mismatch.

---

### Phase 11 — HR and payroll (parallel track)

**Needs before start:** P1 (users, branches, approvals, email, print), P2 (payroll accounts), D19 (attendance machine Q9, policies Q10). Commission (P11.9) needs P6 data.

| ID | Step | SRS | ERPNext/HRMS gives | Type | Our backend | Next.js / Desk (D1) |
|---|---|---|---|---|---|---|
| P11.1 | Employees: personal details, department, designation, branch, joining date; CNIC and contract files | 3.10 | Employee ✅ | ⚙️ | Setup patch | Employee pages |
| P11.2 | Shifts, weekly off days, holidays | 3.10 | Shift Type, Holiday List ✅ | ⚙️ | Setup | Settings |
| P11.3 | Attendance from the machine (7.11) | 3.10, 5 | Employee Checkin, auto attendance ✅ | 🔴 `pk_integrations` | Sync on S11 | Attendance calendar |
| P11.4 | Leave types, balances, requests with approval | 3.10, 3.13 | Leave Type, Policy, Application ✅ | ⚙️ | Workflow on S6 | Leave pages |
| P11.5 | Salary setup: basic, allowances, fixed deductions | 3.10 | Salary Structure ✅ | ⚙️ | Setup | Salary setup |
| P11.6 | Late, absent and half-day deduction rules (7.12) | 3.10 | — | 🔴 | Rules + tests vs manual calculation | Breakdown view |
| P11.7 | Advances and loans recovered from salary | 3.10 | Employee Advance, Lending Loan ✅ | ⚙️ | Setup + tests | Advances page |
| P11.8 | Overtime from attendance | 3.10 | Overtime Type/Slip ✅ | ⚙️ | Setup + tests | — |
| P11.9 | Bonus and sales commission added to salary | 3.10 | Additional Salary ✅ | 🔴 | Commission from sales (P6.4) | — |
| P11.10 | Monthly salary sheet, payslips (email), salary payments, **owner approval** | 3.10, 3.12 | Payroll Entry, Salary Slip ✅ | ⚙️ | Workflow (S6), payslip format (S8) | Payroll run page |
| P11.11 | HR reports: attendance, late/absent summary, leave balances, salary sheet, advances outstanding | 4 | HRMS reports ✅ | 🔴 wrappers | Report APIs | Report pages |

**Exit criteria:** G1–G12 · one full test month matches a manual calculation, including late deduction, overtime, loan instalment and commission.

---

### Phase 12 — Dashboards, alerts, owner summary, report check

**Needs before start:** P3–P11 (their data and reports).

| ID | Step | SRS | ERPNext gives | Type | Our backend | Next.js |
|---|---|---|---|---|---|---|
| P12.1 | Owner dashboard: today's sales by channel, orders by stage, cash in hand, pending payments, low stock, top products | 3.14 | Number Card, Dashboard Chart ✅ | 🔴 | Aggregated API (fast) | Dashboard |
| P12.2 | Monthly sales and profit vs last month | 3.14 | Gross Profit ✅ | 🔴 | API | Trend chart |
| P12.3 | Dashboards for store, accounts and HR staff (their pending work) | 3.14 | — | 🔴 | APIs | 3 dashboards |
| P12.4 | Owner's daily summary email and one-page report (7.13) | 3.12, 4 | Email Digest ✅ | 🔴 | Daily job + template | Summary page |
| P12.5 | Pending approvals: all types in one place (each phase added its type; check all are there) | 3.14 | — | — | Test | Inbox |
| P12.6 | Report conformance: all report rows (old plan 19.4) filter and export correctly | 4 | — | — | Test list | — |

**Exit criteria:** owner reviews the dashboard with real test data; numbers equal the reports.

---

### Phase 13 — Migration, UAT, training, go-live, support

| ID | Step | SRS | Notes |
|---|---|---|---|
| P13.1 | Data migration dry runs: products, prices, barcodes, customers, suppliers, opening balances, opening stock (tools from P2.8, P3.1, P3.2, P4.4, P7.8) | 3.3, 3.5, 3.6, 3.9 | Two dry runs, then the real run |
| P13.2 | Deployment: Docker images, one domain, Nginx, SSL | 1.3, 7 | Hosting decided in P0 |
| P13.3 | Daily automatic backups (D15) + restore test | 6 | |
| P13.4 | Performance test with large data (e.g. 20k products, 100k orders) | 6 | Every list API paginated |
| P13.5 | Security review (Part 11 of the plan) | 6 | |
| P13.6 | UAT per role, written sign-off | — | |
| P13.7 | Training per role and a simple user guide per role | 6 | Guides written during each phase (G11) |
| P13.8 | Cut-over and website switch from WordPress (redirects) | 3.11 | |
| P13.9 | Support after go-live (hypercare) | 6 | |

---

## 8. Traceability matrix (every SRS line → step)

Status today: ✅ built · 🟡 partly · ❌ not built.

### 1–2 Scope and roles
| SRS | Requirement | Step(s) | Today |
|---|---|---|---|
| 1.3 | Admin portal for all back-office work | P1–P12 | 🟡 |
| 1.3 | POS screen with barcode scanning | P6.2 | ❌ |
| 1.3 | New website replacing adeeltraders.com | P9, P13.8 | ❌ |
| 1.3 | Integrations: PostEx, attendance machine, email, scanners and printers | P8.6, P11.3, P1.7, P3.9, P6.2 | 🟡 |
| 2.1 | Owner creates custom roles and picks view/create/edit/delete/approve/export | P1.2 | ❌ |
| 2.1 | Roles: Owner, Manager, Counter, Store, Accountant, HR, Online team, Customer | P1.1, P9B.1 | 🟡 |
| 2.1 | Counter staff: own branch only, cannot change prices | P1.1, P1.4, P6.2 | ❌ |
| 2.1 | Manager: no payroll by default | P1.1 | ❌ |

### 3.1 Products
| # | Requirement | Step(s) | Today |
|---|---|---|---|
| 1 | Name, brand, category, images, description, barcode | built; P3.10 | ✅ |
| 2 | Variants (shade, size, fragrance) with own barcode, price, stock | built | ✅ |
| 3 | Categories and brands managed in admin | built | ✅ |
| 4 | Show on website; featured, new arrival, on sale | P3.6, P9A.2 | ❌ |
| 5 | Excel upload and download of products and prices (incl. brands and categories) | P3.1, P3.1b | 🟡 |
| 6 | Fast search by name, barcode, SKU, brand, category | built; P3.10 | ✅ |
| 7 | Activate / deactivate, history kept | built | ✅ |
| 8 | Bundles / gift sets; stock deducted from each item | built; P6.10 | 🟡 |

### 3.2 Units and pricing
| # | Requirement | Step(s) | Today |
|---|---|---|---|
| 1 | Piece, Dozen, Box, Carton with conversions (1 Carton = 12 Boxes = 144 Pieces) | units exist, only Box set; P3.11 | 🟡 |
| 2 | Buy and sell in any unit, stock converted | P5.12, P6.2/P6.3 | 🟡 |
| 3 | Wholesale, Online, Self-delivery prices | built | ✅ |
| 4 | Cost and margin for each price; warning below cost | built (margin); P2.10, P6.1 | 🟡 |
| 5 | Discounts on items or bills, limit per role | P6.1, P1.6 | ❌ |
| 6 | Dealer special prices; website sale prices; coupons | P6.1, P9B.2 | ❌ |

### 3.3 Inventory and stock
| # | Requirement | Step(s) | Today |
|---|---|---|---|
| 1 | One central stock, real time | built; P6.10 | ✅ |
| 2 | Stock per branch/warehouse, transfers | built; P1.4, P3.7 | 🟡 |
| 3 | Stock held for confirmed orders (no overselling) | setting on; P6.10, P9B.2 | 🟡 |
| 4 | Adjustments (damaged, expired, tester, lost) with reason | built | ✅ |
| 5 | Stock count by scan + difference report | built | ✅ |
| 6 | Minimum stock alerts; levels managed in Settings | P3.3, P3.4 | 🟡 |
| 7 | Full stock history with document | built | ✅ |
| 8 | Opening stock entry | P3.2, P13.1 | ❌ |
| 9 | Stock value at cost per branch and total | built (per location); P3.7 | 🟡 |
| 10 | Damaged and returned kept apart until checked | built; P6.7, P8.7 | ✅ |

### 3.4 Barcode
| # | Requirement | Step(s) | Today |
|---|---|---|---|
| 1 | Use existing barcode or generate one | built | ✅ |
| 2 | Scan at counter, receiving, count, packing | count built; P6.2, P5.6, P8.4 (S13 P3.8) | 🟡 |
| 3 | Print labels with name and price | built (browser); P3.9 | 🟡 |

### 3.5 Suppliers and purchasing
| # | Requirement | Step(s) | Today |
|---|---|---|---|
| 1 | Supplier records, payment terms, opening balance | P4.3, P4.4, P2.9 | ❌ |
| 2 | Reorder list with suggested quantities | P5.1 | ❌ |
| 3 | RFQ and comparison | P5.2 | ❌ |
| 4 | PO, then receiving (full or partial) | P5.3, P5.6 | ❌ |
| 5 | Purchase bills and returns | P5.7, P5.9 | ❌ |
| 6 | Supplier payments and ledger | P4.5, P4.6, P5.10 | ❌ |
| 7 | PO approval before sending | P5.3, P1.6 | ❌ |
| 8 | Email or print PO | P5.4, P1.7, P1.8 | ❌ |
| 9 | Last purchase price per supplier | P5.5 | ❌ |
| 10 | Cost updated from purchases incl. freight | P5.8 | ❌ |

### 3.6 Customers and dealers
| # | Requirement | Step(s) | Today |
|---|---|---|---|
| 1 | Customer types get their price automatically | P2.9 (groups exist), P4.1 | 🟡 |
| 2 | Details, several addresses, purchase history | P4.1 | ❌ |
| 3 | Credit limit; above limit needs manager approval | P4.2, P6.3, P1.6 | ❌ |
| 4 | Ledger and printable/emailable statement | P4.6 | ❌ |
| 5 | Pending payments by age 30/60/90+ | P4.7 | ❌ |
| 6 | Opening balance per customer and dealer | P4.4, P13.1 | ❌ |
| 7 | Email reminders for overdue dealers | P4.8 | ❌ |
| 8 | Top customers; customers inactive for long | P4.9 | ❌ |

### 3.7 Sales
| # | Requirement | Step(s) | Today |
|---|---|---|---|
| 1 | POS: scan, bill, payment, receipt | P6.2 | ❌ |
| 2 | Hold and recall bills; daily cash closing | P6.2 | ❌ |
| 3 | Wholesale: quotation → order → delivery → invoice | P6.3 | ❌ |
| 4 | Self-delivery orders to riders, cash against rider | P6.5 | ❌ |
| 5 | Sales returns and exchanges | P6.7 | ❌ |
| 6 | Branded invoice, challan, receipt | P6.8, P1.8 | ❌ |
| 7 | Cash, card, transfer, JazzCash, Easypaisa, split | P2.3, P6.9 | ❌ |
| 8 | Credit sales to dealer ledger automatically | P6.3 | ❌ |
| 9 | Salesperson on every sale | P6.4 | ❌ |
| 10 | Quotation/invoice PDF, print, email, WhatsApp; reprint | P6.8, P1.8 | ❌ |
| 11 | Rider cash settlement | P6.5 | ❌ |

### 3.8 Online orders and delivery
| # | Requirement | Step(s) | Today |
|---|---|---|---|
| 1 | Website, phone and social orders in one list | P8.2, P8.3 | ❌ |
| 2 | Order stages | P8.1 | ❌ |
| 3 | Book PostEx, print labels | P8.6 | ❌ |
| 4 | Tracking updated from PostEx | P8.6 | ❌ |
| 5 | Match PostEx COD payments | P8.6 | ❌ |
| 6 | Returned parcels back to stock | P8.7 | ❌ |
| 7 | Packing by barcode scan | P8.4 | ❌ |
| 8 | Pick lists and packing slips | P8.4, P8.5 | ❌ |
| 9 | Cancel with reason, release stock | P8.8 | ❌ |
| 10 | Flag customers who return or refuse COD | P8.9 | ❌ |

### 3.9 Accounts and expenses
| # | Requirement | Step(s) | Today |
|---|---|---|---|
| 1 | Cash and bank accounts incl. JazzCash and Easypaisa, with balances | P2.1, P7.1 | ❌ |
| 2 | Receipts from customers, payments to suppliers and others | P4.5, P7.2 | ❌ |
| 3 | Expenses by category with bill photo | P2.1, P7.3 | ❌ |
| 4 | Transfers between accounts; daily cash book | P7.4, P7.5 | ❌ |
| 5 | Customer, supplier and account ledgers | P4.6, P7.10 | ❌ |
| 6 | Opening balances for cash and bank | P2.8, P7.8, P13.1 | ❌ |
| 7 | Cheques received and issued, clearing status | P7.6 | ❌ |
| 8 | Petty cash | P2.1, P7.7 | ❌ |
| 9 | Expense approval | P7.3, P1.6 | ❌ |
| 10 | Simple monthly profit and loss | P7.9 | ❌ |

### 3.10 Employees, attendance and salaries
| # | Requirement | Step(s) | Today |
|---|---|---|---|
| 1 | Employee records | P11.1 | ❌ |
| 2 | Salary setup | P11.5 | ❌ |
| 3 | Shifts, off days, holidays | P11.2 | ❌ |
| 4 | Attendance from the machine | P11.3 | ❌ |
| 5 | Leave types, balances, requests with approval | P11.4 | ❌ |
| 6 | Late, absence, half-day deduction rules | P11.6 | ❌ |
| 7 | Advances and loans from salary | P11.7 | ❌ |
| 8 | Salary sheet, payslips, payments | P11.10 | ❌ |
| 9 | Overtime from attendance | P11.8 | ❌ |
| 10 | Bonus and sales commission | P11.9 | ❌ |
| 11 | Employee documents (CNIC, contract) | P11.1 | ❌ |
| 12 | Salary approval by owner | P11.10, P1.6 | ❌ |

### 3.11 Website
| # | Requirement | Step(s) | Today |
|---|---|---|---|
| 1 | Modern, fast, mobile; no separate product entry | P9A.1, P9A.2 | ❌ |
| 2 | Products, prices, stock live from admin | P9A.1 | ❌ |
| 3 | Shop by category and brand, search, filters (price, skin type, shade) | P9A.2, P3.6 | ❌ |
| 4 | Cart, checkout (COD + online), delivery by city, coupons | P9B.2, P6.6, P10.1 | ❌ |
| 5 | Customer account, order history, tracking | P9B.1, P9C.1 | ❌ |
| 6 | Home banners and sections managed in admin | P9A.3 | ❌ |
| 7 | SEO | P9A.5 | ❌ |
| 8 | Sign-up, login, password reset, saved addresses | P9B.1 | ❌ |
| 9 | Wishlist, reviews after approval, back-in-stock | P9B.3, P9B.4 | ❌ |
| 10 | About, contact, delivery, return pages editable | P9A.4 | ❌ |
| 11 | WhatsApp button, Google Analytics, Facebook Pixel | P9A.5 | ❌ |

### 3.12 Communication and alerts
| # | Requirement | Step(s) | Today |
|---|---|---|---|
| 1 | Customer emails at placed, confirmed, packed, dispatched, delivered | P8.10, P9C.2 | ❌ |
| 2 | Email invoices, statements, payslips | P6.8, P4.6, P11.10 | ❌ |
| 3 | Internal alerts for low stock and pending approvals | P3.4, P1.6, P1.7 | 🟡 |
| 4 | Daily summary email to the owner | P12.4 | ❌ |
| 5 | Email templates with logo, editable | P1.7 | ❌ |

### 3.13 Settings, users and security
| # | Requirement | Step(s) | Today |
|---|---|---|---|
| 1 | Company details, logo, invoice layout | P1.4, P1.8, P6.8 | ❌ |
| 2 | Users with roles and permissions | P1.1, P1.2, P1.3 | 🟡 |
| 3 | Minimum stock and alert settings | P3.3, P3.4 | 🟡 |
| 4 | Masters: units, categories, brands, expense types, leave types | P1.12, built, P2.1, P11.4 | 🟡 |
| 5 | PostEx, attendance machine, email settings | P8.6, P11.3, P1.7 | ❌ |
| 6 | Record of every change | P1.5 | 🟡 |
| 7 | Branch and warehouse setup | P1.4 | 🟡 |
| 8 | Numbering, tax, payment methods, delivery charges by city | P2.6, P2.4, P2.3, P6.6 | ❌ |
| 9 | Strong passwords, auto logout, login history | P1.3 | 🟡 |

### 3.14 Dashboard
| # | Requirement | Step(s) | Today |
|---|---|---|---|
| 1 | Owner dashboard | P12.1 | ❌ |
| 2 | Pending approvals in one place (PO, discounts, credit, expenses, payroll) | P1.6, P12.5 | ❌ |
| 3 | Monthly trend vs last month | P12.2 | ❌ |
| 4 | Store, accounts and HR dashboards | P12.3 | ❌ |

### 4 Reports
| # | Requirement | Step(s) | Today |
|---|---|---|---|
| 0 | Every report filters by date, branch, channel, category, brand; exports to Excel/PDF | P1.9, P1.10, P12.6 | ❌ |
| 1 | Stock reports | P3.5 | ❌ |
| 2 | Sales reports (incl. channel, salesperson, profit) | P6.12 | ❌ |
| 3 | Online order reports | P8.11 | ❌ |
| 4 | Purchase reports | P5.11 | ❌ |
| 5 | Expense reports | P7.10 | ❌ |
| 6 | Accounts reports (ledgers, ageing, cash/bank book, cash closing, P&L) | P4.6, P4.7, P6.12, P7.5, P7.9, P7.10 | ❌ |
| 7 | HR reports | P11.11 | ❌ |
| 8 | Customer reports (top, stopped buying, sales by city) | P4.9, P6.12 | ❌ |
| 9 | Staff reports (by salesperson, cash by rider) | P6.12 | ❌ |
| 10 | Owner's daily summary | P12.4 | ❌ |

### 5 Integrations
| Requirement | Step(s) | Today |
|---|---|---|
| PostEx | P8.6 | ❌ |
| Attendance machine | P11.3 | ❌ |
| Email | P1.7 | ❌ |
| Scanners, label printers, thermal receipt printers | built (scanner); P3.9, P6.2 | 🟡 |
| Payment gateway | P10.1 | ❌ |

### 6 General
| Requirement | Step(s) | Today |
|---|---|---|
| Any modern browser, tablet, mobile | gate G8 (every phase) | 🟡 |
| Secure login, each role sees only what it may | P1.1, P1.3 | 🟡 |
| Daily automatic backups | P13.3 | ❌ |
| Fast with many products and orders | gate rule (paginated APIs), P13.4 | 🟡 |
| Cancelled records kept, never deleted | ✅ Frappe; gate G3 checks no hard delete of submitted documents | ✅ |
| A4 and 80 mm printing | P1.8, P6.8 | ❌ |
| Any list or report exported to Excel | P1.9, gate G7 | ❌ |
| Training and user guide per role | P13.7, gate G11 | ❌ |
| Support after go-live | P13.9 | ❌ |

**Count:** 150 requirement lines. Every one has at least one step. 0 lines without a phase.

---

## 9. What changes compared to the old plan

| Old plan | New roadmap | Why |
|---|---|---|
| Approvals page in Phase 9 | Approvals engine in **Phase 1** | Discounts, PO, credit, expenses and payroll all need it |
| Accounts module had **no phase** | **Phase 2** (foundation) + **Phase 7** (module) | Transactions need accounts; screens can come later |
| Customer groups in two phases | Once, in **Phase 2/4** | No duplicate work |
| Reports all in Phase 9 | Each module ships its reports; Phase 12 only checks them | Gate G7 |
| Wishlist, reviews, skin-type filter, exchanges, WhatsApp share, custom roles, login history, export, opening stock screen were missing | Added (P9B.3, P3.6, P6.7, P1.8, P1.2, P1.3, P1.9, P3.2) | SRS lines with no step |
| Builder and offsite_backups assumed | Decisions D14, D15 | Not installed |
| "Done" = screen exists | 12-check phase gate | Rework prevention |

Nothing from the old plan was removed. Every old task maps to a step above (old Phase 2 → P3, old 3 → P4/P5, old 4 → P6, old 5 → P8, old 6 → P9, old 7 → P10, old 8 → P11, old 9 → P12, old 10–12 → P13; old Phase 1 → P1 + P2).

---

## 10. Effort

Effort is re-estimated **after decision D1** (how many screens are Next.js) and D2 (POS). The old figure (130–218 dev-days, `ARCHITECTURE_REVIEW.md` §7) assumed only 5 Next.js screens. Choosing Next.js for every daily-use screen adds front-end work in P4–P8 and P11.

---

## 11. Next step

1. You review this roadmap and the decisions in section 4.
2. After approval: Phase 0 (stabilise, fix I1–I18, close decisions), then Phase 1.
3. `IMPLEMENTATION_PLAN.md` Part 6 and 19.3 are replaced by sections 7 and 8 of this file.
