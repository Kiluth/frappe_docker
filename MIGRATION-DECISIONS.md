# Custom DocType Migration Decisions

Per-doctype audit of prod's 17 custom doctypes. For each: keep / drop / replace
with native + reasoning. Used to design the new local instance.

## Decisions

### 1. Problem Report → **REPLACE with native `Issue`**
- Internal "tell HR there's a problem" form
- Native `Issue` gives status workflow, priority, email threading, assignment,
  SLA tracking, dashboard for free
- Customer/Project/SLA fields stay blank (irrelevant for internal use)
- Web form `/problem-report/new` rewritten to create an Issue with mapped fields
- Permissions: add HR Manager to Issue

### 2. New Employee Application → **KEEP** (drop redundant Full Name field)
- Self-service form for new hires to give HR personal/employment/payroll info
- Documented in `kiluth-docs/content/hr/new-employee-onboarding-guide` — referenced
  by name as the staging step before HR creates an Employee record
- Naming series `NEA-{YYYY}-{#####}` preserved
- Drop `full_name` field — redundant with First + Last
- **Follow-up (separate from doctype):**
  - Notification rule on creation → email `hr@kiluth.com` + submitter confirmation
  - Field-level permissions: DOB, Personal Email, Mobile, Bank Name, Bank Account No,
    Emergency Contact Name, Emergency Phone → restricted to HR Manager + Accounts Manager

### 3. Employee Request → **DROP, use native `Material Request`** with stock catalog
- Native procurement pipeline: Material Request → Purchase Order → Receipt → Asset
- One-time setup: ~50 anchor Items in the catalog (laptops, monitors, common
  software, peripherals) + per-category "Other (specify in description)" fallback
- HR/Procurement creates new Items on demand, never employees (avoid catalog drift)
- "Other ..." items get promoted to real Items when demand justifies it
- Subscriptions (Figma, Adobe CC monthly) handled separately via ERPNext Subscriptions
  later — not day-1 work

### 4. Equipment Loan Agreement → **KEEP** (Option D — add server script to sync Asset.custodian)
- Primary purpose: legal acknowledgment that employee agreed to terms (use only
  for work, liability for loss/damage, return obligations, data security)
- The 12-section legal text lives in `kiluth-docs/content/hr/equipment-loan-agreement`
- Form captures: employee, list of assets, condition at issue, accessories,
  agreement checkbox, date signed
- **Server Script on `on_submit`:** create matching Asset Movement records for
  each asset in the table → ERPNext's standard Asset.custodian field auto-updates
  → "who has what" reporting works natively
- ~20 lines of Python in a Server Script (built-in feature, no app rebuild)

### 5. Equipment Loan Asset (child table of #4) → **KEEP** (covered by #4 decision)

### 6. Equipment Return → **KEEP** (Option D — mirror of #4)
- Counterpart to Equipment Loan Agreement: documented record of asset return
- Captures: employee, assets returned, return date, condition on return,
  accessories missing, **required photo per asset** (evidence for damage disputes),
  damage details, remarks
- **Server Script on `on_submit`:** create Asset Movement records moving each
  asset back to the default storage location → `Asset.custodian` returns to
  null/HR-Storage → loan-return cycle keeps inventory accurate at all times

### 7. Equipment Return Item (child table of #6) → **KEEP** (covered by #6 decision)

### 8. Expense Claim Request → **DROP, use native HRMS `Expense Claim`**
- Custom doctype was a simplified, broken duplicate of HRMS Expense Claim
- Native HRMS gives: approval workflow with Expense Approver, sanctioned vs
  claimed amount, auto GL entry on submit, Payment Entry integration on
  reimbursement, native dashboards
- Finance docs (`erpnext-financial-recording-guide`) don't even mention the
  custom doctype — Finance was working around it
- Web form `/expense-claim-request/new` rebuilt on top of standard Expense Claim
- Existing Expense Claim Type categories preserved (already standard HRMS)

### 9. Expense Claim Request Detail (child of #8) → **DROP** (covered by #8)

### 10. Meeting Room Booking Request → **DROP, use Google Calendar Resources**
- ERPNext has no native room booking; the custom doctype lacks conflict detection,
  isn't on anyone's actual calendar, and forces double-entry (book in ERPNext + also
  create Google Calendar event for invitees)
- Google Calendar Resources gives: real conflict detection, rooms on everyone's
  calendars, email invites, auto Google Meet, recurring meetings, capacity/amenities
  in resource metadata
- Setup deferred (Kiluth has Google Workspace admin access; ~5 min per room in
  Google Workspace Admin Console)

### 11. Meeting Rooms (lookup) → **DROP** (covered by #10 — handled in Google Workspace)

### 18. Amenity (child of Meeting Rooms) → **DROP** (covered by #10)

### 17. WiFi Access Request → **DROP**
- Wrong tool for credential management; should not live in ERPNext
- Move WiFi credentials to a password manager (Bitwarden / 1Password / Google
  Password Manager) — encrypted, audited, role-based access
- Onboarding: HR shares WiFi password verbally on day 1, or new employee gets
  vault access from password manager admin
- If a structured "guest WiFi request" flow is ever needed, a Google Form →
  email IT covers it without a custom doctype

### 16. WiFi Access Points → **DROP** (covered by #17)
- ⚠️ Note: was storing passwords in **plaintext `Data` field**, not `Password`
  field. Anyone with System Manager or Web Form Manager role could read all
  WiFi creds. This is a real security issue in the live prod instance — should
  be removed from prod ahead of full migration if any sensitive creds are stored.

### 14. Resource → **KEEP + clean up**
- Documented in `kiluth-docs/content/account-management/maintenance-and-hosting-guideline`
- Tracks infrastructure Kiluth provisions for clients (servers, domains, MA)
- Drives the renewal flow with expiry-date notifications (30/15/5/3/1 days)
- 28 active records to migrate
- **Cleanup:**
  - Move from `Accounts` module → new `Kiluth Hosting` module
  - **Cost / Charge stay `Data`.** Originally planned to convert to `Currency`, but
    some resources are subscription-based ("200 THB/month") which `Currency` can't
    express. Free-text `Data` wins; margin math gets done manually.
  - **KEEP `Created By` / `Created Date`** — per the docs these capture WHO provisioned
    and WHEN in real life (not when ERPNext record was typed); distinct from Frappe's
    automatic `owner` / `creation` fields. By-field stays `Link → User` (matches prod;
    guideline's "Link to Employee" intent was diverged-from in prod and not enforced here).
  - Keep `Issued By` / `Issued Date` (issued to client, per docs; `Link → User`)
  - Status vocab: match prod's `Draft / Planned / Active / Expired / Archived / Deleted`
    (richer lifecycle than my initial `Active / Expiring Soon / Expired / Cancelled`;
    "expiring soon" is a computed display, not a stored state)
- **Fold in `ma_period_months`** as a native field (see Custom Fields section below)
- **Scripts to port over** (3 total):
  - Server Script `Resource Status Auto-Calculate` (Before Save) — auto-sets status
    based on dates + auto-calculates MA cost via `((25% × project.estimated_costing)
    / 12) × ma_period_months`
  - Client Script `Auto Calculate MA Cost in Resource Form` — live form UX for the
    MA cost calculation
  - Client Script `Filter Projects Based on Customer in Resource Form` — narrows
    Project dropdown to selected Customer's projects
- **Follow-up:** Notification rules for expiry warnings (per the guideline's renewal timeline)

### 15. Resource Type → **KEEP + move module**
- Lookup table for Resource (3 records: Server, Domain name, MA)
- Move from `Maintenance` module → same module as Resource (`Kiluth Hosting`)

### 2. Client Admin → **DROP**
- Not documented anywhere; Poom doesn't recognize it; only 1 stale record
- Stores client admin credentials → wrong tool (ERPNext is not a password manager)
- Move all client credentials to 1Password Teams / Bitwarden Business with one
  shared vault per client
- Spot-check the 1 existing record before deletion to ensure the credential isn't
  in active use anywhere

---

## Summary tally (all 17 audited)

**KEEP (7 doctypes):**
- 4. Equipment Loan Agreement (+ child #5) — add Asset Movement sync
- 6. Equipment Return (+ child #7) — add Asset Movement sync
- 12. New Employee Application — drop redundant Full Name field
- 14. Resource — clean up + module move
- 15. Resource Type — module move

**DROP (10 doctypes):**
- 1. Amenity (was child of Meeting Rooms)
- 2. Client Admin → password manager
- 3. Employee Request → native Material Request + stock catalog
- 8. Expense Claim Request (+ child #9) → native HRMS Expense Claim
- 10. Meeting Room Booking Request → Google Calendar Resources
- 11. Meeting Rooms → Google Calendar Resources
- 13. Problem Report → native Issue
- 16. WiFi Access Points → password manager
- 17. WiFi Access Request → onboarding handoff / Google Form

**Net result on the new instance: 7 custom doctypes** instead of 17. Most of what
was custom is now either native or replaced with a more appropriate tool.

---

## Custom Fields on Standard Doctypes

Most auto-added by HRMS / Frappe regional / ERPNext defaults — those appear
automatically on local when the apps are installed, no action needed. Only
Kiluth-added fields need decisions:

### F1. `Asset.Serial Number` (Data) → **KEEP**
- Standard Asset doctype has no built-in serial number field
- Essential for distinguishing physical assets (laptops, monitors)

### F2. `Customer.Unique Email ID` (Data, unique+required) → **KEEP as-is**
- Standard `email_id` is Read-Only (auto-populated from Contact); this field is
  a deliberate dedup mechanism to prevent creating the same Customer twice
- For customers without email, create one for them

### F3. `Lead.Budget` (Data) → **KEEP + fix fieldtype to `Currency`**
- No native budget field on Lead (standard `annual_revenue` is the lead's
  company revenue, not project budget)
- Current `Data` fieldtype breaks currency formatting, aggregation, reporting

### F4. `Lead.Lead Rating` (Select: Potential/Qualified/Win/Loss) → **DROP**
- Overlaps with standard `status` (covers Win/Loss via Converted / Lost Quotation)
  and standard `qualification_status` (covers Qualified)
- Lose "Potential" vocabulary — "Unqualified" in `qualification_status` covers it

### F5. `Resource.custom_ma_period_months` (Int) → **KEEP + fold into native Resource**
- Load-bearing for the auto-calc server script (drives MA cost formula)
- Rename to `ma_period_months` (drop `custom_` prefix) as native Resource field
- Update the Server Script's 2 references to new fieldname

### F6. `Terms and Conditions.HR` (Check) → **DROP**
- Incomplete placeholder (empty "Remark" record created 2026-03-23, never filled)
- Legal text stays in `kiluth-docs`, not in ERPNext T&C records
- Drop the empty "Remark" record too

---

## Workflows

Only 3 workflows exist in prod — all on doctypes being dropped (Meeting Room
Booking Request, Expense Claim Request, Employee Request). They vanish with
their doctypes. **Zero workflows to migrate.**

Replacements provide native workflow equivalents:
- HRMS Expense Claim → native `Expense Approver` approval flow
- Material Request → native Draft → Pending → Approved → Ordered
- Issue (for Problem Report) → native Open → Replied → On Hold → Closed
- Google Calendar Resources (for Meeting Room) → not ERPNext

---

## Notifications

**26 total in prod, 6 are HRMS/ERPNext built-ins (auto-installed), 20 are Kiluth-custom.**

### Port directly to kept doctypes (12)
- **Resource** (8): expiry warnings at 30/15/5/3/1 days, on expiry, on assignment,
  on expiry-date update — matches the Maintenance & Hosting docs
- **New Employee Application** (2): HR alert + submitter confirmation (per onboarding docs)
- **Equipment Loan Agreement** (1): new loan alert
- **Equipment Return** (1): new return alert

### Layer on standard doctypes (2)
- **Leave Application** (HRMS native): HR alert + status update — port both

### Drop entirely (1)
- **Timesheet** "New Timesheet" — per-submission email is noise; use list view
  or a weekly digest if tracking needed

### Vanish with dropped doctypes — with replacement plan (5)
- Employee Request → **ADD** "New Material Request" notif on Material Request
  (recipients: HR Manager / Procurement role, trigger: New)
- Expense Claim Request → **SKIP** custom — rely on HRMS native Expense Approver
  notifications (already wired via `Employee.expense_approver`)
- Meeting Room Booking Request → **SKIP** — Google Calendar sends invites natively
- Problem Report → **ADD** scoped Issue notification: create Issue Type
  "HR - Problem Report", notification fires only when `issue_type` matches,
  recipients: HR Manager
- WiFi Access Request → no replacement needed

---

## Naming Series

Only 1 override on a standard doctype in prod: `Salary Slip` autoname overridden
to `SALARY-SLIP-.#####`.

**Decision: DROP the override, use ERPNext default.** The default uses a
`naming_series` dropdown with options like `Sal Slip/.YYYY./.MM./.#####` which
puts pay period in the ID (better for payroll audits), matches ERPNext convention,
and removes a customization with no specific Kiluth-reason behind it.

Custom doctypes being kept retain their own autoname as part of their definition
(e.g. `NEA-{YYYY}-{#####}`, `RESOURCE-.#####`) — already captured in doctype
decisions above.

---

## Print Formats

30 total in prod — 25 are ERPNext/HRMS standard (auto-install), 5 are non-standard.

### Port (3)
- **Payment Entry · Receipt** — Kiluth-branded receipt layout
- **Quotation · Quotation Print** — Kiluth-branded quotation
- **Sales Invoice · Invoice Print** — Kiluth-branded invoice

Port via export from prod (Print Format → ... → Export JSON) → import to local.
**Sanity-check each rendered output** against Thai locale (date format, address
block, THB currency formatting) before finalizing.

### Drop (2)
- Sales Invoice · Receipt Print — already disabled in prod, unused
- Supplier · IRS 1099 Form — US tax form, irrelevant for Thailand (regional artifact)

---

## Roles & Permission Rules

- **0 custom roles.** All 53 roles in prod are ERPNext/HRMS/Frappe defaults → auto-present on local.
- **87 Custom DocPerm rules**, all owned by Poom. Three groups:
  - **Group A (~20 rules)** on doctypes being dropped → vanish naturally, no action
  - **Group B (~11 rules)** on kept custom doctypes (Equipment Loan Agreement,
    Equipment Return, Resource, Resource Type) → port as part of each doctype's
    rebuild (permissions are part of a doctype's definition)
  - **Group C (~56 rules)** on standard doctypes (Expense Claim, Lead, Sales
    Invoice, Item, Project, Department, Timesheet, Asset, Activity Type,
    Accounts Settings, Lead Source) → **SKIP. Start with ERPNext stock defaults.**
    Rationale: Poom explicitly noted "I made many mistakes" on prod;
    permissions are high-consequence/subtle; safer to inherit ERPNext's
    thought-out defaults and add back only what's actually needed as
    permission errors surface during use.

### Additional artifact noted
Two stale doctype names appeared in Custom DocPerm (`WiFi Access Point` singular,
`WiFi Access Request Old`) — rename/migration leftovers. Not carried over.

---

## Client Scripts & Server Scripts

**17 total in prod: 3 Client + 14 Server. Port 10, drop 7.**

### Port (10)

**Server Scripts — utility (2)**
- `health_check` (API endpoint) — lightweight liveness check used externally
- `Resource Status Daily Recalculation` (Scheduler Event, daily) — re-runs the
  Resource status/expiry calc each morning so the 30/15/5/3/1-day notifications
  fire on time without manual edits

**Server Scripts — on kept custom doctypes (3)**
- Equipment Loan Agreement `on_submit` — creates Asset Movement records for each
  borrowed asset (already captured in doctype decision #4)
- Equipment Return `on_submit` — mirrors the above for returns (decision #6)
- Resource `Before Save` — auto-sets status + auto-calculates MA cost (decision #14)

**Server Scripts — on standard doctypes (3)**
- Expense Claim — autofill `employee` from logged-in user
- Leave Application — autofill `employee` from logged-in user
- Timesheet — autofill `employee` / related fields from logged-in user

**Client Scripts (3)**
- Resource: `Auto Calculate MA Cost in Resource Form` (live UX for the Before Save calc)
- Resource: `Filter Projects Based on Customer in Resource Form`
- Sales Invoice: `Filter Projects by Customer` (same pattern as Resource one)

### Drop (6)

- 1 disabled-in-prod Sales Invoice script (no need to port dead code)
- 5 scripts that live on doctypes being dropped (Problem Report, Employee Request,
  Expense Claim Request, Meeting Room Booking Request, Client Admin) — vanish with
  their doctypes

### Follow-up — consolidate repeated pattern

The "autofill `employee` from `frappe.session.user`" pattern appears ~5 times
across Expense Claim, Leave Application, Timesheet (and was also present on the
HR doctypes being dropped). Rather than 5 near-identical Server Scripts, fold
this into a single utility (e.g. `kiluth_portal.hooks.autofill_employee`)
triggered via `doc_events` once a proper Kiluth customizations app exists.
Until then, ship as standalone Server Scripts to keep day-1 simple.

---

## Cross-cutting decisions / follow-ups (not per-doctype)

- **Chart of Accounts:** start fresh from Standard — do NOT import Kiluth's existing
  `- K` suffixed accounts. Will rebuild as needed.
- **Field-level permissions** are a separate audit pass after doctypes are created.
- **Notification rules** are a separate audit pass after doctypes are created.
- **`kiluth_portal` app:** longer-term, move the repeated autofill and any
  other cross-doctype behaviors out of Server Scripts into a versioned app for
  traceability (see Scripts follow-up above).

