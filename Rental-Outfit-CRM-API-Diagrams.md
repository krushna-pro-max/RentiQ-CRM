# Rental Outfit CRM — API Diagrams

Visual companion to `Rental-Outfit-CRM-API-Documentation.md`. All diagrams are Mermaid — render natively on GitHub/GitLab or any Mermaid-compatible viewer.

---

## 1. Entity Relationship Diagram

Shows how `Customer`, `Booking`, `Line Item`, `Payment`, `Product`, and `User` relate — including the fact that one customer can have many bookings (on different days or the same day), and one booking can have many line items and many payments.

```mermaid
erDiagram
    CUSTOMER ||--o{ BOOKING : "makes"
    BOOKING ||--|{ LINE_ITEM : "contains"
    BOOKING ||--o{ PAYMENT : "receives"
    PRODUCT ||--o{ LINE_ITEM : "referenced by"
    USER ||--o{ BOOKING : "customer_by / receipt_by"
    USER ||--o{ PAYMENT : "recorded_by"

    CUSTOMER {
        string id PK
        string name
        string phone_number
        string id_card_type
        string id_card_number
        string source
        datetime created_on
    }

    BOOKING {
        string id PK
        string customer_id FK
        string receipt_number
        string stage
        date booking_date
        date pick_up_date
        date return_date
        date follow_up
        string special_note
        decimal total_rent
        decimal total_deposit
        decimal total_amount
        decimal amount_paid
        decimal balance
        string payment_status
        string customer_by FK
        string receipt_by FK
        string source
    }

    LINE_ITEM {
        string id PK
        string booking_id FK
        string product_code FK
        string alterations
        decimal rent_amount
        decimal deposit_amount
        decimal discount_percent
    }

    PAYMENT {
        string id PK
        string booking_id FK
        decimal amount
        string paid_via
        string upi_name
        datetime paid_on
        string recorded_by FK
    }

    PRODUCT {
        string id PK
        string product_code
        string name
        string brand
        string category
        string sub_category
        decimal product_price
        decimal discount_on_mrp_percent
        decimal rent_amount
        decimal deposit_amount
        decimal unit_tax_percent
        decimal max_discount_percent
        string description
        string status
    }

    USER {
        string id PK
        string name
        string email
        string role
    }
```

**Key takeaway:** `Booking` is transactional and independent — a `Customer` has zero constraint on how many `Booking` rows they can have, including several on the same `booking_date`. Availability conflicts are checked at the `PRODUCT` ↔ `LINE_ITEM` date-range level, never at the customer level.

---

## 2. Create Booking Flow (with Availability Check)

The critical path: staff create a booking, the server checks product availability before allowing `Confirmed`, and either succeeds or returns a conflict.

```mermaid
sequenceDiagram
    actor Staff
    participant App as Front Desk App
    participant API as Booking API
    participant Avail as Availability Engine
    participant DB as Database

    Staff->>App: Select customer + product(s) + dates
    App->>API: GET /availability/check?product_code&pick_up_date&return_date
    API->>Avail: Evaluate overlap against active bookings
    Avail->>DB: Query bookings for product_code (exclude non-blocking stages)
    DB-->>Avail: Matching bookings (if any)
    Avail-->>API: available: true/false (+ conflicts)
    API-->>App: Availability result

    alt Available
        App->>Staff: Show "Available" (green)
        Staff->>App: Confirm booking details, submit
        App->>API: POST /bookings { customer_id, line_items[], dates }
        API->>Avail: Re-validate availability (race condition guard)
        Avail-->>API: Confirmed available
        API->>DB: Insert Booking + Line Items (stage = Enquiry/Booking Confirmed)
        DB-->>API: booking_id, receipt_number
        API-->>App: 201 Created (booking object)
        App-->>Staff: Receipt generated
    else Not Available
        App->>Staff: Show "Not Available" + conflicting booking + next available date
        Staff->>App: Adjust dates or pick different product
    end
```

---

## 3. Record Payment Flow

Shows how a payment recalculates the booking's balance and payment status automatically.

```mermaid
sequenceDiagram
    actor Staff
    participant App as Front Desk App
    participant API as Payment API
    participant DB as Database

    Staff->>App: Enter payment amount + method
    App->>API: POST /bookings/{id}/payments { amount, paid_via }
    API->>DB: Fetch booking (total_amount, amount_paid)
    DB-->>API: Current booking totals

    alt amount would exceed total_amount
        API-->>App: 422 OVERPAYMENT_NOT_ALLOWED
        App-->>Staff: Show error, ask to confirm/override
    else Valid payment
        API->>DB: Insert Payment record
        API->>DB: Recalculate amount_paid, balance, payment_status
        DB-->>API: Updated booking summary
        API-->>App: 201 Created { payment, booking_summary }
        App-->>Staff: Show updated balance + payment status badge
    end
```

---

## 4. Booking Stage Lifecycle

The `stage` field drives the Kanban board (PRD §7.1.2) and gates the availability check. This state diagram shows valid transitions across the updated pipeline.

**Stages:** `Enquiry` → `Booking Confirmed` → `Pickup Pending` → `Ready for Pickup` → `Pick Up Done` → `Return Pending` → `Return Received/Deposit Pending` → `Successful Leads`, with `Postponed/Credit Note` and `Cancelled/Full Refund` as exit branches available from any pre-pickup stage.

```mermaid
stateDiagram-v2
    [*] --> Enquiry

    Enquiry --> BookingConfirmed: Availability check passes
    Enquiry --> Cancelled: Customer declines

    BookingConfirmed --> PickupPending: Prep / alterations in progress
    BookingConfirmed --> Postponed: Event postponed
    BookingConfirmed --> Cancelled: Customer cancels

    PickupPending --> ReadyForPickup: Prep / alterations complete
    PickupPending --> Postponed: Event postponed
    PickupPending --> Cancelled: Customer cancels

    ReadyForPickup --> PickUpDone: Customer collects outfit
    ReadyForPickup --> Postponed: Event postponed
    ReadyForPickup --> Cancelled: Customer cancels

    PickUpDone --> ReturnPending: Rental period in progress

    ReturnPending --> ReturnReceived: Outfit physically returned

    ReturnReceived --> SuccessfulLeads: Deposit settled / no deductions

    Postponed --> BookingConfirmed: Customer rebooks using credit note
    Postponed --> [*]: Credit note issued, no active hold

    Cancelled --> [*]: Full refund issued, product released
    SuccessfulLeads --> [*]: Booking closed

    note right of Cancelled
        Cancelled/Full Refund releases
        the product's date range
        immediately
    end note

    note right of Postponed
        Postponed/Credit Note also
        releases the current date
        range; customer retains a
        credit note for a future
        booking rather than a refund
    end note

    note right of ReturnReceived
        Return Received/Deposit
        Pending = outfit is back in
        store; product is available
        again even though deposit
        settlement is still open
    end note
```

---

## 5. Product Availability Decision Logic

Visualizes the conflict rule from PRD §7.3.3 / API doc §7.1, updated for the new stage list.

**Non-blocking stages** (product is free, excluded from conflict check): `Return Received/Deposit Pending`, `Successful Leads`, `Postponed/Credit Note`, `Cancelled/Full Refund`.
**Blocking stages** (product is held/reserved, included in conflict check): `Enquiry`, `Booking Confirmed`, `Pickup Pending`, `Ready for Pickup`, `Pick Up Done`, `Return Pending`.

> Flag for the business: treating `Enquiry` as blocking is a conservative default (a tentative enquiry holds the date). If enquiries should be soft holds that don't block other customers, move `Enquiry` to the non-blocking list — confirm before build.

```mermaid
flowchart TD
    A[Request: product_code + pick_up_date + return_date] --> B{Any booking on same\nproduct_code?}
    B -- No --> G[Available ✅]
    B -- Yes --> C{Is that booking's stage one of:\nReturn Received/Deposit Pending,\nSuccessful Leads,\nPostponed/Credit Note,\nCancelled/Full Refund?}
    C -- Yes --> D[Ignore this booking\nProduct already free\nCheck next]
    C -- No --> E{Does date range overlap?\npick_up_date <= existing.return_date + buffer\nAND\nreturn_date + buffer >= existing.pick_up_date}
    E -- No overlap --> D
    E -- Overlaps --> F[Not Available ❌\nReturn conflicting booking\n+ suggest next_available_date]
    D --> B
```

---

## 6. Auth Flow

```mermaid
sequenceDiagram
    actor Staff
    participant App
    participant Auth as Auth API

    Staff->>App: Enter email + password
    App->>Auth: POST /auth/login
    Auth-->>App: access_token (60 min) + refresh_token (30 days)
    App->>App: Store tokens, attach access_token to all requests

    Note over App,Auth: When access_token expires
    App->>Auth: POST /auth/refresh { refresh_token }
    Auth-->>App: New access_token + refresh_token

    Staff->>App: Logout
    App->>Auth: POST /auth/logout
    Auth-->>App: 204 (refresh token revoked)
```

---

## 7. High-Level API Resource Map

How the main resource groups relate to each other at a glance.

```mermaid
flowchart LR
    subgraph Core
        Cust[Customers]
        Book[Bookings]
        Pay[Payments]
    end
    subgraph Catalog
        Prod[Products]
        Avail[Availability]
    end
    subgraph Ops
        Rep[Reports]
        Set[Settings]
        Usr[Users]
    end

    Cust -->|customer_id| Book
    Book -->|booking_id| Pay
    Book -->|product_code line_items| Prod
    Prod -->|product_code| Avail
    Book -->|feeds| Rep
    Pay -->|feeds| Rep
    Set -.->|stage list, buffer days| Book
    Set -.->|product_code prefix| Prod
    Usr -->|customer_by / receipt_by| Book
```
