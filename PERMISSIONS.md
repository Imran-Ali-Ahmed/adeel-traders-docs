# Adeel Traders — Permission matrix (draft for review)

> **Status:** APPROVED and built for the areas in Part 1 (P1.1, 11 Nov 2026). Part 2 is refined when each phase starts.
> **Source:** SRS 2.1 (roles), SRS 3.x (what each area does), and the screens built so far.
> **Date:** 11 Nov 2026

## How to read it

| Letter | Meaning |
|---|---|
| **V** | View |
| **C** | Create |
| **E** | Edit |
| **A** | Approve |
| **X** | Export to Excel/PDF |
| **—** | No access |

- Delete is **never** allowed on submitted records (SRS 6: cancelled records are kept). Drafts can be deleted by whoever created them.
- "Own branch" means: restricted to the branch(es) assigned to the user (User Permission on Branch/Warehouse).
- Every cell is enforced on the **server**. The screens only hide what the server would refuse.
- The owner can later make custom roles (P1.2) from these same building blocks, so each row below is a named **capability**.

## Part 1 — Areas that exist today and in Phase 1

| Capability (area) | Owner | Manager | Counter staff | Store | Accountant | HR | Online team |
|---|---|---|---|---|---|---|---|
| `products.view` (list, search, product page) | V X | V X | V | V X | V X | — | V |
| `products.create` (add a product or gift set) | C | C | — | — | — | — | — |
| `products.edit` (change, disable, photo, brand, description, variants, units, gift sets) | E | E | — | — | — | — | — |
| `catalogue.manage` (categories, brands) | C E | C E | — | — | — | — | — |
| `prices.view` (3 selling prices) | V | V | V | V | V | — | V |
| `prices.edit` | E | E | — | — | — | — | — |
| `cost.view` (cost, margin, stock value in Rs) | V | V | — | — | V | — | — |
| `stock.view` (quantities; **not limited by branch**: staff always see other branches' quantities, read-only) | V | V | V | V | V | — | V |
| `stock.move` (transfer between locations) | C | C | — | C | — | — | — |
| `stock.adjust` (damaged, expired, tester, lost, with reason) | C | C | — | C | — | — | — |
| `stock.backdate` (entries dated in the past) | C | C | — | — | — | — | — |
| `stock.opening` (opening stock, P3.2) | C | C | — | — | — | — | — |
| `stock.count` (count by scanning, blind) | C E | C E | — | C E | — | — | — |
| `stock.count.post` (see system quantities and difference, post or cancel) | A | A | — | — | — | — | — |
| `barcode.generate_print` (generate a barcode, print labels) | ✔ | ✔ | — | ✔ | — | — | — |
| `export.lists` (Excel/PDF export, role **Can Export Lists**) | ✔ | ✔ | — | ✔ | ✔ | ✔ | — |
| `import.products` (Excel upload, P3.1) | C | C | — | — | — | — | — |
| `reports.stock` (P3.5) | V X | V X | — | V X | V X | — | — |
| `users.manage` (invite, disable, assign roles and branches) | C E | — | — | — | — | — | — |
| `roles.manage` (custom roles, P1.2) | C E | — | — | — | — | — | — |
| `login_history.view` | V | — | — | — | — | — | — |
| `audit.view` (record of every change) | V X | V | — | — | V | — | — |
| `approvals.inbox` (see pending approvals) | A | A | — | — | — | — | — |
| `settings.company` (company, branches, logo, numbering, tax) | E | — | — | — | E (tax, accounts) | — | — |
| `settings.stock` (minimum stock, alert settings) | E | E | — | — | — | — | — |

## Part 2 — Areas built in later phases (starting point; refined when each phase starts)

| Area (phase) | Owner | Manager | Counter staff | Store | Accountant | HR | Online team |
|---|---|---|---|---|---|---|---|
| Customers and dealers (P4) | C E X | C E X | C E (own branch) | — | V X | — | C E |
| Credit limits (P4) | E | E | — | — | V | — | — |
| Suppliers (P4) | C E | C E | — | V | V | — | — |
| Receive / pay money (P4) | C | C | C (till payments, own branch) | — | C | — | — |
| Reorder list, RFQ, purchase orders (P5) | C E A | C E A | — | V | V | — | — |
| Receive stock against PO, GRN (P5) | C | C | — | C | — | — | — |
| Purchase bills and returns (P5) | C | C | — | — | C | — | — |
| Discounts and price rules (P6) | C E A | C E A | within own limit | — | — | — | within own limit |
| POS and sales orders (P6) | C | C | C (own branch) | — | V | — | C |
| Sales returns and exchanges (P6) | C | C | C (own branch, ≤ limit) | C (receiving) | V | — | C |
| Rider settlement (P6) | C A | C A | — | C | V | — | — |
| Online orders, packing, PostEx (P8) | C E | C E | — | C (packing) | V | — | C E |
| Accounts: cash, bank, expenses, cheques, petty cash (P7) | C E | V | — | — | C E | — | — |
| Expense approval (P7) | A | A | — | — | — | — | — |
| Financial reports, profit and loss (P7) | V X | V X (sales and profit) | — | — | V X | — | — |
| Employees, attendance, leave (P11) | C E X | V (team only) | — | — | — | C E X | — |
| Salary, payroll, payslips (P11) | A | — | — | — | V | C E | — |
| Salary approval (P11) | A | — | — | — | — | — | — |
| Website content, banners, pages (P9) | C E | C E | — | — | — | — | — |
| Dashboards (P12) | owner | manager | — | store | accounts | HR | — |

## Branch rule

| Role | Sees and acts on |
|---|---|
| Counter staff | **Own branch only** for sales, returns and stock actions. Other branches: read-only quantity (`stock.view_other_branches`). |
| Store | **Own branch only** for receiving, moves, adjustments and counts. |
| Owner, Manager, Accountant, HR, Online team | All branches |

## Things this matrix changes compared with today

| # | Today | After |
|---|---|---|
| 1 | "Can see cost" is a hard-coded list of 6 ERPNext roles | A real role **Can View Cost**, given to Owner, Manager and Accountant. Store, Purchase and Item roles no longer see cost by default. |
| 2 | Only Sales Master Manager can edit prices (Item Manager cannot) | `prices.edit` is a clear capability for Owner and Manager |
| 3 | Counter staff (ERPNext "Sales User") can create Product Bundles from Desk | Removed: only `products.edit` roles |
| 4 | Online team has Stock User, so it can create stock entries | Reduced to what packing needs |
| 5 | Store can only generate barcodes if it can edit products | Separate `barcode.generate_print` capability |
| 6 | Counter and Store see all branches | Own branch for actions, read-only quantities elsewhere |
| 7 | Sidebar shows every item to every role | Hidden by capability |

## Open points for the owner

1. Should Manager see **Accounts** (cash, bank, expenses)? Matrix says **view only**.
2. Should Store staff see **cost**? Matrix says **no**.
3. Should Online team be able to give discounts? Matrix says **within a limit set by the owner** (P6).
4. Should Counter staff export lists? Matrix says **no**.

## How it is built (P1.1)

| Piece | Where |
|---|---|
| Capability map (one line per capability, each answered by a real permission check or a role) | `adeel_traders/permissions.py` |
| `session.me` returns `capabilities` and the user's `branches` | `api/v1/session.py` |
| Roles that switch a capability on: **Can View Cost**, **Can Export Lists** (owner gives them to custom roles in P1.2) | fixtures + `setup/core_setup.py` |
| Role profiles for the 7 roles | `fixtures/role_profile.json` |
| Branch rule: User Permission on Branch + `branch` field on Warehouse; stock leaves only from own branch; counts of other branches are hidden | `permissions.py` (Stock Entry and Stock Count hooks) |
| Screens: `useCan()`, `RequireCan`, menu by capability, "No access" page | `web/apps/staff/components/session.tsx`, `no-access.tsx`, `app-shell.tsx` |
| Tests: every role against the matrix, direct API calls, branch rule | `api/v1/test_permissions_api.py` |

## Changed from the draft

- `stock.view_other_branches` was merged into `stock.view`: reading quantities is never limited by branch.
- Export uses a role (**Can Export Lists**) because ERPNext's own export flag is on for every staff member.
- Online order team lost the Stock User role. Packing rights are added exactly when Phase 8 needs them.
- Accountants got **read-only** access to stock quantities.
- Still open for later: Desk access for non-admin staff (D1 says Desk is for admins only) is closed in P1.3 (Users).

## Roles & permissions page (P1.2, built 11 Nov 2026)

| What | Detail |
|---|---|
| Where | Settings, Roles & permissions (`/settings/roles`). Only users with `roles.manage` (the owner). |
| Built-in roles | The 7 roles above. Locked: they open read-only, with **Copy as new role**. |
| Your roles | The owner makes a role (empty or copied), ticks See / Add / Change / Delete (and Post counts) per area, and two switches: **See cost, margin and stock value** and **Export lists**. |
| Stored as | A Frappe Role (`at_managed = 1`) + Custom DocPerm rows + the two switches on the Role. The server enforces it on every call. |
| Areas list | `adeel_traders/permission_areas.py`. **Every later phase adds its own areas here**, so the page, this matrix and the server checks never drift apart. |
| Safety | Built-in and standard roles cannot be changed or deleted. A role in use cannot be deleted. Roles and users management cannot be given to a custom role. Half-working ticks are refused (for example Add without See). Every change is written to the role's history. |
| API rule | Never name an API argument `flags`: Frappe's HTTP layer drops it. Direct Python tests would not notice, so `test_save_role_works_over_http` goes through the real handler. |
| Not yet | The **Approve** column appears when approvals exist (P1.6). Giving a role to a person is on the Users page (P1.3). |

## Capabilities added in P1.5–P1.12, all given from the Roles page (SRS 2.1)

The owner decides who gets these on **Settings > Roles & permissions** (a tick per role). The
built-in roles start with the access below. Nothing is fixed in code: the check asks Frappe.

| Capability | How the Roles page gives it | Starts with |
|---|---|---|
| `reports.stock` | Area **Reports** (puts the role on the report's role list, Frappe's Custom Role) | Owner, Manager, Store, Accountant |
| `audit.view` | Switch **See the change history** (role flag; the API reads the records itself, so nobody gets raw access to Frappe's change records) | Owner, Manager, Accountant |
| `approvals.inbox` | Switch **Decide approvals**, or any role an active approval workflow names | Owner, Manager |
| `units.manage` | Area **Units** | Owner, Manager |
| `payments.manage` | Area **Payment methods** | Owner, Accountant |
| `email.manage` | Area **Email settings** | Owner |
| `integrations.view` | Area **Integrations** (See / Manage = try a failed call again) | Owner |
| `numbering.view` | Area **Numbering** | Owner |

Not on the Roles page on purpose: `users.manage`, `roles.manage`, `company.manage` (a role that can
make roles or users could give itself everything).
