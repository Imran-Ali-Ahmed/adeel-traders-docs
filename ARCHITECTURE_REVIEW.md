# Adeel Traders — Hybrid Architecture Review (for approval)

> **CONFIDENTIAL — Nesway and Adeel Traders only.**
> Status: **DRAFT for review.** Nothing in `IMPLEMENTATION_PLAN.md` changes until this is approved.
> Date: 10 Nov 2026 · Platform checked: Frappe 16.55 / ERPNext 16.53 (local install) + Webshop/HRMS `version-16` source.

---

## 0. How to read this

| Icon | Meaning |
|---|---|
| ✅ | Verified in the installed v16 source or the app's v16 branch |
| ⚠️ | Must be proven in the architecture PoC before we commit |
| 🟦 | Frappe Desk (standard ERPNext screens) |
| 🟩 | Next.js (our custom screens) |
| 🔴 | Custom backend work in our Frappe app |

**Rule used for every screen:** build it in Next.js **only** if it gives clear business value
(used all day by many people, needs speed/scanning/bulk actions, customer-facing, or the owner's main view).
Otherwise keep the standard ERPNext screen.

---

## 1. Verified facts this design depends on

| # | Fact | Where checked | Why it matters |
|---|---|---|---|
| F1 | Frappe has REST **API v1 and v2** (`frappe/api/v2.py`: get_list, create_doc, run_doc_method…) | ✅ installed source | Next.js can use standard endpoints; permissions apply |
| F2 | Session-auth writes need header **`X-Frappe-CSRF-Token`** (`auth.py validate_csrf_token`) | ✅ installed source | Next.js must send the CSRF token |
| F3 | The CSRF helper `frappe.sessions.get_csrf_token()` exists but is **not** a public API | ✅ installed source | We need one small safe endpoint to give the token to our frontend |
| F4 | Session cookie is **`SameSite=Lax`** (`auth.py`) | ✅ installed source | Frontend and backend should be on the **same origin** (or cookies get complicated) |
| F5 | **CORS** is controlled by `allow_cors` in site config (`app.py`) | ✅ installed source | Only needed if we do NOT use same origin |
| F6 | **OAuth2 provider with PKCE** exists (`integrations/oauth2.py`, OAuth Client doctypes) | ✅ installed source | Alternative auth option (ADR-005) |
| F7 | **Realtime (socket.io)** accepts the session cookie **or** an Authorization header | ✅ `realtime/middlewares/authenticate.js` | Live order updates in Next.js are possible |
| F8 | `@rate_limit` decorator exists (`rate_limiter.py`) | ✅ installed source | Protect public/storefront endpoints |
| F9 | **Webhook** doctype exists | ✅ installed source | Tell the storefront to refresh its cache when prices/products change |
| F10 | `frappe.get_all` = `get_list` **with `ignore_permissions=True`** | ✅ installed source | Our APIs must use `get_list` / permission checks, never `get_all` for user data |
| F11 | Webshop cart is a set of **whitelisted APIs**: `get_cart_quotation`, `update_cart`, `update_cart_address`, `apply_shipping_rule`, `apply_coupon_code` (guest), `place_order`; product listing `get_product_filter_data` (guest); wishlist; reviews | ✅ webshop `version-16` source | **The storefront can reuse Webshop's backend logic** — no duplicated pricing/cart rules |
| F12 | Webshop `place_order` **submits** the Sales Order | ✅ webshop source | Stock reservation happens at order placement |
| F13 | Frappe's own modern apps (HRMS PWA, Builder) use **Vue + frappe-ui**, not React | ✅ their `package.json` | Next.js is **not** Frappe's own recommended frontend pattern → we take extra integration work (see ADR-002) |

---

## 2. Validated hybrid architecture

```
                    Browser (staff)                 Browser (customers)
                          │                                 │
              app.adeeltraders.com              www.adeeltraders.com
                          │                                 │
┌─────────────────────────▼─────────────────────────────────▼──────────────┐
│                     Nginx reverse proxy (HTTPS)                           │
│  /api/*  /socket.io/*  /files/*  /private/files/*  /assets/*  /app/* (Desk)│
│        └────────────► Frappe                                              │
│  everything else ────► Next.js (staff app / storefront)                   │
└──────────┬──────────────────────────────────────────────┬─────────────────┘
           │                                              │
┌──────────▼───────────┐                      ┌───────────▼──────────────┐
│ Next.js (2 apps)      │  server-side calls   │ Frappe v16 (authority)   │
│ • staff app 🟩         │ ───────────────────► │ ERPNext · HRMS · Lending │
│ • storefront 🟩        │   same-origin API    │ Payments · Webshop(API)  │
│ shared: ui, api-client│                      │ adeel_traders 🔴          │
└───────────────────────┘                      │ pk_integrations 🔴        │
                                               └─────┬───────────┬────────┘
                                                     │           │
                                          MariaDB + Redis   Workers + Scheduler
                                                                 │
                                                  PostEx · Payment gateway · SMTP
```

**Why same origin:** the session cookie (`SameSite=Lax`, F4) and CSRF header (F2) work with no CORS setup, no tokens in the browser, and realtime works with the cookie (F7).
⚠️ Prove in PoC: login, CSRF, realtime and file download all work through the proxy.

### 2.1 Who uses which UI (one main UI per role)

| Role | Main UI | Why |
|---|---|---|
| Owner | 🟩 Next.js dashboard + approvals · 🟦 Desk for deep reports | Owner's daily view needs a clear, designed screen |
| Manager | 🟩 approvals, products & margin · 🟦 purchasing, pricing rules | Mixed |
| Online order team | 🟩 **Next.js only** (orders board, packing, PostEx) | Highest volume, bulk actions, scanning |
| Counter staff | 🟦 **ERPNext POS** (full screen, branded) | See ADR-006 |
| Store | 🟦 Desk (GRN, transfers, stock count with Scan Mode) | Standard screens already support scanning |
| Accountant | 🟦 Desk | Standard accounting screens are complete; rebuilding = risk |
| HR | 🟦 Desk + HRMS mobile PWA for employees | HRMS already has both |
| Customers | 🟩 Next.js storefront | Customer-facing, SEO, brand |

---

## 3. Standard ERPNext vs custom: feature matrix

Columns: **Backend** (where the logic lives) · **UI** (where people use it) · **Custom work**.

| SRS area | Feature | Backend | UI | Custom work |
|---|---|---|---|---|
| 3.1 | Items, variants, brands, categories, bundles, Excel import | ✅ ERPNext | 🟦 | — |
| 3.1 | Website visibility, featured, new/sale tags | ✅ Webshop Website Item + custom fields | 🟦 edit · 🟩 shown on storefront | small |
| 3.2 | UOM conversion, 3 price lists, price by customer group | ✅ ERPNext | 🟦 | — |
| 3.2 | **Cost + margin for each price** | 🔴 read API (valuation + 3 prices + stock) | 🟩 **Products & margin grid** | yes |
| 3.2 | Discount limit per role → approval | 🔴 validate hook + ✅ Workflow | 🟦 forms · 🟩 approvals inbox | yes |
| 3.3 | Stock, warehouses, transfers, reservation, count, reorder, history, value | ✅ ERPNext | 🟦 | — |
| 3.3 | Adjustment reason | ✅ Stock Entry Type + custom field | 🟦 | small |
| 3.4 | Scan at counter/receiving/count | ✅ | 🟦 POS / Desk | — |
| 3.4 | Barcode generator, labels | 🔴 | 🟦 button + print format | yes |
| 3.5 | Purchasing: RFQ, PO approval, GRN, bills, returns, landed cost | ✅ ERPNext + Workflow | 🟦 | — |
| 3.6 | Credit limits, ledgers, statements, ageing, dunning | ✅ ERPNext | 🟦 | — |
| 3.6 | Credit-limit approval | 🔴 hook + ✅ Workflow | 🟩 approvals inbox | yes |
| 3.7 | POS, split payment, hold (draft), cash closing | ✅ ERPNext POS | 🟦 POS (branded) | print format |
| 3.7 | Wholesale flow, returns, salesperson | ✅ ERPNext | 🟦 | — |
| 3.7 | Rider assignment (Driver + Delivery Trip) | ✅ ERPNext | 🟦 | — |
| 3.7 | Rider cash settlement | 🔴 new doctype | 🟦 (low volume → Desk is enough) | yes |
| 3.8 | Order list, stages, phone/social orders | 🔴 fields + transition API | 🟩 **Online orders board** | yes |
| 3.8 | PostEx book/label/track/COD | 🔴 `pk_integrations` | 🟩 actions on orders board | yes |
| 3.8 | Packing by scan | ✅ Pick List scan · ⚠️ or 🔴 packing API | 🟩 packing screen | yes |
| 3.8 | Returns back to stock, cancel releases stock | ✅ ERPNext | 🟩 actions | — |
| 3.9 | Accounts, receipts, payments, cheques, petty cash, P&L | ✅ ERPNext | 🟦 | — |
| 3.10 | Employees, shifts, leave, salary, overtime, advances, loans | ✅ HRMS + Lending | 🟦 + HRMS PWA | — |
| 3.10 | Attendance machine, late rule, commission → salary | 🔴 / ✅ sync tool | 🟦 | yes |
| 3.11 | Catalogue, filters, cart, coupon, wishlist, reviews, account | ✅ **Webshop APIs** (F11) | 🟩 **storefront** | UI only |
| 3.11 | City delivery charge, back-in-stock, order tracking | 🔴 | 🟩 | yes |
| 3.11 | Online payment | 🔴 gateway on `payments` app | 🟩 checkout | yes |
| 3.12 | Emails, alerts | ✅ Notification, Email Digest | — | small |
| 3.13 | Roles, permissions, 2FA, audit, login history | ✅ Frappe | 🟦 | — |
| 3.14 | Owner dashboard, trend vs last month | 🔴 aggregated read API | 🟩 **Owner dashboard** | yes |
| 3.14 | Pending approvals in one place | ✅ Workflow + 🔴 list API | 🟩 **Approvals inbox** | yes |
| 3.14 | Store / accounts / HR dashboards | ✅ Workspaces + Number Cards | 🟦 | config |
| 4 | 32 of 36 reports | ✅ ERPNext/HRMS reports | 🟦 (export Excel/PDF) | — |
| 4 | 4 custom reports | 🔴 Script Reports | 🟦 | yes |

### Next.js scope for version 1 (where it adds real value)

| 🟩 Screen | Users | Value |
|---|---|---|
| Storefront (home, category, product, cart, checkout, account, tracking) | Customers | Brand, SEO, conversion |
| Online orders board + packing + PostEx actions | Online team | Volume, bulk actions, scan-to-pack, live updates |
| Owner dashboard | Owner | One clear daily view |
| Approvals inbox | Owner, Manager | One place for all approvals (SRS 3.14) |
| Products & margin grid | Owner, Manager | 3 prices + cost + margin + stock in one grid (SRS 3.2) |

**Phase 2 candidates (only if v1 shows a need):** custom POS, customer 360 page, mobile store scanning screens.

---

## 4. Frontend / backend responsibility

| Concern | Frappe (backend) — **authority** | Next.js (presentation) |
|---|---|---|
| Pricing, discounts, taxes, coupons | ✅ all calculation | shows the result only |
| Stock, reservation, valuation | ✅ | shows only |
| Accounting entries | ✅ | never |
| Approvals | ✅ Workflow state + rules | shows inbox, calls "apply action" |
| Order stage rules (who/which next stage) | ✅ server transition table | shows allowed buttons returned by API |
| Validation | ✅ **always** (source of truth) | only for nicer forms (never the only check) |
| Permissions | ✅ roles, user permissions, `has_permission` | hides buttons the API says are not allowed |
| Integrations (PostEx, payment) | ✅ background jobs, retries, logs | triggers actions, shows status |
| Files, PDFs, print | ✅ Frappe print/PDF endpoints | links/downloads |
| Design system, layout, UX | — | ✅ shadcn + our brand tokens |
| Caching public catalogue | sends Webhook on change | ✅ ISR/revalidate on Webhook |

**Golden rule:** if a value can affect money, stock or approvals, it is calculated in Frappe. Next.js never re-implements it.

---

## 5. Security and API architecture

### 5.1 Authentication

| User type | Method |
|---|---|
| Staff | Frappe login (`/api/method/login`) through the same origin → session cookie |
| Customers | Frappe website users (sign-up / login / reset via Frappe) → session cookie on the storefront origin |
| Next.js server (public catalogue) | Calls **guest** endpoints only (F11). No secret keys needed |
| Integrations (PostEx webhooks, payment callbacks) | Our endpoints with signature/secret check — not user sessions |

🚫 **Never** put API key/secret in the browser. 🚫 Never use an "admin" key in Next.js for user actions.

### 5.2 Request flow

```
Browser ──(cookie sid)──► Nginx ──► Frappe /api/...            (reads)
Browser ──(cookie + X-Frappe-CSRF-Token)──► Frappe              (writes)
Next.js server ──(forwards user cookie)──► Frappe               (server-rendered staff pages)
Next.js server ──(no auth, guest)──► Frappe guest APIs          (public catalogue, cached)
```

### 5.3 Our API layer (`adeel_traders/api/v1/…`)

| Rule | Detail |
|---|---|
| Use standard REST first | Simple reads/writes on a doctype → `/api/v2/document/...` (permissions apply automatically) |
| Custom endpoints only for screen needs | e.g. `orders.board()`, `orders.move_stage()`, `products.margin_grid()`, `dashboard.owner()`, `approvals.inbox()`, `session.csrf()` |
| Versioned | `/api/method/adeel_traders.api.v1.<module>.<method>` — breaking change = new version |
| Permission check in every method | `frappe.has_permission(...)` / `doc.check_permission(...)` / `frappe.get_list` (never `get_all` for user data, F10) |
| Input validation | Type hints + explicit checks; reject unknown fields |
| No raw SQL with user input | Query Builder / parameters only |
| Rate limits | `@rate_limit` on guest/public endpoints (F8) |
| Errors | One error shape: `{code, message, field?}`; no stack traces to the browser |
| Contract docs | Each endpoint documented (input, output, permissions) + typed client in `packages/api-client` (zod schemas) |
| Tests | Python tests for every endpoint: allowed role, denied role, bad input |

### 5.4 Other security points

- Same-origin + HTTPS + HSTS; strict Content-Security-Policy on Next.js.
- Desk (`/app`) reachable only for staff roles; customers are website users (no desk access).
- Integration secrets in Password fields / `site_config.json`; webhooks idempotent and signed.
- Audit: Frappe Version / Audit Trail + our approval and stage logs.
- Dependency scanning on both Python and Node (CI).

---

## 6. Architecture Decision Records (summary)

Each ADR: decision · options · why · trade-off · status.

### ADR-001 Backend and database
- **Decision:** Frappe v16 + ERPNext v16 + MariaDB.
- **Options:** NestJS + PostgreSQL (SRS) · Frappe/ERPNext.
- **Why:** stock, accounting, POS, purchasing, HR, payroll already exist and are proven.
- **Trade-off:** MariaDB, not PostgreSQL; GPL v3; we follow Frappe's way of building.
- **Status:** ⏳ needs client written approval.

### ADR-002 Frontend technology for custom screens ⚠️ (challenge)
- **Options:**
  1. **Next.js + shadcn** (your preference, matches SRS).
  2. **Vue + frappe-ui** served by Frappe — this is what Frappe's own HRMS PWA and Builder use (F13).
  3. Desk only (Workspaces, custom pages).
- **Honest comparison:**

| | Next.js + shadcn | Vue + frappe-ui |
|---|---|---|
| Auth/CSRF/realtime with Frappe | We wire it (proxy, CSRF endpoint, socket client) | Built in |
| Deployment | Extra Node service | Served by Frappe (one service) |
| Storefront SEO/SSR | ✅ strong | weaker (SPA) |
| Component quality / design freedom | ✅ shadcn, huge React ecosystem | good, smaller ecosystem |
| Team skill | ❓ (React?) | ❓ (Vue?) |
| Matches SRS | ✅ | ❌ |

- **Recommendation:** **Next.js is acceptable** *only with* same-origin deployment, server-side authority (section 4), and the PoC passing auth/CSRF/realtime tests. If the team is stronger in Vue than React, frappe-ui is the lower-risk choice for the **staff app** (the storefront should stay Next.js either way, for SEO).
- **Status:** ⏳ your decision (team skill is the deciding factor).

### ADR-003 Hybrid scope
- **Decision:** Next.js only for the 5 v1 screens in section 3; everything else Desk.
- **Why:** rebuilding standard screens duplicates logic and adds risk with no business value.
- **Trade-off:** some staff use 2 UIs → mitigated by "one main UI per role" (2.1) and consistent branding.

### ADR-004 Storefront = Next.js on top of Webshop APIs (headless)
- **Options:** Webshop's own Jinja pages · Next.js + **Webshop APIs** · Next.js + fully custom cart.
- **Decision:** Next.js + Webshop APIs (F11).
- **Why:** reuses tested cart, pricing rules, coupons, shipping rules, Quotation → Sales Order flow. No duplicated business logic.
- **Trade-off:** Webshop has **no guest checkout** (login required); guest cart can live in the browser until login (UI state only, no business rules). ⚠️ PoC: call the cart APIs from a non-Jinja frontend end-to-end.

### ADR-005 Authentication
- **Options:** same-origin session cookie + CSRF · OAuth2 PKCE bearer tokens (F6) · API keys.
- **Decision:** same-origin session + CSRF.
- **Why:** least moving parts; Frappe's standard; realtime works with cookie (F7).
- **Trade-off:** requires one domain per app with Nginx path routing. OAuth2 stays as a fallback if we ever need a separate domain or a native mobile app.

### ADR-006 POS stays standard ERPNext POS in v1 ⚠️ (challenge)
- **Why:** POS has pricing rules, batches, loyalty, payments, offline handling, opening/closing and consolidation. Rebuilding it in Next.js means **copying business logic** — the exact technical debt we want to avoid.
- **Trade-off:** POS look ≠ SRS sample screen. We brand it and test it with cashiers; custom POS becomes a Phase 2 decision based on real feedback.

### ADR-007 Online order stages
- **Decision:** custom field `order_stage` (Allow on Submit) + server-side transition API; not Frappe Workflow (order is already submitted, F12).

### ADR-008 Approvals
- **Decision:** Frappe Workflow with transition conditions (discount, credit limit, PO, expense, payroll); Next.js inbox calls the standard "apply workflow action".

### ADR-009 Integrations
- **Decision:** separate app `pk_integrations`; all calls in background jobs with retry, idempotency key and request log.

### ADR-010 Sales channel
- **Decision:** Accounting Dimension "Sales Channel", auto-filled (POS Profile / website hook / company default).

### ADR-011 Deployment
- **Decision:** Docker: Frappe custom image (pinned apps) + Next.js images (staff, storefront) behind one Nginx; CI builds all three; staging before production.

---

## 7. Realistic estimates (planning ranges, dev-days)

### 7.1 What changes from plan v1.2 (75–137)

| Change | Dev-days |
|---|---|
| Remove: Webshop theme/template work (old row 3) | −14 to −29 |
| Remove: margin **report** (old row 12, now a Next.js screen) | −2 to −4 |
| Remove: approvals **page** in Desk (part of old row 11) | −2 to −3 |
| **Subtotal kept from v1.2** | **57–101** |

### 7.2 New work

| # | Work | Dev-days |
|---|---|---|
| N1 | Design phase: brand tokens (frontend-design), Figma for 5 screens + storefront, usability check | 8–12 |
| N2 | Next.js foundation: monorepo, shadcn design system, auth/CSRF, API client, proxy, CI, error/empty states | 10–15 |
| N3 | Storefront (headless Webshop) incl. city charge, back-in-stock, tracking, account, SEO | 20–35 |
| N4 | Online orders board + packing + PostEx actions + realtime | 10–15 |
| N5 | Owner dashboard | 6–10 |
| N6 | Approvals inbox | 4–6 |
| N7 | Products & margin grid | 5–8 |
| N8 | API layer: contracts, permission tests for N3–N7 | 5–8 |
| N9 | Extra end-to-end testing (Playwright) for Next.js flows | 5–8 |
| | **New work subtotal** | **73–117** |

### 7.3 Total

```
Kept backend + config work      57 – 101
New frontend + API work          73 – 117
────────────────────────────────────────
TOTAL                          ≈ 130 – 218 dev-days
```

| Team | Calendar time (incl. PoC, UAT, go-live) |
|---|---|
| 2 backend + 1 frontend + part-time designer/QA | ≈ **22 – 28 weeks** |
| 2 backend + 2 frontend + designer | ≈ **16 – 20 weeks** |

⚠️ Planning estimate, not a commitment. PostEx API, payment provider, design approval and data quality can move it. The increase vs v1.2 (~+75%) is the real cost of a custom UI — the client should see this number.

---

## 8. Implementation phases (proposed)

| Phase | Weeks | Content | Exit gate |
|---|---|---|---|
| **A0 Architecture PoC** | 2 | Same-origin proxy · login + CSRF + realtime from Next.js · headless Webshop cart → Sales Order → reservation · permissions through API (allowed/denied) · Workflow approval via API · the 6 business flows from plan v1.2 | ADR-002/004/005 confirmed or changed |
| **A1 Foundations** | 2–3 | Backend core setup (company, accounts, roles, security) ‖ Next.js foundation + design system ‖ design phase starts | Login works, roles correct, tokens approved |
| **A2 Back office (Desk)** | 3 | Products, pricing, stock, purchasing, customers, accounts config | Store/accounts UAT scripts pass |
| **A3 Sales + approvals** | 2–3 | POS (branded), wholesale, discount/credit approvals backend + **approvals inbox** + **products & margin grid** | Credit/discount approval flow F4 passes |
| **A4 Online orders** | 3 | Stages API, phone/social orders, PostEx (`pk_integrations`), **orders board + packing** | 50 test orders Pending → Delivered |
| **A5 Storefront + payments** | 3–4 | Headless storefront, city charge, back-in-stock, payment gateway | Real order end-to-end, Lighthouse ≥ 80 |
| **A6 HR + payroll** | 2–3 (parallel) | HRMS setup, machine sync, payroll rules | One month payroll matches manual |
| **A7 Dashboards + reports** | 2 | **Owner dashboard**, workspaces, custom reports | Owner sign-off |
| **A8 Migration, UAT, go-live** | 3 | Data migration, security review, performance test, training, cut-over, hypercare | UAT signed |

---

## 9. Things I am challenging (please decide)

| # | Topic | My position |
|---|---|---|
| C1 | Next.js for the **staff** app | OK, but it is not Frappe's own pattern (they use Vue + frappe-ui). Choose by team skill (ADR-002). Storefront in Next.js: yes |
| C2 | Custom **POS** in v1 | **No.** Use ERPNext POS, brand it, decide later with real cashier feedback (ADR-006) |
| C3 | Rebuilding **reports** in Next.js | **No.** Use the 32 standard reports in Desk; Next.js only for the owner dashboard |
| C4 | Rider settlement, HR, accounts screens in Next.js | **No for v1.** Low volume or expert users — Desk is enough |
| C5 | Storefront with its **own** cart/pricing logic | **No.** Use Webshop's APIs so pricing/coupons stay in one place (ADR-004) |
| C6 | Cost increase | ≈ **+75% effort**. Client must approve scope and budget before we commit |

---

## 10. Next step after your review

1. You decide ADR-002 (team skill) and C1–C6.
2. Client approves ADR-001 (tech change) and the budget impact (C6).
3. Run **A0 Architecture PoC** (2 weeks).
4. Then I update `IMPLEMENTATION_PLAN.md` to v1.3 with the approved decisions.
