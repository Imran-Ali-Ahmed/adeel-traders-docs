# Adeel Traders ERP — Implementation Plan

> **CONFIDENTIAL — for Nesway and Adeel Traders only.**
> Based on: *Adeel Traders — Software Requirements Specification v1.0 (28 Oct 2026)*.
> Platform: **Frappe Framework v16 + ERPNext v16** and official Frappe apps.
> Plan version: **1.2** · Created: 10 Nov 2026 · Owner: Nesway dev team
>
> **Change log**
> - v1.1 — Print Designer dropped (not v16), Builder/Insights v16 confirmed, Sales Channel auto-fill, backups, payments, PostEx scope, estimates marked as planning.
> - v1.2 — Accounting design decision (1.4), real approval workflow (7.17), product margin view now **mandatory**, phone/social order entry, exact stock-reservation timing, barcode wording fixed, full report-by-report matrix (19.4), acceptance criteria (19.5), Builder pinned to a commit, 6 end-to-end PoC flows, estimate total corrected to **69–127**.
>
> ⚠️ The SRS is still a **draft awaiting client sign-off**. Items marked ❓ are assumptions until the client answers.

---

## 📖 How to use this document

| Step | What to do |
|---|---|
| 1 | Read **Part 1–3** once. They explain the goal, the rules and the system design |
| 2 | Work **phase by phase** (Part 6). Do not start a phase before the previous one is "Done" |
| 3 | Tick the `[ ]` boxes as you finish tasks. Commit this file to Git so everyone sees progress |
| 4 | When you build a custom feature, open its spec in **Part 7** |
| 5 | When you build any screen, print format or web page, follow **Part 8 (UI/UX)** |
| 6 | Before go-live, every item in **Part 11 (Security)** and **Part 15 (Go-live)** must be ✅ |

### Legend

| Icon | Meaning |
|---|---|
| ✅ | Built into ERPNext/Frappe. **Only configuration** |
| 🟡 | Built-in **plus small custom work** |
| 🔴 | **Custom development** |
| ➕ | Extra feature (not in the SRS) that adds real value |
| ⚠️ | Risk or "must test first" |
| ❓ | Waiting for an answer from the client |

---

## Table of contents

1. Goals and golden rules
2. System architecture
3. Apps we will use
4. Team, tools and way of working
5. Timeline overview
6. Phases (step by step)
7. Custom development specifications
8. UI / UX design system (colours, fonts, screens, print, website)
9. Configuration blueprint (settings and masters)
10. Integrations
11. Security plan
12. Performance and scalability plan
13. Testing plan
14. Data migration plan
15. Deployment, go-live and support
16. Training and documentation
17. Risks and how we handle them
18. Open questions for the client
19. Appendix: effort table, naming rules, checklists

---

# Part 1 — Goals and golden rules

## 1.1 What we are building

```
                 ┌───────────────────────────────────────────┐
                 │        ONE DATABASE · ONE STOCK            │
                 └───────────────────────────────────────────┘
        ┌──────────────┐   ┌──────────────┐   ┌──────────────────┐
        │ Admin portal │   │  POS screen  │   │ E-commerce site  │
        │ (Desk)       │   │  (counter)   │   │ (Webshop)        │
        │ owner, staff │   │ cashier      │   │ public customers │
        └──────────────┘   └──────────────┘   └──────────────────┘
```

A sale on the **website**, at the **counter**, or to a **dealer** reduces the **same stock** at once.

## 1.2 Project goals

| # | Goal | How we measure it |
|---|---|---|
| G1 | Cover every SRS requirement | Requirement checklist (Part 19.3) is 100% ✅ |
| G2 | **Secure** | Security checklist (Part 11) is 100% ✅ before go-live |
| G3 | **Scalable** | Pages load < 2 s with 5,000 SKUs and 500 orders/day (Part 12) |
| G4 | **Upgrade-safe** | Zero changes inside `frappe/` or `erpnext/` folders |
| G5 | Easy for staff | Each role finishes its main task after a 1-hour training |

## 1.3 Golden rules (never break these)

1. 🚫 **Never edit core code** (`apps/frappe`, `apps/erpnext`, `apps/hrms`, …). All changes go in **our custom app**.
2. 📦 **Everything is code.** Custom Fields, Property Setters, Workflows, Roles, Print Formats → exported as **fixtures** to Git.
3. 🧪 **No feature without a test** (at least one happy-path test).
4. 🔀 **Every change goes through a Pull Request** and one review.
5. 🧭 **Config first, code second.** Before writing code, check if ERPNext can do it with settings.
6. 🔒 **Least privilege.** Each user gets only the roles they need.
7. 📌 **Pin versions.** All apps on `version-16` branches (or a fixed tag). Never deploy `develop`.
8. 🗂️ **Staging before production.** Every release is tested on staging first.

## 1.3.1 How we customise without touching core (the "safety ladder")

Always use the **lowest step** that solves the problem. Go up only when the step below cannot do it.

```
Step 1  Settings / configuration (no code)               ← safest
Step 2  Customize Form: Custom Field, Property Setter     (exported as fixtures)
Step 3  Workflow, Notification, Print Format, Report, Workspace (as fixtures)
Step 4  New DocTypes + new pages/reports in OUR app
Step 5  Hooks in OUR app: doc_events, doctype_js, scheduler_events
Step 6  extend_doctype_class (v16) — add methods to a core class
Step 7  override_doctype_class / override_whitelisted_methods  ← last resort, needs tech-lead approval
🚫     Editing files in apps/frappe, apps/erpnext, apps/hrms, apps/webshop — NEVER
```

| Feature | Step used |
|---|---|
| Price lists, stock reservation, FEFO, credit limits, roles, 2FA | 1 |
| `order_stage`, `order_source`, `adjustment_reason`, `needs_approval`, "new arrival" tags | 2 |
| PO / expense / payroll / discount approvals, order emails, invoices, labels, workspaces | 3 |
| PostEx Settings + Log, Rider Cash Settlement, Role Discount Limit, City Delivery Charge, Back In Stock Request, custom reports | 4 |
| Discount check, approval flags, Sales Channel auto-fill, city charge on cart, PostEx buttons, tracking sync job, back-in-stock email | 5 |
| Payment gateway | 4 + 5 (own gateway doctype registered through the `payments` app's normal API) |
| Website look | Website Theme + Builder pages + our own templates. ⚠️ Any change to Webshop pages must use supported hooks/template overrides from our app — verify in PoC |
| Step 6 / 7 | **None planned today.** If ever needed, write down why, and add a test that fails if the core method changes after an upgrade |

**Upgrade check:** after every Frappe/ERPNext minor update, run our test suite on staging before production.

## 1.4 Two decisions that need written client approval

### A. Technology change
The SRS (section 7) names **Next.js + NestJS + PostgreSQL**. This plan uses **Frappe v16 + ERPNext v16 + MariaDB**.

| Why we change | What the client must know |
|---|---|
| Stock, accounting, POS, purchasing, HR and payroll already exist and are tested | The admin portal looks like ERPNext (branded), not a fully custom Next.js design |
| Much less custom code → lower risk and cost | Database is **MariaDB**. ERPNext v16 does not support PostgreSQL in production |
| Open source (GPL v3), no licence fee | Future features follow the Frappe ecosystem |

- [ ] Write a 1-page "technology change" note and get the client's **written approval** before Phase 1.

### B. Accounting design (SRS says "full double-entry is out of scope")

ERPNext **always** keeps double-entry records. We cannot switch that off. So we split it into 3 layers:

```
┌──────────────────────────────────────────────────────────────┐
│ 1. What staff see  (SIMPLE)                                  │
│    Receive payment · Pay supplier · Add expense · Transfer   │
│    Cash closing · Customer / supplier ledger                 │
├──────────────────────────────────────────────────────────────┤
│ 2. What ERPNext does in the background  (AUTOMATIC)          │
│    Every submit → General Ledger debit/credit entries        │
│    Stock moves → stock value + cost of goods sold            │
├──────────────────────────────────────────────────────────────┤
│ 3. What the accountant checks  (REVIEW)                      │
│    Chart of Accounts, P&L, Balance Sheet, month-end close    │
└──────────────────────────────────────────────────────────────┘
```

| Rule | Detail |
|---|---|
| Staff never make Journal Entries | They use Payment Entry, Expense (Purchase Invoice / simple Journal template), POS closing |
| Simple workspace | Accounts workspace shows only the simple actions + ledgers (Part 8.5) |
| Accountant owns the setup | Chart of Accounts, tax, valuation method, month-end checklist |
| Bonus for the client | Balance Sheet and full GL come "for free" — but nobody is forced to use them |

- [ ] Get the **accountant's written approval** of this design.

---

# Part 2 — System architecture

## 2.1 Big picture

```
                         Internet (HTTPS only)
                                │
                     ┌──────────▼──────────┐
                     │  Nginx (frontend)   │  SSL, gzip, static files
                     └──────────┬──────────┘
              ┌─────────────────┼──────────────────┐
              ▼                 ▼                  ▼
      ┌──────────────┐  ┌──────────────┐   ┌──────────────┐
      │ Gunicorn     │  │ Socket.io    │   │ Workers      │
      │ (web/API)    │  │ (realtime)   │   │ short / long │
      └──────┬───────┘  └──────┬───────┘   └──────┬───────┘
             │                 │                  │
      ┌──────▼─────────────────▼──────────────────▼──────┐
      │   MariaDB (data)   ·   Redis cache   ·   Redis queue │
      └───────────────────────────────────────────────────┘
             ▲                                     ▲
     Scheduler (cron jobs)              External APIs: PostEx,
     PostEx tracking sync, backups      payment gateway, SMTP,
                                        attendance machine sync
```

## 2.2 Environments

| Environment | Purpose | Data | Who uses it |
|---|---|---|---|
| 🧑‍💻 **Local** | Each developer codes here (frappe_docker / bench) | Fake test data | Developers |
| 🧪 **Staging** | Test every release, UAT with client | Copy of production (masked) | QA, client key users |
| 🚀 **Production** | Live business | Real data | Adeel Traders staff + customers |

```
feature branch ──PR──► develop ──deploy──► Staging ──UAT OK──► main ──deploy──► Production
```

## 2.3 Server sizing (starting point)

| Item | Staging | Production (start) | Notes |
|---|---|---|---|
| vCPU | 2 | 4 | Add CPUs when workers are busy |
| RAM | 4 GB | 8–16 GB | MariaDB buffer pool ≈ 50% of RAM |
| Disk | 60 GB SSD | 160 GB SSD | Images and backups grow fast |
| OS | Ubuntu LTS | Ubuntu LTS | Or Docker on any Linux |
| Hosting | Same provider | Pakistan-friendly cloud or Frappe Cloud | ❓ Decide with client |

⚠️ Choose **Docker (frappe_docker custom image)** or **Frappe Cloud**. Recommendation: **Docker custom image** if Nesway manages the server; **Frappe Cloud** if the client wants managed hosting and automatic backups.

---

# Part 3 — Apps we will use

## 3.1 App stack

| App | Branch | Why | Status |
|---|---|---|---|
| `frappe` | version-16 | Framework | ✅ |
| `erpnext` | version-16 | Stock, selling, buying, accounts, POS | ✅ |
| `hrms` | version-16 | Attendance, leave, payroll, overtime | ✅ |
| `lending` | version-16 | Staff loans repaid from salary | ✅ |
| `payments` | version-16 | Payment gateway framework (we add our gateway) | ✅ |
| `webshop` | version-16 | E-commerce website | ✅ |
| `builder` | **pinned commit** of `master` (record the hash in `apps.json`) | Drag-and-drop website pages, banners | ✅ Supports v16 from **16.36.1** (pyproject note). We run 16.55. `master` moves, so never build from the branch name |
| ~~`print_designer`~~ | — | ❌ **Not used.** Its README says it supports only Frappe **develop and v15**, not v16 | 🚫 |
| `offsite_backups` | version-16 | Backups to S3 / Google Drive / Dropbox | ✅ |
| `blog` | version-16 | SEO articles (➕) | ✅ optional |
| `newsletter` | version-16 | "Get offers by email" (➕) | ✅ optional |
| `insights` | **version-3** | Advanced BI dashboards (➕) | ✅ optional (pyproject: Frappe ≥15, <17) |

> **Print formats and labels:** we use standard **Print Format (Jinja HTML)**. Barcodes come from Frappe's built-in **Barcode** field type (stores the barcode as an image), or from a small barcode library inside our app. See spec 7.6.
| **`adeel_traders`** | our repo | All client-specific customisation | 🔴 we build |
| **`pk_integrations`** | our repo | PostEx + Pakistani payment gateways (reusable for other clients) | 🔴 we build |

## 3.2 Our custom apps

```
adeel_traders/                     ← client-specific
├── adeel_traders/
│   ├── hooks.py                   ← doc_events, fixtures, scheduler, overrides
│   ├── at_selling/                ← discount limits, rider settlement, channel logic
│   ├── at_stock/                  ← adjustment reasons, barcode generator, back-in-stock
│   ├── at_online_orders/          ← order stage workflow helpers, COD risk
│   ├── at_hr/                     ← late-deduction rules, commission → salary
│   ├── at_dashboard/              ← owner dashboard, approvals page, daily summary
│   ├── at_website/                ← theme, city delivery charges, templates
│   ├── fixtures/                  ← custom fields, workflows, roles, print formats
│   ├── patches/                   ← data patches
│   └── tests/
└── pyproject.toml

pk_integrations/                   ← reusable for future Pakistani clients
├── postex/                        ← PostEx Settings, booking, tracking, COD reconciliation
├── jazzcash/                      ← gateway (if chosen)
└── easypaisa/                     ← gateway (if chosen)
```

**Why two apps?** PostEx and payment gateways are useful for **every** Pakistani client. Keeping them separate lets Nesway reuse them.

---

# Part 4 — Team, tools and way of working

## 4.1 Team (suggested)

| Role | Count | Main work |
|---|---|---|
| Project Manager / Business Analyst | 1 | Client contact, requirements, UAT, training plan |
| Frappe Backend Developer | 1–2 | Doctypes, hooks, integrations, reports |
| Frontend / Website Developer | 1 | Webshop theme, Builder pages, Vue/Jinja |
| UI/UX Designer | 0.5 | Design system, website, print formats |
| QA / Tester | 0.5 | Test cases, regression, UAT support |
| DevOps (can be a developer) | 0.25 | Servers, CI/CD, backups, monitoring |

## 4.2 Tools

| Need | Tool |
|---|---|
| Code | GitHub (private repos) |
| Tasks | GitHub Projects / Jira (one card per task in this plan) |
| CI | GitHub Actions: lint + tests on every PR |
| Code quality | `pre-commit` with **ruff** (Python), **eslint/prettier** (JS) — same as Frappe's own setup |
| Design | Figma |
| Docs | This file + user guides in `docs/` |
| Communication | Weekly client call + written MOM |

## 4.3 Git branching

| Branch | Purpose |
|---|---|
| `main` | Production code only |
| `develop` | Ready for staging |
| `feat/<short-name>` | One feature |
| `fix/<short-name>` | One bug fix |

Commit style: `feat(postex): book shipment from Shipment doctype` · `fix(pos): …` · `chore: …`

## 4.4 Definition of Done (every task)

- [ ] Works on local and staging
- [ ] Fixtures exported (`bench --site <site> export-fixtures`)
- [ ] Test written and passing
- [ ] Permissions checked (right roles can, wrong roles cannot)
- [ ] Reviewed and merged by PR
- [ ] User-guide note added if users will see it

---

# Part 5 — Timeline overview

Assumption: **2 backend developers + 1 frontend developer + part-time designer/QA**. Total build effort ≈ **75–137 dev-days** (see Appendix 19.1). ⚠️ The 14-week chart is a **planning assumption**, not a delivery promise. PostEx docs, payment provider, attendance machine, data quality and client answers can all move it.

```
Week:   1   2   3   4   5   6   7   8   9  10  11  12  13  14
P0  ███                                                      Foundation + PoC
P1      ██████                                               Core setup (company, accounts, roles)
P2          ██████                                           Products, pricing, stock, barcode
P3              ██████                                       Purchasing + customers
P4                  ████████                                 Sales, POS, riders
P5                      ████████                             Online orders + PostEx
P6              ████████████████████                         Website (design starts early)
P7                          ████████                         Payment gateway
P8                      ████████                             HR + payroll
P9                                  ██████                   Dashboards, reports, alerts
P10                                     ██████               Data migration + UAT
P11                                           ████           Training + go-live
P12                                               ████████   Hypercare (support)
```

| Milestone | Week | Sign-off by |
|---|---|---|
| M1 PoC approved | 1 | Nesway + client |
| M2 Back office (stock, buy, sell) on staging | 6 | Client key users |
| M3 Online orders + PostEx on staging | 8 | Online order team |
| M4 Website + payments on staging | 9 | Owner |
| M5 HR/payroll on staging | 8 | HR |
| M6 UAT passed | 11 | Owner (written) |
| M7 Go-live | 12 | Owner |
| M8 Hypercare closed | 14 | Owner |

---

# Part 6 — Phases (step by step)

Each phase has: 🎯 goal · ✅ tasks · 📦 deliverables · 🏁 done when.

## Phase 0 — Foundation and PoC (Week 1)

🎯 Prove the risky parts work, set up the team's tools.

**Setup**
- [ ] Create GitHub repos `adeel_traders`, `pk_integrations`
- [ ] `bench new-app adeel_traders` and `bench new-app pk_integrations`
- [ ] Add pre-commit (ruff, eslint, prettier), GitHub Actions CI
- [ ] Local dev environment for each developer (frappe_docker devcontainer or bench)
- [ ] ⚠️ The current `pwd.yml` setup is a **demo only** (for watching and testing features). Never write code inside its containers: changes are lost when containers are recreated. All code lives in our Git repos, mounted into the dev bench
- [ ] Staging server with all apps from Part 3 on `version-16`
- [ ] `apps.json` file listing all apps + branches (used to build the Docker image)

**PoC: 6 end-to-end business flows (⚠️ must pass before Phase 1)**

A feature that *exists* is not the same as a business flow that *works*. The PoC proves full flows:

| # | Flow | Pass when |
|---|---|---|
| F1 | Product → 3 prices → UOM (piece/box) → barcode → stock | Correct price per customer type and per unit; scan finds the item |
| F2 | Website order → stock reservation → packing scan → cancel | Reserved on order, released on cancel, wrong item blocked at packing |
| F3 | POS sale → split payment → return → cash closing | Closing totals match payments |
| F4 | Wholesale credit sale above limit → **manager approval** → invoice → payment | Approval recorded (who + when) before submit |
| F5 | PostEx booking → tracking → COD reconciliation (sandbox) | Status and payout match |
| F6 | Attendance → late deduction → overtime → salary → loan recovery | Salary matches a manual calculation |

**Technical PoC checks**
- [ ] Install `webshop`, `hrms`, `lending`, `payments`, `builder`, `offsite_backups` on v16 → no errors in `bench migrate`
- [ ] Create 3 Price Lists + Customer Groups → price changes per customer type
- [ ] Create a Product Bundle → selling it reduces stock of each part (the **bundle item itself must be a non-stock item** — ERPNext blocks a stock item as parent)
- [ ] Sales Order **Workflow** + Webshop: Webshop submits the order in code — check the workflow does not block it
- [ ] Item with **Batch + Expiry** → FEFO picking works
- [ ] Place a website order → Sales Order is created **and stock is reserved**
- [ ] Accounting Dimension "Sales Channel" → P&L filtered by channel
- [ ] Sales Channel is filled **automatically** for POS, wholesale and website orders (spec 7.16)
- [ ] Jinja label format with a barcode prints correctly on the client's label printer
- [ ] Stock reservation edge cases: partial stock, many warehouses, cancel → release, 2 orders for the last unit at the same time
- [ ] POS full day: hold (draft) → recall → payment → return → closing
- [ ] Attendance machine: identify model ❓ and test sync tool if ZKTeco-compatible

📦 PoC report (1 page: pass/fail per item).
🏁 Owner and Nesway agree on the platform and final scope.

---

## Phase 1 — Core setup (Weeks 2–3)

🎯 Company, accounts, users and security ready.

**Company and accounts**
- [ ] Company "Adeel Traders", currency **PKR**, country Pakistan, fiscal year **July–June** ❓ confirm
- [ ] Chart of Accounts (Part 9.2) including cash, banks, JazzCash, Easypaisa, petty cash, COD receivable from PostEx
- [ ] Cost Centers: one per branch ❓ number of branches
- [ ] **Accounting Dimension: Sales Channel** (Wholesale / Online / Shop-Self-delivery) on Sales Order, Sales Invoice, Delivery Note, POS Invoice, Payment Entry, Journal Entry
- [ ] **Auto-fill the channel** on every document (spec 7.16). Creating the dimension alone does **not** fill it. Old transactions are not filled either
- [ ] Tax templates (GST) ❓ client answer pending
- [ ] Modes of Payment: Cash, Card, Bank Transfer, JazzCash, Easypaisa, Cheque, PostEx COD
- [ ] Naming series for every document (Part 19.2)
- [ ] Letter Head + company logo

**Users, roles, security**
- [ ] Create roles from Part 9.6 (Owner, Manager, Counter Staff, Store, Accountant, HR, Online Order Team)
- [ ] **Role Profiles** for each SRS role (one click to give all roles)
- [ ] **Module Profiles** (hide modules a role does not use)
- [ ] User Permissions: counter staff → only their branch/warehouse
- [ ] System Settings: password policy, session expiry, 2FA for Owner/Manager/Accountant, login attempts limit
- [ ] Per-user **login hours** for shop staff (➕), **IP restriction** where possible (➕)
- [ ] Enable **Track Changes** on all key doctypes (audit)

📦 Configured staging site + fixtures in Git.
🏁 Each test user logs in and sees only their modules.

---

## Phase 2 — Products, pricing, stock, barcode (Weeks 3–4)

🎯 Product catalogue and stock rules ready.

**Products**
- [ ] Item Groups (Makeup › Lips/Eyes/Face, Skin care, Hair care, Fragrance, Deals/Bundles)
- [ ] Brands
- [ ] Item Attributes: Shade, Size, Fragrance → variant templates
- [ ] UOMs: Piece, Dozen, Box, Carton + conversion on each item. ⚠️ "Buy and sell in **any** unit" means **any unit listed on that item** — each allowed unit must be added to the item's UOM table. If a box has a special price (not just 12 × piece price), add an **Item Price for that UOM**
- [ ] Item defaults (warehouse, income/expense accounts)
- [ ] Custom fields: `is_new_arrival`, `is_on_sale` (website badges) 🟡
- [ ] ➕ Batch + Expiry on items that expire; shelf life in days
- [ ] Product Bundles (gift sets) — parent item = **non-stock**; stock is kept on the child items

**Pricing**
- [ ] Price Lists: **Wholesale**, **Online**, **Self-delivery** (+ Standard Buying)
- [ ] Customer Groups → default price list (Wholesale Dealer, Online Customer, Walk-in)
- [ ] Pricing Rules for dealer special prices
- [ ] Selling Settings: **validate selling price vs cost** ON (blocks loss sales)
- [ ] Item "Max Discount %"
- [ ] 🔴 Role-based discount limit (spec 7.3) → over the limit goes to **approval**, not just an error (spec 7.17)
- [ ] 🔴 **Product Price & Margin view — MANDATORY** (SRS 3.2 asks for cost + margin for each price) (spec 7.14)
- [ ] ➕ Promotional Schemes (buy X get Y), Coupon Codes
- [ ] ➕ Loyalty Program (if client wants)

**Stock**
- [ ] Warehouses: Main Warehouse, Shop Counter, (other branches ❓), **Damaged**, **Returns – To Check**, **Testers**
- [ ] Stock Settings: enable **stock reservation** + **auto reserve stock**; negative stock **OFF**; batch pick = **Expiry** (FEFO)
- [ ] Stock Entry Types: Damaged, Expired, Tester, Lost, Transfer 🟡 + reason field
- [ ] Item Reorder levels + auto Material Request + email to Store/Manager
- [ ] Stock count procedure using Stock Reconciliation with **Scan Mode**

**Barcode**
- [ ] Import existing barcodes into Item Barcode table
- [ ] 🔴 Barcode generator for items without one (spec 7.6)
- [ ] Label print format (Jinja + barcode, spec 7.6) — sizes in Part 8.7

> **Phase 2 status, 2026-11-11 (checked line by line against SRS 3.1 to 3.4):**
> Done: products, shades/sizes/fragrances (generic variants), units and unit prices, 3 prices + margin, brands, description and photo, categories and brands admin (Settings, Catalogue), gift-set builder, stock by location (not-for-sale kept apart), movements with reason, batches + expiry, reorder level + auto request + email, stock value (whole business and per location), stock history, stock count by scanning with difference report (own `Stock Count` DocType that posts an ERPNext Stock Reconciliation), barcode generate/scan/print, import templates.
> Moved to their phase: dealer special prices, role discount limits + approval, coupons (Sales); featured / new arrival / on sale flags (Website); opening stock for batch products and scan-at-receiving (Purchasing); scan at counter / packing (POS / Online orders).
> Needs the client: label size and printer (Q7), SKU format (Q22), damaged-stock account (Q23), email account for the reorder email.

📦 Catalogue + pricing + stock rules on staging.
🏁 Store user receives, transfers, counts and prints labels using only the scanner.

---

## Phase 3 — Purchasing and customers (Weeks 4–5)

**Purchasing**
- [ ] Suppliers + payment terms + opening balances (Opening Invoice Creation Tool)
- [ ] Workflow: **Purchase Order approval** (Draft → Pending Approval → Approved → Sent)
- [ ] RFQ → Supplier Quotation → comparison report
- [ ] Purchase Receipt (GRN) with scan, partial receipts
- [ ] Purchase Invoice, Purchase Return
- [ ] Landed Cost Voucher for freight
- [ ] PO email template + print format
- [ ] ➕ Supplier Portal for online RFQ replies (optional)

**Customers and dealers**
- [ ] Customer Groups + default price lists
- [ ] Credit limits on dealers; set **Role allowed to bypass credit limit** = Manager
- [ ] Credit sale above limit → **approval workflow** (spec 7.17). Built-in ERPNext only blocks the user and offers to email the credit controller — that is a request, not a recorded approval
- [ ] Process Statement of Accounts with **auto email** (monthly)
- [ ] Dunning types (reminders for overdue)
- [ ] Customer opening balances
- [ ] ➕ Customer Portal for dealers (see orders, invoices, balance)
- [ ] ➕ Web Form "Become a dealer" → creates a Lead (or Frappe CRM)

🏁 Full buy cycle and a dealer credit sale pass test scripts.

---

## Phase 4 — Sales, POS and riders (Weeks 5–7)

**POS (shop counter)**
- [ ] POS Profile per counter: warehouse, price list (Self-delivery), payment methods, receipt format
- [ ] POS Settings: invoice type (**Sales Invoice** or POS Invoice) — choose one and keep it
- [ ] POS Opening/Closing Entry = daily cash closing
- [ ] Disable rate change for counter staff ("Allow User to Edit Rate" OFF)
- [ ] Hold/recall = Draft invoices; train staff
- [ ] Thermal receipt 80 mm format (Part 8.7)
- [ ] Barcode scanner tested in POS search box

**Wholesale**
- [ ] Quotation → Sales Order → Delivery Note → Sales Invoice
- [ ] Sales Team (salesperson) mandatory on Sales Order 🟡 (property setter)
- [ ] Branded quotation, invoice, delivery challan (A4, Jinja Print Formats)
- [ ] ➕ Auto Repeat for regular dealer orders
- [ ] ➕ Payment Terms (credit days)

**Self-delivery + riders**
- [ ] Driver records for each rider (linked to Employee)
- [ ] Delivery Trip for each rider's route
- [ ] 🔴 Rider cash settlement (spec 7.4)

**Returns**
- [ ] Sales Return → goes to **Returns – To Check** warehouse, then moved to sellable or Damaged

🏁 A cashier completes 10 POS sales + closing; a rider trip settles cash correctly.

---

## Phase 5 — Online orders and PostEx (Weeks 6–8)

- [ ] Order stages for online orders (spec 7.1)
- [ ] **Phone / WhatsApp / Instagram / Facebook orders**: staff create the Sales Order in desk with Channel = Online, Order Source = Phone/WhatsApp/…, customer, delivery address, Online price list → same list and stages as website orders
- [ ] Kanban board "Online Orders" by stage
- [ ] Pick List for the day's orders (one pick list, many orders) + packing by scan
- [ ] Packing slip print format
- [ ] ➕ Assignment Rule: new online orders shared among the order team (round-robin)
- [ ] 🔴 PostEx integration (spec 7.2): settings, city list, booking, label, tracking sync, COD reconciliation
- [ ] Return parcel flow → Sales Return → Returns – To Check
- [ ] Cancel order → stock reservation released (✅ built-in, test it)
- [ ] 🔴 COD risk flag report (spec 7.9)
- [ ] Order-stage emails to customers (Notification doctype)

🏁 50 test orders go Pending → Delivered; PostEx sandbox bookings and tracking work; COD payout matched.

---

## Phase 6 — Website (Weeks 3–9, runs in parallel)

**Design (starts Week 3)**
- [ ] Figma: home, category, product, cart, checkout, account, order tracking, policies, become-a-dealer (Part 8.8)
- [ ] Client approves designs before build

**Build**
- [ ] Webshop Settings: enable shopping cart, checkout, field filters + attribute filters, show stock availability, price list = Online
- [ ] Website Items for all online products, with images, specs, **recommended items**
- [ ] Website Theme + custom templates matching the design (Part 8.2)
- [ ] Home page sections and banners with **Builder** (editable by staff)
- [ ] Static pages: About, Contact, Delivery, Return policy (Web Page / Builder)
- [ ] Customer sign-up, login, password reset, addresses, order history (portal)
- [ ] Coupon codes in cart
- [ ] 🔴 Delivery charge by city (spec 7.7)
- [ ] 🔴 "Notify me when back in stock" (spec 7.8)
- [ ] 🔴 Order tracking page with PostEx status
- [ ] SEO: meta title/description per item & page, sitemap, clean URLs, alt text
- [ ] Google Analytics ID (Website Settings), Facebook Pixel + WhatsApp button (head/footer HTML)
- [ ] ➕ Blog app (SEO), Newsletter app (footer signup)
- [ ] Domain switch plan from WordPress (Part 14.4)

🏁 Lighthouse mobile score ≥ 80 (performance, accessibility, SEO); real order end-to-end.

---

## Phase 7 — Online payment gateway (Weeks 7–9)

- [ ] ❓ Client chooses provider (JazzCash / Easypaisa / card / Paymob Pakistan)
- [ ] Get merchant sandbox credentials
- [ ] 🔴 Build gateway in `pk_integrations` on the `payments` app pattern (spec 7.10)
- [ ] Connect to Webshop checkout (Payment Gateway Account)
- [ ] Test success, failure, cancel, duplicate callback, refund

🏁 100 sandbox payments reconcile to Payment Entries with zero mismatch.

---

## Phase 8 — HR and payroll (Weeks 6–8)

- [ ] Departments, Designations, Branches, Employees (+ CNIC, contract as attachments)
- [ ] Holiday Lists, Shift Types (grace period, half-day hours, absent hours), **Enable Auto Attendance**
- [ ] Attendance machine sync (spec 7.11) ❓ model
- [ ] Leave Types, Leave Policy, approvers
- [ ] Salary Components, Salary Structures, assignments
- [ ] Overtime Types (v16) → Overtime Slips
- [ ] Two **different** processes — define both with HR:

| | **Employee Advance** (HRMS) | **Employee Loan** (Lending app) |
|---|---|---|
| Use | Short advance, e.g. against salary or a trip | Bigger amount, paid back over months |
| Recovery | Deducted once (Additional Salary / advance settlement) | **Monthly instalments** on the salary slip ("Repay from Salary") |
| Report | Employee Advance Summary | Loan repayment schedule |
- [ ] 🔴 Late/absence deduction rules (spec 7.12) ❓ policy
- [ ] 🟡 Sales commission → Additional Salary (spec 7.12)
- [ ] Payroll Entry + **approval workflow** (owner approves)
- [ ] Payslip print format + email
- [ ] ➕ Frappe HR mobile app (PWA) for leave requests and check-in

🏁 One full month of test payroll matches a manual calculation.

---

## Phase 9 — Dashboards, reports and alerts (Weeks 9–10)

- [ ] Workspaces per role (Part 8.5)
- [ ] Owner dashboard: Number Cards + Charts (Part 8.6)
- [ ] 🟡 Pending approvals page (spec 7.5)
- [ ] Owner daily summary email (spec 7.13)
- [ ] Notifications: low stock, approvals, overdue dealers
- [ ] Auto Email Reports (weekly sales, stock value, ageing)
- [ ] Check every SRS report (Part 19.4) exists and filters by date / branch / channel / category / brand
- [ ] ➕ Insights dashboards (optional)

🏁 Owner reviews the dashboard and every report list item is ticked.

---

## Phase 10 — Data migration and UAT (Weeks 10–11)

- [ ] Migration dry run #1 on staging (Part 14)
- [ ] UAT scripts per role (Part 13.4) → client key users test
- [ ] Fix bugs → migration dry run #2
- [ ] Performance test (Part 12.3)
- [ ] Security checklist (Part 11) complete
- [ ] UAT sign-off (written)

---

## Phase 11 — Training and go-live (Week 12)

- [ ] Role-based training sessions (Part 16)
- [ ] Cut-over (Part 15.2): freeze old system, final migration, opening balances, DNS switch
- [ ] Go-live day checklist
- [ ] Hypercare starts

## Phase 12 — Hypercare (Weeks 12–14)

- [ ] Daily check-in with client for week 1, then 3× per week
- [ ] Bug triage: P1 within 4 hours, P2 within 1 day, P3 next release
- [ ] First month-end closing done together with the accountant
- [ ] Handover document + support agreement

---

# Part 7 — Custom development specifications

Each spec: **What → Where → How → Done when.** All code in `adeel_traders` unless noted.

## 7.1 Online order stages 🟡 (4–6 days)

**What:** Stages Pending → Confirmed → Packed → Booked → Dispatched → Delivered, plus Returned / Cancelled.

```
Pending ─► Confirmed ─► Packed ─► Booked ─► Dispatched ─► Delivered
   │           │          │         │           │
   └───────────┴──────────┴─────────┴───────────┴──► Cancelled / Returned
```

| Item | Detail |
|---|---|
| Doctype | Sales Order (only when Sales Channel = Online) |
| Fields | `order_stage` (Select, **Allow on Submit**) and `order_source` (Website / Phone / WhatsApp / Instagram / Facebook) — custom fields, shown in list + Kanban |
| Stage control | Server-side transition table (which stage → which next stage, which role). Not a Frappe Workflow, because the order is already **submitted** when stages change. Every change logged in the timeline (who + when) |
| Website orders | Webshop **submits** the Sales Order when the customer places it (verified in `webshop/shopping_cart/cart.py`) → stock is reserved **at "Pending"** |
| Phone/social orders | Staff click **Submit** when the order is confirmed → stock reserved then. Until submit = draft, no reservation |
| Cancel | Cancel the Sales Order → reservation released (built-in) → stage = Cancelled + reason (mandatory) |
| Automation | Packed → Pick List done · Booked → PostEx shipment created · Delivered → from PostEx tracking |
| Emails | Notification per stage to the customer |
| UI | Kanban "Online Orders", list filters, bulk actions: Mark Packed, Book with PostEx, Print labels |

🏁 Stage changes only through allowed roles; each stage sends the right email.

## 7.2 PostEx integration 🔴 (8–13 days) — `pk_integrations/postex`

```
Sales Order (Packed) ──► Shipment (built-in doctype) ──► PostEx API: create order
        │                         │                          │
        │                         ◄── tracking no + label ───┘
        ▼
  Scheduler (every 30 min) ──► PostEx: track status ──► update Shipment + order_stage
        ▼
  PostEx COD payout ──► match tracking no ──► draft Payment Entry ──► accountant submits
```

| Part | Detail |
|---|---|
| `PostEx Settings` (Single) | API token (Password field, encrypted), environment (sandbox/live), pickup address, default package |
| `PostEx City` | Cached list of serviceable cities (sync job) |
| Booking | Button on Shipment / bulk action on Sales Orders; validate city; save tracking no in **AWB Number**, URL in **Tracking URL**, status **Booked** |
| Label | Download PostEx label PDF → attach to Shipment; bulk print |
| Tracking | Scheduler job (background, **long queue**), only for open shipments; map PostEx statuses → order_stage |
| COD reconciliation | Import payout (API or CSV) → match tracking no → **draft** Payment Entry against the Sales Invoice; mismatch list report |
| Errors | `PostEx Log` doctype: request, response, status; retry button; never store full card/customer secrets |
| Security | Token in Password field; API calls server-side only; webhook (if any) checks signature/secret |

⚠️ Need PostEx merchant account + API docs ❓.
⚠️ **The 8–13 day estimate is an assumption.** We have not seen PostEx's API docs yet. Re-estimate after reading them.

**Scope checklist (must be covered):**
- [ ] Shipment booking
- [ ] AWB / tracking number
- [ ] Label printing
- [ ] Shipment cancellation
- [ ] Tracking sync + delivery status updates
- [ ] COD reconciliation
- [ ] Returned and refused parcels
- [ ] Retry on failure + **no duplicate bookings** (idempotency)

🏁 Sandbox: book, label, track, return, payout match all pass tests.

## 7.3 Discount limit per role 🔴 (1–2 days)

| Item | Detail |
|---|---|
| Doctype | `Role Discount Limit` (Role, Max Discount %) |
| Hook | `validate` on Quotation, Sales Order, Sales Invoice, POS Invoice |
| Logic | Highest limit among the user's roles. If any line or bill discount is above it → set `needs_approval = 1` with reason "Discount above limit" |
| Desk documents | Go into the approval flow (spec 7.17) |
| POS | Cannot wait for a workflow at the counter → block + ask a manager to enter their PIN/login on that bill (or manager does the sale). ❓ confirm with client |

## 7.4 Rider cash settlement 🔴 (4–6 days)

```
Delivery Trip (rider) ──► orders delivered + cash collected per stop
        ▼
Rider Cash Settlement ──► expected cash vs cash handed over ──► difference
        ▼
Payment Entries to "Cash – Rider <name>" → transfer to main cash
```

| Item | Detail |
|---|---|
| Accounts | One cash account per rider (or one "Riders Cash" with party = employee) |
| Doctype | `Rider Cash Settlement` (rider, date, trips, expected, received, difference, remarks) |
| Report | Cash collected by rider (SRS staff report) |

## 7.5 Pending approvals page 🟡 (2–3 days)

One page (Workspace + custom page or report) listing documents waiting for **the current user's** workflow action: Purchase Orders, large discounts, credit sales, expenses, payroll. Click → open document.

## 7.6 Barcode generator + labels 🔴 (2–4 days)

| Item | Detail |
|---|---|
| Label printing | Jinja Print Format sized to the label roll. Barcode drawn by a barcode library in our app (or Frappe's **Barcode** field type, which takes a barcode value and draws it as SVG). ⚠️ The Barcode field does **not** create numbers or check EAN-13 rules — our generator does that. (Print Designer is **not** v16-compatible) |
| Tests | Unique numbers · correct EAN-13 check digit · existing barcodes never changed · label size on real printer · real scanner reads it |
| Rule | If an item has no barcode → button "Generate barcode" (and bulk action) |
| Format | **EAN-13** with internal prefix (e.g. `2xxxxxxxxxxxC`, check digit computed) — ❓ confirm client does not have a GS1 prefix |
| Uniqueness | Check against all Item Barcode rows |

## 7.7 Delivery charge by city 🔴 (2–3 days)

| Item | Detail |
|---|---|
| Doctype | `City Delivery Charge` (city, charge, free above amount, active) |
| Hook | Webshop cart/checkout → read shipping address city → add charge as a tax/charge row |
| Note | Built-in Shipping Rule works only by **country**, so this is needed |

## 7.8 "Notify me when back in stock" 🔴 (2–3 days)

| Item | Detail |
|---|---|
| Doctype | `Back In Stock Request` (item, email, customer, status) |
| Website | Button on out-of-stock product page |
| Trigger | When stock of that item goes from 0 to > 0 (stock ledger hook / scheduled check) → email → status Notified |

## 7.9 COD risk flag 🔴 (1–2 days)

Report + customer field `cod_risk` (Low / Medium / High) based on count of returned/refused COD parcels in the last 6 months. Online order team sees a warning badge on new orders from high-risk customers.

## 7.10 Payment gateway 🔴 (5–8 days each) — `pk_integrations/<provider>`

```
Checkout ─► Payment Request ─► provider hosted page ─► customer pays
                                     │
            callback / webhook ◄─────┘ (verify hash/signature)
                  ▼
     Payment Request "Paid" ─► Payment Entry ─► Sales Order paid
```

| Rule | Detail |
|---|---|
| Existing gateways | The `payments` v16 code has Stripe, PayPal, Razorpay, Braintree, Paytm, GoCardless, M-Pesa, Paymob (its README lists only the first 5). **None is verified for Pakistan.** Paymob in the app uses its **Egypt** API — do not assume it works with a Pakistani merchant account |
| JazzCash | Has official merchant API docs (sandbox) → custom gateway is possible |
| Secrets | In a Settings doctype Password field, never in code |
| Callback | Verify signature/hash; idempotent (same callback twice = one payment) |
| Status job | Scheduler re-checks pending payments with the provider |
| Logs | Request/response log doctype (no full card data ever stored) |

## 7.11 Attendance machine 🟡 (1–2 days ZKTeco / 3–7 other)

```
Machine (LAN) ──► sync tool (on a shop PC/server) ──► HRMS API: add_log_based_on_employee_field
                                                      ▼
                                          Employee Checkin ──► Auto Attendance (Shift Type)
```
- Map machine user ID → Employee "Attendance Device ID".
- If machine is not ZKTeco-compatible → build a small sync script for its API ❓.

## 7.12 Payroll rules 🟡/🔴 (3–5 days)

| Rule | How |
|---|---|
| Late arrivals (e.g. 3 lates = 1 day cut) | Monthly job counts `late_entry` attendances → creates **Additional Salary** (deduction) before payroll |
| Absent / half day | Built-in (Shift Type thresholds, leave without pay) |
| Sales commission | Monthly job reads Sales Person commission → **Additional Salary** (earning) |
| Approval | Workflow on Payroll Entry (Owner approves) |

❓ Exact policy from the client.

## 7.13 Owner daily summary 🟡 (1–2 days)

Email Digest covers income, receivables, orders. Custom daily email (scheduler, 9 PM) adds: sales by channel, cash in hand, orders by stage, expenses, low-stock list. Use an **Email Template** with the brand style.

## 7.14 Product Price & Margin view — MANDATORY 🔴 (2–4 days)

SRS 3.2 requires **cost price and profit margin for each of the 3 prices**. No standard ERPNext screen shows this in one place.

Custom report "Product Price & Stock": item, SKU, unit, cost (valuation rate), Wholesale / Online / Self-delivery price, **margin % for each**, stock per location, website visible. Matches the SRS sample "Products" screen. Hidden from roles that must not see cost (Counter staff).

## 7.15 Stock adjustment reason 🟡 (0.5 day)

Custom field `adjustment_reason` (mandatory when Stock Entry Type = Damaged/Expired/Tester/Lost) + report "Adjustments by reason".

## 7.16 Sales Channel auto-fill 🟡 (1–2 days)

Creating the **Sales Channel** Accounting Dimension does **not** fill it on transactions. We must set it on every document, or P&L by channel will be wrong.

```
POS sale        → POS Profile has the dimension field → set "Shop"
Website order   → hook on Sales Order insert (from Webshop) → set "Online"
Wholesale order → company default dimension = "Wholesale"
        ▼
SO → DN → SI → Payment: value carries forward (verify in PoC)
```

| Item | Detail |
|---|---|
| Built-in pieces (verified in v16 code) | Dimension field is added to POS Profile, Sales Order, Delivery Note, Sales Invoice, POS Invoice, Payment Entry. Dimension has a **default per company** and a **"mandatory for P&L"** option |
| Custom | Hook that sets "Online" on website orders; validation that blocks a sales document with no channel |
| Old data | Not filled automatically. Only matters if we migrate old transactions |

## 7.17 Approval workflow (discounts + credit limit) 🔴 (3–5 days)

SRS 2.1, 3.6 and 3.14 need **real approvals**, not just "blocked" errors.

```
Staff saves order ──► system checks: discount > role limit?  credit limit crossed?
        │                         │ no                        │ yes
        │                         ▼                           ▼
        │                  normal submit            State = "Pending Approval"
        │                                                   │ notification to Manager
        │                                                   ▼
        │                                       Manager: Approve ──► submit allowed
        │                                                Reject  ──► back to staff with reason
        ▼
  Approval log: who, when, reason, amount (kept forever)
```

| Item | Detail |
|---|---|
| Tool | Frappe **Workflow** on Quotation / Sales Order / Sales Invoice with **transition conditions** (verified: Workflow Transition has a `condition` field), e.g. `doc.needs_approval == 1` |
| Flags | `needs_approval`, `approval_reason` set by our validate hook (7.3 discount, credit limit check) |
| Credit limit | Built-in check runs on submit and blocks non-controllers. Our hook runs **before** that, sends the order to Pending Approval; when the Manager (credit controller role) submits, the built-in check passes |
| Record | Workflow action history + our `Approval Log` child table (approver, time, reason) |
| One place | All pending approvals shown on the Approvals page (7.5) |
| Webshop | Website orders have no manual discount → condition false → normal flow. ⚠️ test in PoC that the workflow does not block Webshop's auto-submit |

---

# Part 8 — UI / UX design system

## 8.1 Design principles

1. **Simple first.** Show only what each role needs (workspaces + module profiles).
2. **Scanner first.** Every stock and sales screen works with a barcode scanner, without a mouse.
3. **Fast.** Fewer clicks for daily jobs (POS sale, GRN, packing).
4. **Consistent.** Same colours, words and button order everywhere.
5. **Mobile-friendly.** Website is mobile-first; desk works on tablet.
6. **Clear status.** Colour + text for every status (never colour only).

## 8.2 Brand colours

> Picked from the SRS sample screens. ⚠️ Designer must confirm final values.

| Token | Hex | Use |
|---|---|---|
| `--brand-primary` | `#7A2350` | Main buttons, active menu, links (plum) |
| `--brand-primary-hover` | `#621B40` | Hover / pressed |
| `--brand-accent` | `#D4849F` | Online channel, highlights (rose) |
| `--brand-sand` | `#C8B29C` | Shop/self-delivery channel, soft elements |
| `--bg-page` | `#F5F0EB` | Website page background (warm cream) |
| `--bg-card` | `#FFFFFF` | Cards, tables |
| `--text-main` | `#1F1A1C` | Main text |
| `--text-muted` | `#6B6266` | Secondary text |
| `--border` | `#E6DED7` | Lines, inputs |
| `--success` | `#2E7D4F` | Delivered, paid, in stock |
| `--warning` | `#B7791F` | Pending, low stock |
| `--danger` | `#B42318` | Returned, overdue, out of stock |
| `--info` | `#2B6CB0` | Booked, in transit |

**Channel colours (charts):** Wholesale = primary plum · Online = rose · Shop = sand (same as SRS sample).

**Order stage badges:**

| Stage | Colour |
|---|---|
| Pending | warning (amber) |
| Confirmed | info (blue) |
| Packed | purple `#6B46C1` |
| Booked / In transit | info (blue) |
| Delivered | success (green) |
| Returned / Cancelled | danger (red) |

**Contrast rule:** text must pass **WCAG AA** (4.5:1). White text on `#7A2350` passes.

**Dark mode:** website = light only (v1). Desk follows Frappe's own light/dark theme — do not override.

## 8.3 Typography

| Use | Font | Size |
|---|---|---|
| Website headings | Serif (e.g. *Lora* or *Playfair Display*, Google Fonts) ⚠️ designer to confirm | H1 40–48 px, H2 28–32 px |
| Website body | *Inter* | 16 px, line height 1.5 |
| Prices | *Inter* semibold | 18–22 px |
| Desk | Frappe default (do not change) | — |
| Receipts (80 mm) | Monospace / system | 11–12 px |

## 8.4 Desk (admin portal) branding — upgrade-safe

| Do ✅ | Don't 🚫 |
|---|---|
| App logo + name (Navbar / Website Settings) | Override Frappe core CSS classes |
| Brand colour in our own components only | Change Frappe layout or JS |
| Custom Workspaces, icons, number cards | Hide core fields with CSS (use Property Setter) |
| Login page logo and background | Edit files inside `apps/frappe` |

## 8.5 Workspaces per role

| Workspace | Role | Shortcuts / cards |
|---|---|---|
| 🏠 **Owner** | Owner | Dashboard cards, Approvals, Reports, Settings |
| 🧾 **Counter** | Counter Staff | POS, Customer, Sales Return, Today's sales |
| 📦 **Store** | Store | Purchase Receipt, Stock Transfer, Stock Count, Print Labels, Low Stock |
| 🚚 **Online Orders** | Order Team | Kanban, Pick List, Book PostEx, Print Labels, Returns |
| 💰 **Accounts** | Accountant | Payment Entry, Expenses, Bank Clearance, Ledgers, Ageing, P&L |
| 👥 **HR** | HR | Employees, Attendance, Leave, Payroll, Payslips |
| 🛒 **Purchasing** | Manager | Reorder list, RFQ, PO, Supplier ledger |

Each workspace: max **8 shortcuts** + **4 number cards** + 1 chart. Most-used first.

## 8.6 Owner dashboard (matches SRS sample)

```
┌──────────────┬──────────────┬──────────────┬──────────────┬──────────────┐
│ Today's sales│ Orders to    │ Cash in hand │ Receivable   │ Low stock    │
│ Rs 248,500   │ process: 37  │ Rs 86,200    │ Rs 1.42M     │ 18 items     │
└──────────────┴──────────────┴──────────────┴──────────────┴──────────────┘
┌───────────────────────────────┐ ┌──────────────────────────────────────┐
│ Sales by channel (stacked bar)│ │ Low stock alerts (table + reorder btn)│
└───────────────────────────────┘ └──────────────────────────────────────┘
┌──────────────────────────────────────────────────────────────────────────┐
│ Latest online orders (stage badges)                                        │
└──────────────────────────────────────────────────────────────────────────┘
```
Built with **Number Card** + **Dashboard Chart** + a custom HTML/report block. Data by **Sales Channel** dimension.

## 8.7 Print formats

| Format | Size | Must show |
|---|---|---|
| Sales Invoice (A4) | A4 | Logo, NTN/STRN ❓, customer, items, UOM, discount, tax, total in words, payment terms, bank details, signature |
| POS receipt | **80 mm** thermal | Logo (mono), bill no, cashier, items, total, paid, change, return policy line, QR (optional) |
| Delivery challan | A4 | No prices option, receiver signature |
| Quotation | A4 | Validity date, terms |
| Purchase Order | A4 | Supplier, delivery date, terms |
| Packing slip | A5 / 80 mm | Order no, items, barcode of order no |
| **Barcode label** | e.g. **38 × 25 mm** ⚠️ match client's label rolls | Product name (short), shade, price, barcode |
| Payslip | A4 | Earnings, deductions, net pay, attendance summary |
| Customer statement | A4 | Opening, invoices, payments, closing, ageing |

Rules: same letter head, same fonts, prices right-aligned, PKR format `Rs 1,250`. Test on the client's real printers ❓ (models). For direct thermal printing consider Frappe **raw printing (QZ Tray)** ⚠️ verify on v16.

## 8.8 Website UX

**Pages and main flow**
```
Home ─► Category ─► Product ─► Cart ─► Checkout ─► Order placed ─► Track order
  │                     │                  │
  └─► Search            └─► Wishlist        └─► Login / Sign-up (required)
```

| Page | Key UX points |
|---|---|
| Home | Hero banner (Builder), shop by category tiles, best sellers, "Become a dealer" banner, trust strip (COD, live stock, tracking) |
| Category | Filters (price, brand, skin type, shade), sort, product cards with stock badge |
| Product | Image gallery, **shade swatches**, live stock, price, Add to cart + Buy now, COD/delivery/returns boxes, how to use, ingredients, **Complete the look** (recommended items), reviews |
| Cart | Quantity edit, coupon, delivery charge by city, total |
| Checkout | Address (saved), payment (COD / online), clear total, one page |
| Account | Orders, tracking, addresses, wishlist |
| Order tracking | Stage timeline (same colours as 8.2), PostEx tracking link |
| Policies | About, Contact, Delivery, Returns (editable) |

**Website quality targets**

| Area | Target |
|---|---|
| Mobile | Designed for 360 px width first |
| Speed | Images WebP, lazy-loaded; LCP < 2.5 s on 4G |
| Accessibility | Alt text, labels on inputs, keyboard navigation, AA contrast |
| SEO | Unique title + meta description, clean URLs, sitemap, product structured data (optional) |
| Trust | Clear COD info, return policy link, WhatsApp button |

## 8.9 POS UX

- Search box always focused for the scanner.
- Large buttons (min 44 px), big totals.
- Price list switch (Shop / Wholesale / Online) — only for allowed roles.
- Keyboard: Enter = add, F-keys for payment (configure in training).
- Receipt prints automatically after payment.

## 8.10 Language and formats

| Item | Setting |
|---|---|
| Language | English (v1). Urdu labels possible later with Frappe translations ❓ |
| Date | `dd-mm-yyyy` |
| Number | `#,###.##` |
| Currency | PKR, symbol **Rs** |
| Time zone | Asia/Karachi |

## 8.11 Email look

One branded **Email Template** layout: logo header, plum button, footer with contact + policy links. Used for order stages, invoices, statements, payslips, daily summary.

---

# Part 9 — Configuration blueprint

## 9.1 Global settings

| Setting | Value |
|---|---|
| System Settings → time zone | Asia/Karachi |
| Date / number format | dd-mm-yyyy / #,###.## |
| Currency | PKR |
| Fiscal year | 1 Jul – 30 Jun ❓ |
| Password policy | ON, min score 3 |
| Session expiry | 8 hours (desk) |
| 2FA | Owner, Manager, Accountant, HR |
| Encrypt backups | ON |

## 9.2 Chart of Accounts (key ledgers)

| Group | Ledgers |
|---|---|
| Cash | Cash – Main, Cash – Shop Counter, Petty Cash, Cash – Riders |
| Bank | Bank – <names> ❓, **JazzCash Wallet**, **Easypaisa Wallet** |
| Receivables | Debtors, **PostEx COD Receivable** |
| Stock | Stock In Hand, Stock Received But Not Billed |
| Liabilities | Creditors, GST payable ❓, Salary Payable, Employee Advances |
| Income | Sales (by Sales Channel dimension), Delivery Charges Income |
| Expenses | COGS, Rent, Utilities, Fuel, Packaging, Marketing, Courier Charges, Salaries, Stock Adjustment (Damaged/Expired/Tester) |

## 9.3 Stock Settings

| Setting | Value |
|---|---|
| Enable stock reservation | ✅ ON |
| Auto reserve stock | ✅ ON |
| Allow negative stock | ❌ OFF |
| Pick batch based on | Expiry (FEFO) |
| Auto Material Request at reorder level | ✅ ON + email |
| Valuation method | Moving Average (default) ❓ confirm with accountant |

## 9.4 Selling / Buying settings

| Setting | Value |
|---|---|
| Validate selling price vs cost | ✅ ON |
| Sales Order required for Delivery (wholesale) | ✅ ON |
| Role allowed to bypass credit limit | Manager |
| PO required for Purchase Receipt | ✅ ON |
| Maintain same rate across purchase cycle | ✅ ON |

## 9.5 Workflows

| Doctype | States | Approver |
|---|---|---|
| Purchase Order | Draft → Pending Approval → Approved → (Submitted) | Owner / Manager |
| Sales Order (online) | Pending → Confirmed → Packed → Booked → Dispatched → Delivered / Returned / Cancelled | Order team / Manager |
| Sales Order (credit above limit) | handled by credit-limit bypass role | Manager |
| Expense (Journal Entry / Purchase Invoice) | Draft → Pending Approval → Approved | Owner / Manager |
| Payroll Entry | Draft → Pending Approval → Approved | Owner |
| Leave Application | Open → Approved / Rejected (built-in) | Leave approver |

## 9.6 Roles matrix

| SRS role | Frappe roles (Role Profile) | Can NOT |
|---|---|---|
| Owner / Admin | System Manager, Accounts Manager, Stock Manager, Sales Manager, Purchase Manager, HR Manager | — |
| Manager | Sales Manager, Purchase Manager, Stock Manager, Item Manager | Payroll (by default) |
| Counter staff | Sales User + POS access, branch User Permission | Change prices, see cost, other branches |
| Store | Stock User, Purchase Receipt create | Prices, accounts |
| Accountant | Accounts User/Manager | Stock settings, HR |
| HR / Payroll | HR User/Manager | Sales, accounts settings |
| Online order team | Sales User + custom "Online Order Team" role | Prices, accounts |
| Customer (website) | Customer (portal) | Desk access |

## 9.7 Naming series (example)

| Doc | Series |
|---|---|
| Sales Invoice | `SINV-.YY.-.#####` |
| POS Invoice | `POS-.YY.-.#####` |
| Sales Order (online) | `OL-.#####` (matches SRS "#OL-10482") |
| Sales Order (wholesale) | `SO-.YY.-.#####` |
| Purchase Order | `PO-.YY.-.#####` |
| Purchase Receipt (GRN) | `GRN-.YY.-.#####` |
| Delivery Note (challan) | `DC-.YY.-.#####` |
| Payment Entry | `PAY-.YY.-.#####` |

---

# Part 10 — Integrations

| Integration | Method | Owner | Status |
|---|---|---|---|
| PostEx | REST API, our app (7.2) | Backend | 🔴 ❓ credentials |
| Payment gateway | REST + callback (7.10) | Backend | 🔴 ❓ provider |
| Attendance machine | sync tool → Employee Checkin (7.11) | Backend/DevOps | 🟡 ❓ model |
| Email | SMTP (domain email), **SPF + DKIM + DMARC** set on DNS | DevOps | ✅ |
| Barcode scanners | USB/Bluetooth as keyboard (HID) — no code | Setup | ✅ ❓ models |
| Label + receipt printers | Browser print / raw printing | Setup | 🟡 ❓ models |
| Google Analytics / FB Pixel | Website Settings + head HTML | Frontend | ✅ |
| WhatsApp chat button | Link button `wa.me/<number>` | Frontend | ✅ |

**Integration rules**
1. All API keys in **Password** fields or `site_config.json` — never in Git.
2. All external calls run in **background jobs** with timeout + retry.
3. Every call is **logged** (request, response, status) for 90 days.
4. Webhooks verify a **signature/secret** and are **idempotent**.

---

# Part 11 — Security plan

## 11.1 Application

- [ ] Password policy ON (min score 3), lockout after failed attempts
- [ ] 2FA for Owner, Manager, Accountant, HR
- [ ] Session expiry set; logout all sessions on password reset
- [ ] Role Profiles + Module Profiles; least privilege review
- [ ] User Permissions by branch/warehouse for counter staff
- [ ] Login hours + IP restriction where possible
- [ ] Track Changes + Audit Trail reviewed monthly
- [ ] No "Administrator" use in daily work; named users only
- [ ] API keys only for integrations, one per integration, rotated yearly
- [ ] Guest permissions reviewed (website only sees published items)
- [ ] Every custom `@frappe.whitelist()` method checks permissions (`frappe.has_permission` / `doc.check_permission`)
- [ ] No raw SQL with user input (use query builder / parameters)

## 11.2 Server

- [ ] HTTPS only (Let's Encrypt), HSTS
- [ ] Firewall: only 80/443 (and SSH from office IP) open
- [ ] SSH keys only, no root/password login
- [ ] MariaDB and Redis **not** exposed to the internet
- [ ] OS security updates monthly
- [ ] fail2ban or provider equivalent

## 11.3 Backups

| What | How often | Where | Keep |
|---|---|---|---|
| Database + files | Daily. Bare-metal bench: `bench setup backups` adds the cron job. **Docker:** containers have no cron, so add a host cron / scheduled container running `bench --site <site> backup --with-files` | Server | 7 days |
| Offsite copy | Daily | S3 / Google Drive via `offsite_backups` | 30 days |
| Before every deploy | Each release | Server | Until next release |
| **Restore test** | Monthly | Staging | — |

⚠️ The Frappe app itself does **not** schedule backups. Bench gives the command (`bench setup backups`), but someone must set it up and **check it runs**. A local backup is **not** an offsite backup, and a backup is only real after a **restore test**.

---

# Part 12 — Performance and scalability plan

## 12.1 Rules

1. Heavy jobs (PostEx sync, payroll, big reports, imports) → **background workers** (long queue).
2. Heavy reports → **Prepared Reports** (like Stock Balance).
3. Monthly **Stock Closing Entry** (v16) to keep stock reports fast.
4. Add database **indexes** for custom fields used in filters (e.g. `order_stage`, `cod_risk`).
5. Website images: compress + WebP; CDN if traffic grows.
6. Never loop `frappe.get_doc` over thousands of rows — use `frappe.get_all` with fields.

## 12.2 Scaling steps

```
Start: 1 server (all-in-one)
  ▼ when CPU > 70% or slow pages
Add workers + more RAM
  ▼ when DB is the bottleneck
Separate DB server (MariaDB tuning, buffer pool)
  ▼ when traffic is high
Multiple web containers behind load balancer + CDN for website
```

## 12.3 Performance tests (before go-live)

| Test | Target |
|---|---|
| 5,000 SKUs, 50,000 stock ledger rows | Item list < 2 s |
| 30 POS users at once | Sale saved < 1 s |
| 500 online orders/day | PostEx sync job < 5 min |
| Website home/product | LCP < 2.5 s (4G) |

## 12.4 Monitoring

- Uptime monitor (e.g. external ping every 1 min)
- Disk, CPU, RAM alerts
- Frappe **Error Log** checked daily in hypercare, weekly after
- Scheduled job failures (RQ Job / Scheduled Job Log) alert

---

# Part 13 — Testing plan

## 13.1 Test levels

| Level | What | Who |
|---|---|---|
| Unit / integration tests | Custom Python code (Frappe test runner) | Developers |
| Functional tests | Each phase's "done when" | QA |
| Accounting tests | GL matches expected for each flow | QA + accountant |
| UAT | Real users with real scenarios | Client key users |
| Regression | Re-run main flows before each release | QA |
| Performance | Part 12.3 | DevOps |
| Security | Part 11 checklist + permission tests | QA + DevOps |

## 13.2 Must-pass business flows

1. Purchase: PO → approval → GRN (partial) → bill → payment
2. Wholesale credit sale: quotation → SO → DC → invoice → partial payment → statement
3. POS: scan → split payment → receipt → return → closing
4. Online COD: website order → reserve → pack → PostEx book → deliver → COD payout match
5. Online prepaid: website order → gateway payment → delivered
6. Return parcel: returned → Returns warehouse → check → sellable / damaged
7. Rider: trip → cash collected → settlement
8. Stock: transfer, damage adjustment, stock count with difference
9. Payroll: attendance sync → late rule → overtime → loan → salary → approval → payslip email
10. Month-end: P&L, Balance Sheet, stock value matches stock account

## 13.3 Test data

Use a **fake data script** (in `adeel_traders/tests/fixtures`) — 50 items with variants, 3 price lists, 20 customers, 5 suppliers, 10 employees. Never test with real customer data on local machines.

## 13.4 UAT script format

| Step | Action | Expected result | Pass/Fail | Notes |
|---|---|---|---|---|

One script per role (Counter, Store, Online team, Accountant, HR, Owner).

---

# Part 14 — Data migration plan

## 14.1 What we move

| Data | Source | Tool | Owner |
|---|---|---|---|
| Products, variants, barcodes, images | WordPress/WooCommerce + current stock system ❓ | Export CSV → Data Import | BA + Dev |
| Prices (3 lists) | Client Excel | Data Import (Item Price) | BA |
| Customers + addresses | WooCommerce + dealer list | Data Import | BA |
| Suppliers | Client Excel | Data Import | BA |
| Opening stock (qty + cost per warehouse) | Physical count on cut-over day | Stock Reconciliation (Opening) | Store + BA |
| Customer/supplier opening balances | Client ledgers | Opening Invoice Creation Tool | Accountant |
| Cash/bank opening balances | Bank statements | Journal Entry (Opening) | Accountant |
| Employees + salary data | HR records | Data Import | HR |
| Old orders (history) | WooCommerce | ⚠️ Optional — usually only keep a CSV archive | Decide ❓ |

## 14.2 Steps

```
1. Templates  →  2. Client fills  →  3. Clean (duplicates, missing SKU)
→ 4. Dry run on staging  →  5. Check totals  →  6. Fix  →  7. Final run on go-live
```

## 14.3 Checks after import

- [ ] Item count = source count
- [ ] Stock value = client's closing stock value
- [ ] Debtors / creditors total = client ledgers
- [ ] Trial Balance opening = client's balance sheet

## 14.4 Website switch (WordPress → new site)

- [ ] List all old URLs (products, categories) → **301 redirects** to new URLs (protects Google ranking)
- [ ] Lower DNS TTL 2 days before
- [ ] Switch DNS on go-live night; keep WordPress backup for 90 days

---

# Part 15 — Deployment, go-live and support

## 15.1 Deployment (Docker path)

```
apps.json (all apps + branches)
   ▼  GitHub Action: build custom image (frappe_docker)
Image tag: adeel-erp:<version>
   ▼  deploy to staging → bench migrate → smoke test
   ▼  approved → deploy same tag to production
       1. backup  2. pull image  3. migrate  4. smoke test  5. (rollback = previous tag + restore backup)
```

## 15.2 Cut-over plan (go-live weekend)

| Time | Task |
|---|---|
| Fri evening | Freeze old system (no new entries); final exports |
| Fri night | Physical stock count |
| Sat | Final migration + opening balances + checks (14.3) |
| Sat evening | Owner checks key numbers → **GO / NO-GO** |
| Sun | DNS switch, website live, smoke tests, test orders |
| Mon | Shop opens on new system; Nesway team on-site/online |

## 15.3 Go-live checklist

- [ ] All UAT scripts signed
- [ ] Security checklist ✅
- [ ] Backups + offsite + restore test ✅
- [ ] Print formats tested on real printers
- [ ] Scanners tested at every counter
- [ ] PostEx + payment gateway on **live** credentials
- [ ] Email deliverability tested (SPF/DKIM)
- [ ] Users created, passwords set, 2FA on
- [ ] Support contacts shared

## 15.4 Support after go-live

| Priority | Example | Response |
|---|---|---|
| P1 | POS down, website checkout broken | 4 hours |
| P2 | One report wrong, one print format issue | 1 working day |
| P3 | Small change request | Next planned release |

**Upgrades:** apply Frappe/ERPNext v16 patch releases monthly on staging first. Major version (v17) only after a planned upgrade project.

---

# Part 16 — Training and documentation

| Audience | Length | Topics |
|---|---|---|
| Owner | 2 h | Dashboard, approvals, reports, settings, users |
| Manager | 2 h | Purchasing, pricing, approvals, reports |
| Counter staff | 1 h + practice | POS, returns, closing |
| Store | 1.5 h | GRN, transfers, stock count, labels |
| Online order team | 1.5 h | Kanban, packing, PostEx, returns |
| Accountant | 3 h | Payments, expenses, bank, ledgers, month-end |
| HR | 2 h | Employees, attendance, leave, payroll |

Deliverables:
- [ ] One-page **quick guide** per role (PDF, with screenshots)
- [ ] Short screen videos (2–5 min) for top 10 tasks
- [ ] Admin guide for Nesway support (settings, jobs, integrations)

---

# Part 17 — Risks and how we handle them

| # | Risk | Impact | Plan |
|---|---|---|---|
| R1 | PostEx API docs/credentials late | Online orders delayed | Ask in Week 1; build with sandbox; manual booking fallback |
| R2 | Payment provider not chosen | Website without online pay | Launch with **COD first**; add gateway in release 1.1 |
| R3 | Attendance machine not ZKTeco-compatible | Extra custom work | Get model in Week 1; budget 3–7 days |
| R4 | Builder issue on v16 | Website delay | PoC in Week 1; fallback = Webshop templates + Web Pages. (Print Designer already dropped: not v16-compatible) |
| R5 | Bad master data (duplicates, no SKUs) | Wrong stock | Data cleaning step + 2 dry runs |
| R6 | Staff resist new system | Low adoption | Role workspaces, short trainings, champions in each team |
| R7 | Scope creep | Delay | Change requests in writing (SRS sign-off rule) |
| R8 | Core upgrade breaks custom code | Downtime | No core edits, tests in CI, staging first |
| R9 | Accounting mistakes by users | Wrong reports | Accountant training + month-end checklist |

---

# Part 18 — Open questions for the client

From SRS section 8 + our own:

| # | Question | Needed by |
|---|---|---|
| Q1 | How many branches/shops/warehouses? | Phase 1 |
| Q2 | Self-delivery price also for walk-in customers? | Phase 2 |
| Q3 | Dealers on credit? Credit limits? | Phase 3 |
| Q4 | Online payment methods besides COD? Which provider? | Phase 7 |
| Q5 | GST on invoices? NTN/STRN details? | Phase 1 |
| Q6 | Number of products (with shades/sizes)? | Phase 2 |
| Q7 | Barcode scanner, label printer, receipt printer models? Label size? | Phase 2 |
| Q8 | Number of users and roles? | Phase 1 |
| Q9 | Attendance machine model + API details | Phase 8 |
| Q10 | Leave policy, salary deduction policy | Phase 8 |
| Q11 | PostEx merchant account + API token | Phase 5 |
| Q12 | Fiscal year (Jul–Jun?) and valuation method | Phase 1 |
| Q13 | Hosting preference (own server vs Frappe Cloud) | Phase 0 |
| Q14 | Guest checkout needed? (not available by default) | Phase 6 |
| Q15 | Keep old WooCommerce order history? | Phase 10 |
| Q16 | Brand guidelines (logo files, exact colours, fonts)? | Phase 6 |
| Q17 | Approve the technology change (Frappe/ERPNext/MariaDB instead of Next.js/NestJS/PostgreSQL)? | Phase 0 |
| Q18 | Accountant approves the accounting design in 1.4-B? | Phase 1 |
| Q19 | Discount above limit at the POS counter: manager PIN, or manager does the sale? | Phase 4 |
| Q20 | Slow / dead stock: 60 / 180 days OK? | Phase 9 |
| Q21 | Expense: above what amount is a bill photo mandatory? | Phase 1 |
| Q22 | Product code (SKU) format, e.g. `VML-05`? It is used on every order and purchase. | Phase 2 |
| Q23 | Damaged stock: should a separate "damaged stock loss" account be used? (Moving stock to Damaged has no accounting effect today; only Expired/Lost write-offs post a loss.) | Phase 2 |
| Q24 | Do products need a Brand field? (No brands are set up yet.) | Phase 2 |
| Q25 | Confirm: only a Stock Manager may enter stock transactions dated in the past (set in Phase 2). | Phase 2 |
| Q26 | Confirm: valuation method FIFO (current setting). It is hard to change after real stock exists. Same as Q12. | Phase 2 |

---

# Part 19 — Appendix

## 19.1 Effort table (custom work)

| # | Work | Dev-days |
|---|---|---|
| 1 | PostEx integration | 8–13 |
| 2 | Payment gateway (each) | 5–8 |
| 3 | Website design + build + city charge + back-in-stock + tracking | 14–29 |
| 4 | Online order stages workflow + Kanban | 4–6 |
| 5 | Rider cash settlement | 4–6 |
| 6 | Channel dimension + **auto-fill** (7.16) + owner dashboard | 4–8 |
| 7 | Discount limit per role | 1–2 |
| 8 | Payroll rules | 3–5 |
| 9 | Attendance machine | 1–2 / 3–7 |
| 10 | Barcode generator + labels + print formats (Jinja, no Print Designer) | 5–8 |
| 11 | COD flag, adjustment reasons, approvals page, daily summary | 5–8 |
| 12 | Product Price & Margin view (**mandatory**) | 2–4 |
| 13 | Setup, roles, workflows, migration | 8–15 |
| 14 | Testing, deployment, security | 5–8 |
| 15 | Approval workflow for discounts + credit limit (7.17) | 3–5 |
| 16 | Custom reports from the matrix (sales by city, delivery success rate, expenses by branch) (19.4) | 3–5 |
| | **Total** | **≈ 75–137** |

> Arithmetic check: rows 1–14 sum to **69–127** (v1.1 wrongly said 72–129). v1.2 adds rows 15–16 (+6–10) → **75–137**.

> ⚠️ **These are planning estimates, not a commitment.** Items 1 (PostEx), 2 (payment gateway), 3 (website) and 13 (migration) need a detailed breakdown after the PoC and after we get the API docs. Do not promise the client a delivery date before that.

## 19.2 Naming rules for our code

| Thing | Rule | Example |
|---|---|---|
| Custom doctype | Title Case, clear noun | `Rider Cash Settlement` |
| Custom field | `snake_case`, our prefix if it could clash | `order_stage`, `at_cod_risk` |
| Module | `at_<area>` | `at_selling` |
| Hooks | One function per purpose in `<module>/events/<doctype>.py` | `at_selling/events/sales_order.py` |
| Tests | `test_<feature>.py` | `test_role_discount_limit.py` |

## 19.3 SRS requirement checklist (tick when done)

**3.1 Products**
- [ ] Item master (name, brand, category, images, description, barcode)
- [ ] Variants with own barcode/price/stock
- [ ] Website visibility, featured, new arrival, on sale
- [ ] Excel bulk import/export
- [ ] Fast search (name, barcode, SKU, brand, category)
- [ ] Activate/deactivate
- [ ] Bundles / gift sets

**3.2 UOM & pricing**
- [ ] Piece/Dozen/Box/Carton with conversion
- [ ] Buy/sell in any unit
- [ ] Wholesale / Online / Self-delivery prices
- [ ] Cost + margin + below-cost check
- [ ] Discount limit per role
- [ ] Dealer prices, website sale prices, coupons

**3.3 Inventory**
- [ ] Central real-time stock
- [ ] Per-warehouse stock + transfers
- [ ] Reservation for confirmed orders
- [ ] Adjustments with reason
- [ ] Barcode stock count + difference report
- [ ] Minimum stock alerts
- [ ] Full stock history
- [ ] Opening stock
- [ ] Stock value per branch + total
- [ ] Damaged/returned kept separate

**3.4 Barcode**
- [ ] Use existing / generate new
- [ ] Scan at counter, receiving, count, packing
- [ ] Print labels with name + price

**3.5 Suppliers & purchasing**
- [ ] Supplier records, terms, opening balance
- [ ] Reorder list with suggested qty
- [ ] RFQ + comparison
- [ ] PO → receiving (full/partial)
- [ ] Bills + returns
- [ ] Payments + supplier ledger
- [ ] PO approval
- [ ] Email/print PO
- [ ] Last purchase price
- [ ] Cost incl. freight

**3.6 Customers & dealers**
- [ ] Customer types → automatic price
- [ ] Details, addresses, history
- [ ] Credit limit + approval
- [ ] Ledger + statement
- [ ] Ageing 30/60/90+
- [ ] Opening balances
- [ ] Email reminders for overdue
- [ ] Top / inactive customers

**3.7 Sales**
- [ ] POS (scan, pay, receipt)
- [ ] Hold/recall, daily cash closing
- [ ] Wholesale flow
- [ ] Rider assignment + cash
- [ ] Returns/exchanges
- [ ] Branded invoice, challan, receipt
- [ ] Cash/card/transfer/JazzCash/Easypaisa/split
- [ ] Credit sales to ledger
- [ ] Salesperson on every sale
- [ ] PDF quotation/invoice, reprint
- [ ] Rider cash settlement

**3.8 Online orders**
- [ ] All orders in one list
- [ ] Order stages
- [ ] PostEx booking + labels
- [ ] PostEx tracking auto-update
- [ ] PostEx COD matching
- [ ] Returned parcels back to stock
- [ ] Packing by scan
- [ ] Pick lists + packing slips
- [ ] Cancel with reason, release stock
- [ ] Flag COD refusers

**3.9 Accounts**
- [ ] Cash/bank/wallet accounts
- [ ] Receipts + payments
- [ ] Expenses by category + photo
- [ ] Transfers + daily cash book
- [ ] Ledgers
- [ ] Opening balances
- [ ] Cheques + clearing
- [ ] Petty cash
- [ ] Expense approval
- [ ] Monthly P&L

**3.10 HR & payroll**
- [ ] Employee records
- [ ] Salary setup
- [ ] Shifts, off days, holidays
- [ ] Attendance from machine
- [ ] Leave types, balances, approval
- [ ] Late/absent/half-day rules
- [ ] Advances and loans
- [ ] Salary sheet, payslips, payments
- [ ] Overtime
- [ ] Bonus + commission
- [ ] Employee documents
- [ ] Salary approval

**3.11 Website**
- [ ] Modern, fast, mobile, no double entry
- [ ] Live products/prices/stock
- [ ] Category, brand, search, filters
- [ ] Cart, checkout (COD + online), city delivery charges, coupons
- [ ] Account, order history, tracking
- [ ] Editable banners/sections
- [ ] SEO
- [ ] Sign-up, login, reset, addresses
- [ ] Wishlist, reviews (approved), back-in-stock
- [ ] Editable policy pages
- [ ] WhatsApp, GA, FB Pixel

**3.12 Communication**
- [ ] Order stage emails
- [ ] Email invoices/statements/payslips
- [ ] Low stock + approval alerts
- [ ] Owner daily summary
- [ ] Branded email templates

**3.13 Settings & security**
- [ ] Company, logo, invoice layout
- [ ] Users, roles, permissions
- [ ] Minimum stock settings
- [ ] Masters (units, categories, brands, expense types, leave types)
- [ ] PostEx, attendance, email settings
- [ ] Change history
- [ ] Branches/warehouses
- [ ] Numbering, tax, payment methods, city delivery charges
- [ ] Strong passwords, auto logout, login history

**3.14 Dashboard**
- [ ] Owner dashboard
- [ ] Pending approvals
- [ ] Monthly trend vs last month
- [ ] Role dashboards

**4 Reports** (all filter by date/branch/channel/category/brand + Excel/PDF)
- [ ] Stock reports
- [ ] Sales reports (incl. by channel, profit by product/invoice)
- [ ] Online order reports
- [ ] Purchase reports
- [ ] Expense reports
- [ ] Accounts reports
- [ ] HR reports
- [ ] Customer reports
- [ ] Staff reports
- [ ] Owner's daily summary

**6 General**
- [ ] Any browser, tablet, mobile
- [ ] Role-based secure login
- [ ] Daily automatic backups
- [ ] Fast with large data
- [ ] Cancelled records kept
- [ ] A4 + 80 mm printing
- [ ] Export any list to Excel
- [ ] Training + user guides
- [ ] Post-go-live support

## 19.4 Report-by-report acceptance matrix

Every report from SRS section 4. All report names below were **checked to exist** in ERPNext v16.53 / HRMS v16 code. "Custom" = we build it.

**Common rule for every report (SRS 4):** filter by date, branch, channel, category, brand **where the data has that field**, and export to Excel/PDF. ⚠️ Not every built-in report has every filter — the "Filters to add" column lists the gaps. Each row is tested in UAT.

| # | SRS report | Source | Type | Filters to add / notes | Test |
|---|---|---|---|---|---|
| **Stock** |||||
| 1 | Current stock | Stock Balance (prepared report) | ✅ | Warehouse = branch | [ ] |
| 2 | Stock value | Stock Balance · Warehouse wise Item Balance Age and Value · "Warehouse wise Stock Value" dashboard chart | ✅ | — | [ ] |
| 3 | Low stock | Stock Projected Qty · Itemwise Recommended Reorder Level | ✅ | — | [ ] |
| 4 | Stock movement | Stock Ledger | ✅ | — | [ ] |
| 5 | Slow-moving and dead stock | Stock Ageing · Inactive Sales Items | ✅ | Define: slow = no sale in **60 days**, dead = **180 days** ❓ confirm | [ ] |
| **Sales** |||||
| 6 | Daily / monthly / yearly sales | Sales Analytics (range Daily…Yearly) | ✅ | — | [ ] |
| 7 | Sales by channel | Sales Register / Gross Profit + **Sales Channel** dimension | 🟡 | Needs 7.16 auto-fill | [ ] |
| 8 | By product, category | Item-wise Sales Register · Sales Analytics (Item / Item Group) | ✅ | — | [ ] |
| 9 | By brand | Gross Profit (group by **Brand**) | ✅ | — | [ ] |
| 10 | By customer | Sales Analytics (Customer) · Customer Ledger Summary | ✅ | — | [ ] |
| 11 | By salesperson | Sales Person-wise Transaction Summary | ✅ | — | [ ] |
| 12 | Profit by product / by invoice | Gross Profit (group by Item Code / Invoice) | ✅ | — | [ ] |
| **Online orders** |||||
| 13 | Orders by stage | Sales Order list/Kanban grouped by `order_stage` + small report | 🟡 | — | [ ] |
| 14 | Delivery success and return rate | 🔴 custom (from Shipment/PostEx status) | 🔴 | Date, city, courier | [ ] |
| 15 | PostEx COD received vs pending | 🔴 custom (7.2) | 🔴 | Date | [ ] |
| **Purchase** |||||
| 16 | By supplier and product | Purchase Analytics · Item-wise Purchase Register | ✅ | — | [ ] |
| 17 | Pending purchase orders | Purchase Order Analysis | ✅ | — | [ ] |
| 18 | Supplier balances | Accounts Payable · Supplier Ledger Summary | ✅ | — | [ ] |
| **Expenses** |||||
| 19 | By category, branch, month | Profit and Loss (periodicity Monthly, filter Cost Center = branch) · General Ledger | 🟡 | A simple "Expenses by category × month" view is friendlier → small custom report | [ ] |
| **Accounts** |||||
| 20 | Customer and supplier ledgers | General Ledger (party) · Customer / Supplier Ledger Summary | ✅ | — | [ ] |
| 21 | Pending payments by age | Accounts Receivable / Payable (ageing 30/60/90) | ✅ | — | [ ] |
| 22 | Cash and bank book | General Ledger (account = cash/bank) | ✅ | — | [ ] |
| 23 | Daily cash closing | POS Closing Entry · POS Register | ✅ | — | [ ] |
| 24 | Cheques and clearing | Bank Clearance Summary · Cheques and Deposits Incorrectly cleared | ✅ | — | [ ] |
| 25 | Simple P&L | Profit and Loss Statement | ✅ | — | [ ] |
| **HR** |||||
| 26 | Attendance | Monthly Attendance Sheet | ✅ | — | [ ] |
| 27 | Late and absent summary | Shift Attendance (late entry / early exit) · Monthly Attendance Sheet | ✅ | — | [ ] |
| 28 | Leave balances | Employee Leave Balance / Summary | ✅ | — | [ ] |
| 29 | Salary sheet | Salary Register | ✅ | — | [ ] |
| 30 | Advances outstanding | Employee Advance Summary (+ loan schedule from Lending) | ✅ | — | [ ] |
| **Customers** |||||
| 31 | Top customers and dealers | Sales Analytics (Customer, sorted) · Customer Acquisition and Loyalty | ✅ | — | [ ] |
| 32 | Customers who stopped buying | Inactive Customers | ✅ | — | [ ] |
| 33 | Sales by city | 🔴 custom (city is on the **Address**, not on Territory) — or set Territory = city at setup | 🟡/🔴 | Decide in Phase 1 | [ ] |
| **Staff** |||||
| 34 | Sales by salesperson | Sales Person-wise Transaction Summary / Commission Summary | ✅ | — | [ ] |
| 35 | Cash collected by rider | 🔴 custom (7.4) | 🔴 | Date, rider | [ ] |
| **Owner** |||||
| 36 | Owner's daily summary | Email Digest + custom daily email (7.13) | 🟡 | — | [ ] |

## 19.5 Acceptance criteria for requirements that need more detail

| Requirement | Accepted when |
|---|---|
| Expense with bill photo | Expense cannot be submitted without an attachment (if amount > ❓ limit); goes to approval (Workflow); photo visible from the ledger line |
| Cheques received | Payment Entry mode = Cheque, cheque no + date mandatory → status "Uncleared" until Bank Clearance sets clearance date → Bank Clearance Summary shows it |
| Cheques issued | Same, via Pay entry + Cheque Print Template |
| Monthly sales/profit vs last month | Dashboard chart: this month vs last month sales and gross profit (from Gross Profit data); numbers match the reports |
| Packing by barcode | Scanning an item **not in the pick list** shows an error (built-in barcode scanner shows "Cannot find…" messages — verify in PoC that a wrong item is blocked, not added); qty over the order is blocked |
| Cancelled records kept | Test: cancel an invoice → it stays with status Cancelled; delete is blocked for submitted docs; audit shows who cancelled |
| Export to Excel/PDF | Each of the 36 reports in 19.4 exported once in UAT |
| Slow / dead stock | Uses the 60 / 180 day rule (❓ confirm) |
| Branch permissions | Counter user of Branch A cannot open Branch B's invoices, stock, POS profile or reports (test every doctype in the role's workspace) |
| Phone/social orders | Order created in desk appears in the same Online Orders list and Kanban as website orders |
| Stock reservation | Website order reserves at placement; desk order reserves at submit; cancel releases; last unit cannot be sold twice |

---

*End of plan. Update the version number at the top every time this plan changes.*
