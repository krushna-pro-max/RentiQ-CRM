# Product Requirements Document (PRD)
## Rental Outfit CRM

| | |
|---|---|
| **Document Owner** | Product Analyst |
| **Status** | Draft v1.0 |
| **Last Updated** | 17 August 2026 |
| **Stakeholders** | Founder/Owner, Store Staff (Booking/Receipt Desk), Alteration Team, Accounts |

---

## 1. Executive Summary

The business currently manages outfit rentals (bookings, customers, payments, alterations, and stock) through manual registers/spreadsheets. This PRD defines a purpose-built **Rental Outfit CRM** with three core modules:

1. **Customer & Booking Management** – single source of truth for every customer, booking, and payment.
2. **Product Catalog** – master list of rentable outfits with pricing, tax, and discount rules.
3. **Product Availability Checker** – a date-range calendar/search tool to instantly see whether a specific product (or product code) is free to book, since each outfit is a physical, non-duplicable unit.

The system replaces manual double-booking checks, manual balance calculation, and scattered payment records with a single workflow-driven tool.

---

## 2. Problem Statement

- Store staff currently track bookings in spreadsheets/registers, making it hard to know **which outfit is free on a given date** before confirming a booking.
- Payment status (Paid / Partial / Balance Due) is manually calculated, leading to errors.
- No centralized customer history (repeat customers, ID proof, past rentals).
- No visibility into product-level performance (most rented, most profitable, most damaged/altered).
- Alteration requests and special notes get lost between booking and pickup.

## 3. Goals & Objectives

**Business Goals**
- Eliminate double-booking of the same physical outfit.
- Reduce payment/balance reconciliation errors to near zero.
- Provide a searchable customer + booking history for repeat business and follow-ups.
- Provide basic sales/inventory reporting (revenue, top products, source of leads).

**User Goals**
- Front-desk staff can create a booking end-to-end in under 2 minutes.
- Staff can check "Is Product X available from [Pick-up Date] to [Return Date]?" in one screen, with zero manual cross-checking.
- Owner/Accounts can see outstanding balances and follow-ups due at a glance.

**Success Metrics** (see Section 12)

## 4. Non-Goals / Out of Scope (v1)

- Online/self-serve customer-facing booking website (this is an internal staff-facing CRM).
- Online payment gateway integration (v1 records payments made offline: Cash/UPI/Card).
- Multi-store/multi-location inventory transfer (assume single store in v1).
- Automated SMS/WhatsApp reminders (flagged as a fast-follow, not v1 — see Roadmap).

---

## 5. Personas

| Persona | Role | Key Needs |
|---|---|---|
| **Front Desk / Booking Staff** | Creates customer records, bookings, collects payment | Fast entry, instant availability check, clear payment status |
| **Alteration Staff** | Handles fitting/alteration requests | Clear alteration notes tied to booking & pickup date |
| **Accounts / Owner** | Tracks revenue, dues, performance | Dashboards, balance-due lists, product performance reports |
| **Store Manager** | Manages catalog, pricing, discounts | Easy catalog CRUD, bulk price updates |

---

## 6. Key User Stories

1. As a **booking staff member**, I want to check if "Product Code PRD-1023" is free between 12 Sep and 15 Sep, so I don't double-book it.
2. As a **booking staff member**, I want to create a new customer + booking in one flow, so I don't re-enter data across screens.
3. As a **booking staff member**, I want the system to auto-calculate Balance = Total Amount − Amount Paid, so I don't make manual math errors.
4. As an **accounts user**, I want to filter all bookings with `Payment Status = Balance Due`, so I can follow up for pending payments.
5. As a **manager**, I want to add/edit products with rent, deposit, tax, and discount rules, so pricing stays consistent across bookings.
6. As a **staff member**, I want to see Special Notes and Alteration details on the booking, so the pickup is prepared correctly.
7. As an **owner**, I want to see which Source (Instagram, Walk-in, Referral, etc.) brings the most bookings, so I can plan marketing spend.

---

## 7. Functional Requirements

### 7.1 Module A — Customer & Booking Management

This is the primary operational screen (the "Customer List" described by the business), extended into a full booking record.

**Core capability:** every row = one booking event tied to a customer, a product, and a date range.

**Field specification** (based on provided fields, with type/behavior recommendations):

| Field | Type | Notes / Business Rule |
|---|---|---|
| # (Record ID) | Auto-number | System generated, unique, immutable |
| Created On | Date-time | Auto-captured on record creation |
| Name | Text | Customer full name; searchable |
| Stage | Dropdown (Pipeline) | Recommended values: `Enquiry → Confirmed → Alteration in Progress → Ready for Pickup → Picked Up → Returned → Closed → Cancelled` |
| Follow-up | Date | Next follow-up date; drives a "Follow-ups Due Today" view |
| Booking Date | Date | Date booking was made |
| Receipt Number | Text/Auto-number | Unique per transaction; printable on receipt |
| Product | Linked field (from Catalog) | Pulled from Product Catalog; supports multi-product booking (see 7.1.1) |
| Alterations | Text / Tag list | Free text or structured tags (e.g., "Sleeve -1in", "Waist +2in") |
| Product Code | Auto-filled from Product | Read-only, pulled from selected Product |
| Special Note | Text (long) | Free text, visible on booking summary |
| Pick Up Date | Date | **Drives availability engine** |
| Return Date | Date | **Drives availability engine**; must be ≥ Pick Up Date |
| Total Rent | Currency, auto-calc | Defaults from Product's Rent Amount, editable (for negotiated price) |
| Total Deposit | Currency, auto-calc | Defaults from Product's Deposit Amount, editable |
| Total Amount | Currency, auto-calc | = Total Rent + Total Deposit + Tax − Discount |
| Amount Paid | Currency | Sum of all payment entries (supports partial/multiple payments) |
| Balance if Any | Currency, auto-calc | = Total Amount − Amount Paid (read-only) |
| Payment Status | Auto-calc tag | `Unpaid` (Paid=0) / `Partially Paid` (0<Paid<Total) / `Paid` (Paid≥Total) |
| Paid Via | Dropdown | Cash / UPI / Card / Bank Transfer / Multiple |
| UPI Name | Text | Only shown/required if Paid Via = UPI |
| ID Card Number | Text | For deposit/KYC purposes |
| ID Card Type | Dropdown | Aadhaar / PAN / Driving License / Passport / Voter ID |
| Phone Number | Text, validated | 10-digit validation; duplicate-check to detect repeat customers |
| Customer By | Dropdown/User | Staff who onboarded the customer |
| Receipt By | Dropdown/User | Staff who issued the receipt (may differ from Customer By) |
| Source | Dropdown | Walk-in / Instagram / Facebook / Referral / Google / Other |
| User | System field | Logged-in staff creating/editing the record (audit trail) |

> **Note on original field list:** "ID Card Type Phone Number" appears to be two fields merged in the source list (`ID Card Type` and `Phone Number`). This PRD treats them as two separate fields — please confirm.

**7.1.1 Multi-product bookings:** A single customer visit may rent more than one outfit (e.g., outfit + accessories). Recommend the booking record support a **line-item sub-table** (Product, Product Code, Rent, Deposit, Alteration) under one Receipt Number, rather than forcing one product per row. This avoids duplicate customer records for multi-item rentals. *(Flagged as a v1 design decision — see Open Questions.)*

**7.1.2 Views/Filters required:**
- All Bookings (default, sorted by Booking Date desc)
- Follow-ups Due (Today / This Week)
- Balance Due (Payment Status ≠ Paid)
- By Stage (Kanban-style board across the Stage pipeline)
- By Pickup Date / Return Date (for daily ops — "what's going out today", "what's due back today")
- Search by Name / Phone / Receipt Number / Product Code

**7.1.3 Validation rules:**
- Return Date cannot be before Pick Up Date.
- Cannot confirm a booking (Stage → Confirmed) if the Product is already booked for an overlapping date range (hard block, with override permission for managers).
- Phone Number + ID Card Number recommended as duplicate-detection keys to merge/identify repeat customers.

---

### 7.2 Module B — Product Catalog

Master data for every rentable item.

| Field | Type | Notes |
|---|---|---|
| Name | Text | Outfit name (e.g., "Red Silk Lehenga") |
| Brand | Text/Dropdown | Manufacturer/designer label |
| Category | Dropdown | e.g., Lehenga, Sherwani, Gown, Suit, Accessories |
| Sub Category | Dropdown (dependent on Category) | e.g., under Lehenga: Bridal, Party Wear |
| Product Price | Currency | MRP / retail value of the item (for insurance/loss reference) |
| Discount on MRP (%) | Percentage | Standard discount applied to rent calc |
| Rent Amount | Currency | Base rental price |
| Deposit Amount | Currency | Refundable security deposit |
| Unit Tax | Percentage or fixed | GST/tax applied |
| Max Discount | Percentage | Ceiling — prevents staff from discounting below this at booking time |
| Product Code | Text, unique | Primary key referenced by bookings and availability |
| Description | Text (long) | Fabric, size, color, condition notes |

**Additional recommended fields (not in original list, flagged as suggestions):**
- **Size** (S/M/L/XL or numeric) — critical for outfit rentals.
- **Color**
- **Photo(s)** — visual reference for staff and catalog browsing.
- **Quantity/Units in stock** — if the same design exists in multiple physical pieces (common in rental businesses), each physical unit needs its own Product Code or a unit-tracking sub-field, otherwise the availability engine cannot distinguish "Design available" vs "This exact piece available."
- **Status** — Active / Under Maintenance / Retired / Lost-Damaged.

**Functional behavior:**
- Product Code auto-generated (editable) with a configurable prefix (e.g., `LHG-0001`).
- Rent/Deposit/Tax fields auto-populate into a new Booking when a Product is selected, but remain editable per booking (for negotiated pricing) without altering the master catalog value.
- Catalog list supports filter by Category/Sub-category/Brand/Status and search by Name/Product Code.
- Bulk import/update via CSV (for onboarding existing inventory).

---

### 7.3 Module C — Product Availability Checker (core differentiator)

**Purpose:** Instantly determine whether a product (by name or Product Code) is available for a requested date range, before a booking is confirmed.

**7.3.1 Search/Check flow:**
- Input: Product (search by Name/Code/Category) + Date Range (Pick Up Date → Return Date).
- Output: **Available** / **Not Available**, and if not available, show the conflicting booking(s) — Receipt Number, Customer Name, Booked Return Date — so staff can inform the customer of the next available date.
- Include a configurable **buffer/turnaround time** between bookings (e.g., 1 day for dry-cleaning/inspection before the next rental) as a global or per-category setting.

**7.3.2 Calendar view:**
- A calendar/timeline (Gantt-style) per product or per category showing all booked date ranges at a glance, so staff can visually scan upcoming free slots without typing a specific date range.
- Color-code by Stage (Confirmed / Alteration / Ready / Picked Up) so staff know not just "booked" but "where in process."

**7.3.3 Availability logic:**
```
Product is UNAVAILABLE for [RequestedStart, RequestedEnd] if there exists
any Booking with the same Product Code where Stage is NOT
(Cancelled OR Closed/Returned) AND:

   BookingPickUpDate <= RequestedEnd + Buffer
   AND
   BookingReturnDate + Buffer >= RequestedStart
```
- Cancelled bookings, and bookings already marked "Returned," are excluded from conflict checks (their date range is freed).
- If Product Catalog tracks multiple physical units of the same design (see 7.2 recommendation), availability is calculated **per physical unit/Product Code**, and the system should suggest an alternate unit of the same design if one is free.

**7.3.4 Where this surfaces:**
- Standalone "Check Availability" search page.
- Inline check at the moment of creating/editing a Booking (real-time validation before allowing Stage = Confirmed).
- Calendar view accessible from the Product Catalog record itself ("View this product's booking calendar").

---

### 7.4 Module D — Dashboard & Reports (recommended addition)

Not explicitly requested but necessary to make the data usable:

- Revenue summary (by day/week/month, by Category, by Source).
- Balance Due report (aged: 0–7 days, 8–30 days, 30+ days).
- Follow-ups due today/this week.
- Top 10 most-booked products; least-booked (dead stock) products.
- Staff performance (bookings by Customer By / Receipt By).

---

## 8. Data Model (Entity Relationships)

```
Customer (1) ────< Booking (many)
Product Catalog (1) ────< Booking Line Item (many)
Booking (1) ────< Payment (many)     [supports partial/multiple payments]
Booking (1) ──── Alteration Notes (embedded/sub-record)
```

- **Customer** is a distinct entity (deduplicated by Phone Number/ID Card Number) even though it's entered via the Booking screen — prevents duplicate customer records across repeat visits.
- **Booking** is the transactional core, referencing one Customer and one-or-more Products (line items) via Product Code.
- **Payment** is broken out as its own sub-entity so "Amount Paid" can reflect multiple partial payments over time (advance at booking, balance at pickup, etc.), each with its own Paid Via/date.
- **Product Catalog** is the master; Booking pulls Rent/Deposit/Tax at time of booking as a snapshot (so later price changes in the catalog don't retroactively alter past bookings).

---

## 9. Business Rules Summary

| Rule | Behavior |
|---|---|
| Balance calculation | `Balance = Total Amount - Amount Paid` (always system-calculated, never manually entered) |
| Payment Status | Auto-derived from Balance, not manually set |
| Double-booking prevention | Hard block at Stage = Confirmed if date range overlaps an active booking on the same Product Code |
| Total Amount calculation | `Total Amount = (Rent + Deposit) + Tax - Discount`, discount capped at Product's Max Discount |
| Product Code | Immutable once assigned; used as the join key between Catalog, Booking, and Availability |
| Cancelled bookings | Automatically release the product's date range back to available |

---

## 10. Non-Functional Requirements

- **Roles & Permissions:** Admin/Owner (full access + reports), Manager (catalog + override booking conflicts), Staff (create/edit bookings, no catalog price edits, no deleting records).
- **Audit trail:** every create/edit logs the `User` and timestamp (already a listed field — extend to full change history).
- **Data integrity:** Product Code and Receipt Number must be unique (system-enforced).
- **Performance:** Availability check should return a result in <1 second even with a full season of bookings.
- **Access:** Web-based, usable on desktop (front desk) and tablet/mobile (for quick lookups on the shop floor).
- **Backup:** Daily automated backup of all records (customer PII and payment data involved).
- **Data privacy:** ID Card Number and phone number are sensitive — restrict export/visibility to Admin/Manager roles; mask ID numbers in list views, show full number only on record open.

---

## 11. Recommended Screens (Information Architecture)

1. Dashboard (Home)
2. Customers & Bookings (list + Kanban by Stage + record detail)
3. New Booking (guided flow: Select/Create Customer → Select Product(s) + auto availability check → Pricing → Payment → Receipt)
4. Product Catalog (list + record detail + bulk import)
5. Product Availability Checker (search + calendar/timeline view)
6. Reports (Revenue, Balance Due, Follow-ups, Product Performance)
7. Settings (Users/Roles, Stage pipeline config, Source list, Buffer/turnaround settings)

---

## 12. Success Metrics (KPIs)

| Metric | Target |
|---|---|
| Double-booking incidents | 0 per month (down from manual-tracking baseline) |
| Time to create a booking | < 2 minutes |
| Balance reconciliation errors | 0 (fully system-calculated) |
| % of bookings with correct availability check before confirmation | 100% |
| Staff adoption (vs. old register/spreadsheet) | 100% within 4 weeks of launch |

---

## 13. Assumptions

- Single store/location for v1.
- Each row in the current spreadsheet represents one booking transaction, not strictly one unique customer (a customer may have multiple rows over time).
- Deposit is refundable and tracked, but refund workflow is not detailed in the source fields — flagged as an open question.

## 14. Risks

| Risk | Mitigation |
|---|---|
| Same outfit design has multiple physical copies but only one Product Code | Introduce unit-level tracking (Section 7.2) before launch, or availability data will be wrong |
| Staff bypass availability check | Make the check a hard gate in the Booking flow, not a separate optional tool |
| Data migration from existing spreadsheet has inconsistent formats | Run a data-cleaning pass before import; validate phone numbers/dates |

---

## 15. Open Questions (need business input before build)

1. Can one customer rent **multiple products** in a single visit under one Receipt Number? (Assumed yes — recommend line-item design in 7.1.1.)
2. Does the same outfit design ever exist in more than one physical piece/size? If yes, how should each physical unit be uniquely coded?
3. Is there a deposit **refund** workflow to track (e.g., partial deduction for damage)? Not present in the current field list.
4. Should "ID Card Type" and "Phone Number" be confirmed as two separate fields (they appear merged in the source list)?
5. What are the exact Stage pipeline values the business currently uses, if different from the recommended set in 7.1?
6. Is a buffer/turnaround time between bookings needed (e.g., dry cleaning), and how many days?

---

## 16. Suggested Release Plan

| Phase | Scope |
|---|---|
| **Phase 1 (MVP)** | Customer & Booking Management, Product Catalog, Availability Checker (search + hard-block), core Dashboard |
| **Phase 2** | Calendar/timeline view, multi-product line-item bookings, roles & permissions, reports |
| **Phase 3** | WhatsApp/SMS follow-up & pickup/return reminders, deposit refund workflow, multi-store support |

