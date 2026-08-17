# Rental Outfit CRM — API Documentation

| | |
|---|---|
| **Version** | v1.0 |
| **Base URL** | `https://api.rentaloutfitcrm.com/v1` |
| **Protocol** | HTTPS only |
| **Format** | JSON (`Content-Type: application/json`) |
| **Auth** | Bearer token (JWT) |
| **Last Updated** | 17 August 2026 |

---

## 1. Conventions

### 1.1 Authentication

All endpoints except `POST /auth/login` require an `Authorization` header:

```
Authorization: Bearer <access_token>
```

Tokens are issued via `POST /auth/login` and refreshed via `POST /auth/refresh`. Access tokens expire in 60 minutes; refresh tokens expire in 30 days.

### 1.2 Roles & Permissions

| Role | Access |
|---|---|
| `admin` | Full access to all endpoints, including Settings and Reports |
| `manager` | Full access to Bookings, Customers, Products; can override booking conflicts; read-only Reports |
| `staff` | Create/read/update Bookings and Customers; read-only Products; cannot delete records or edit pricing |

Endpoints below list the minimum role required under **Auth**.

### 1.3 Standard Response Envelope

**Success:**
```json
{
  "success": true,
  "data": { },
  "meta": { }
}
```

**Error:**
```json
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Return Date cannot be before Pick Up Date.",
    "details": [
      { "field": "return_date", "issue": "must be >= pick_up_date" }
    ]
  }
}
```

### 1.4 Standard HTTP Status Codes

| Code | Meaning |
|---|---|
| `200 OK` | Request succeeded |
| `201 Created` | Resource created |
| `204 No Content` | Request succeeded, no body returned (e.g. delete) |
| `400 Bad Request` | Validation error / malformed request |
| `401 Unauthorized` | Missing/invalid/expired token |
| `403 Forbidden` | Authenticated but insufficient role |
| `404 Not Found` | Resource does not exist |
| `409 Conflict` | Business rule conflict (e.g. product already booked for the date range) |
| `422 Unprocessable Entity` | Semantically invalid data (e.g. discount exceeds Max Discount) |
| `429 Too Many Requests` | Rate limit exceeded |
| `500 Internal Server Error` | Unexpected server error |

### 1.5 Pagination

List endpoints accept:

| Param | Type | Default | Notes |
|---|---|---|---|
| `page` | integer | `1` | 1-indexed |
| `limit` | integer | `25` | Max `100` |
| `sort` | string | endpoint-specific | e.g. `-created_on` (prefix `-` = descending) |

Paginated responses include:
```json
"meta": {
  "page": 1,
  "limit": 25,
  "total_records": 342,
  "total_pages": 14
}
```

### 1.6 Filtering & Search

List endpoints accept filter query params specific to that resource (documented per-endpoint) plus a generic:

- `q` — free-text search across key fields (e.g. Name, Phone Number, Receipt Number, Product Code)

### 1.7 Rate Limiting

429 responses include headers:
```
X-RateLimit-Limit: 120
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1755423600
```
Default limit: 120 requests/minute per token.

### 1.8 Idempotency

`POST` endpoints that create financial records (e.g. Payments) accept an optional header:
```
Idempotency-Key: <client-generated-uuid>
```
Replaying the same key returns the original response instead of creating a duplicate.

---

## 2. Auth

### 2.1 `POST /auth/login`
**Auth:** None

**Description:** Authenticates a staff user and returns access/refresh tokens.

**Request Body:**
```json
{
  "email": "staff@shop.com",
  "password": "••••••••"
}
```

**Response `200`:**
```json
{
  "success": true,
  "data": {
    "access_token": "eyJhbGciOi...",
    "refresh_token": "d3f1c9a2...",
    "expires_in": 3600,
    "user": {
      "id": "usr_1029",
      "name": "Priya Sharma",
      "role": "staff"
    }
  }
}
```

**Errors:** `401` invalid credentials, `429` too many login attempts.

---

### 2.2 `POST /auth/refresh`
**Auth:** None (requires valid refresh token in body)

**Request Body:**
```json
{ "refresh_token": "d3f1c9a2..." }
```

**Response `200`:** same shape as `2.1`.

---

### 2.3 `POST /auth/logout`
**Auth:** Any authenticated role

**Description:** Revokes the current refresh token.

**Response `204`:** empty body.

---

## 3. Customers

Customer is a deduplicated entity (see PRD §9), keyed by `phone_number` / `id_card_number`, distinct from the transactional Booking record.

### 3.1 `GET /customers`
**Auth:** staff+

**Query Params:**

| Param | Type | Description |
|---|---|---|
| `q` | string | Search by name or phone number |
| `source` | string | Filter by lead Source (e.g. `Instagram`, `Walk-in`) |
| `page`, `limit`, `sort` | — | See §1.5 |

**Response `200`:**
```json
{
  "success": true,
  "data": [
    {
      "id": "cust_5001",
      "name": "Ananya Rao",
      "phone_number": "9876543210",
      "id_card_type": "Aadhaar",
      "id_card_number": "XXXX-XXXX-4321",
      "source": "Instagram",
      "created_on": "2026-06-12T10:15:00Z",
      "total_bookings": 3
    }
  ],
  "meta": { "page": 1, "limit": 25, "total_records": 1, "total_pages": 1 }
}
```

---

### 3.2 `POST /customers`
**Auth:** staff+

**Description:** Creates a new customer. If `phone_number` or `id_card_number` matches an existing customer, returns `409` with the existing record referenced, so staff can link the booking to the existing customer instead of duplicating.

**Request Body:**
```json
{
  "name": "Ananya Rao",
  "phone_number": "9876543210",
  "id_card_type": "Aadhaar",
  "id_card_number": "XXXX-XXXX-4321",
  "source": "Instagram"
}
```

| Field | Type | Required | Notes |
|---|---|---|---|
| `name` | string | Yes | |
| `phone_number` | string | Yes | 10-digit, validated |
| `id_card_type` | enum | No | `Aadhaar`, `PAN`, `Driving License`, `Passport`, `Voter ID` |
| `id_card_number` | string | No | |
| `source` | enum | No | `Walk-in`, `Instagram`, `Facebook`, `Referral`, `Google`, `Other` |

**Response `201`:**
```json
{ "success": true, "data": { "id": "cust_5001", "name": "Ananya Rao", "created_on": "2026-08-17T09:00:00Z" } }
```

**Response `409` (duplicate found):**
```json
{
  "success": false,
  "error": {
    "code": "DUPLICATE_CUSTOMER",
    "message": "A customer with this phone number already exists.",
    "details": [{ "existing_customer_id": "cust_5001" }]
  }
}
```

---

### 3.3 `GET /customers/{customer_id}`
**Auth:** staff+

**Response `200`:** full customer object including a `bookings` summary array (id, receipt_number, stage, booking_date).

---

### 3.4 `PATCH /customers/{customer_id}`
**Auth:** staff+ (own edits) / manager+ (ID card fields)

**Request Body:** any subset of fields from §3.2.

**Response `200`:** updated customer object.

---

### 3.5 `GET /customers/{customer_id}/bookings`
**Auth:** staff+

**Description:** Returns the full booking history for a single customer, across all dates. A customer is not limited to one booking — this list can include multiple bookings on different days, or multiple bookings on the same day (e.g. two separate events booked in one visit), each as its own record with its own `receipt_number`. To combine multiple products into a single receipt for one event, use `line_items[]` on one booking (§4.2) instead of creating separate bookings.

**Query Params:**

| Param | Type | Description |
|---|---|---|
| `stage` | enum | Filter by Stage |
| `from_date`, `to_date` | date | Same overlap semantics as §4.1 |
| `page`, `limit`, `sort` | — | See §1.5. Default sort: `-booking_date` |

**Response `200`:** array of booking summary objects (same shape as §4.1 list items).

```json
{
  "success": true,
  "data": [
    {
      "id": "bkg_9001",
      "receipt_number": "RCPT-2026-0341",
      "stage": "Booking Confirmed",
      "booking_date": "2026-08-17",
      "pick_up_date": "2026-09-12",
      "return_date": "2026-09-15",
      "total_amount": 11600.00,
      "balance": 6600.00,
      "payment_status": "Partially Paid"
    },
    {
      "id": "bkg_9004",
      "receipt_number": "RCPT-2026-0342",
      "stage": "Enquiry",
      "booking_date": "2026-08-17",
      "pick_up_date": "2026-10-02",
      "return_date": "2026-10-04",
      "total_amount": 6200.00,
      "balance": 6200.00,
      "payment_status": "Unpaid"
    }
  ],
  "meta": { "page": 1, "limit": 25, "total_records": 2, "total_pages": 1 }
}
```

> Note the two bookings above share the same `booking_date` (created same day) but are entirely independent transactions with different receipt numbers, products, and date ranges — this is the normal pattern for a repeat or same-day multi-event customer.

---

### 3.6 `DELETE /customers/{customer_id}`
**Auth:** admin

**Description:** Soft-deletes a customer (blocked if active bookings exist — returns `409`).

**Response `204`.**

---

## 4. Bookings

Booking is the core transactional record (PRD §7.1). Supports multi-product line items per booking (PRD §7.1.1).

### 4.1 `GET /bookings`
**Auth:** staff+

**Query Params:**

| Param | Type | Description |
|---|---|---|
| `stage` | enum | `Enquiry`, `Booking Confirmed`, `Pickup Pending`, `Ready for Pickup`, `Pick Up Done`, `Return Pending`, `Return Received/Deposit Pending`, `Successful Leads`, `Postponed/Credit Note`, `Cancelled/Full Refund` |
| `payment_status` | enum | `Unpaid`, `Partially Paid`, `Paid` |
| `from_date`, `to_date` | date | Single date-range filter. Returns bookings whose rental period overlaps this window — i.e. `pick_up_date <= to_date AND return_date >= from_date`. Pass only `from_date` for "on/after," only `to_date` for "on/before." |
| `source` | enum | Lead source |
| `q` | string | Search by customer name, phone, receipt number, product code |
| `page`, `limit`, `sort` | — | See §1.5. Default sort: `-booking_date` |

**Response `200`:**
```json
{
  "success": true,
  "data": [
    {
      "id": "bkg_9001",
      "receipt_number": "RCPT-2026-0341",
      "customer": { "id": "cust_5001", "name": "Ananya Rao", "phone_number": "9876543210" },
      "stage": "Booking Confirmed",
      "created_on": "2026-08-17T09:05:00Z",
      "booking_date": "2026-08-17",
      "pick_up_date": "2026-09-12",
      "return_date": "2026-09-15",
      "follow_up": "2026-09-01",
      "total_amount": 12500.00,
      "amount_paid": 5000.00,
      "balance": 7500.00,
      "payment_status": "Partially Paid",
      "source": "Instagram",
      "line_items_count": 1
    }
  ],
  "meta": { "page": 1, "limit": 25, "total_records": 1, "total_pages": 1 }
}
```

---

### 4.2 `POST /bookings`
**Auth:** staff+

**Description:** Creates a booking. For each line item, the server performs a **hard availability check** (PRD §7.3.3) before allowing `stage` to be set to `Booking Confirmed` or beyond; if any product conflicts, the request fails with `409` unless the caller has `manager`/`admin` role and passes `"override_conflict": true`.

**Request Body:**
```json
{
  "customer_id": "cust_5001",
  "booking_date": "2026-08-17",
  "pick_up_date": "2026-09-12",
  "return_date": "2026-09-15",
  "follow_up": "2026-09-01",
  "special_note": "Bride requested extra dupatta pins",
  "customer_by": "usr_1029",
  "receipt_by": "usr_1029",
  "source": "Instagram",
  "line_items": [
    {
      "product_code": "LHG-0042",
      "alterations": "Waist +2in, sleeve -1in",
      "rent_amount": 8000.00,
      "deposit_amount": 4000.00,
      "discount_percent": 5
    }
  ],
  "override_conflict": false
}
```

| Field | Type | Required | Notes |
|---|---|---|---|
| `customer_id` | string | Yes | Must reference an existing customer |
| `booking_date` | date | Yes | Defaults to today if omitted |
| `pick_up_date` | date | Yes | |
| `return_date` | date | Yes | Must be ≥ `pick_up_date` |
| `follow_up` | date | No | |
| `special_note` | string | No | |
| `customer_by` / `receipt_by` | string (user id) | Yes | |
| `source` | enum | No | |
| `line_items` | array | Yes, min 1 | See §4.2.1 |
| `override_conflict` | boolean | No | manager+/admin only; bypasses §7.3.3 hard block |

**§4.2.1 Line Item Object**

| Field | Type | Required | Notes |
|---|---|---|---|
| `product_code` | string | Yes | References Product Catalog |
| `alterations` | string | No | |
| `rent_amount` | number | No | Defaults to Product's `rent_amount` if omitted; editable per booking |
| `deposit_amount` | number | No | Defaults to Product's `deposit_amount` |
| `discount_percent` | number | No | Capped server-side at Product's `max_discount`; `422` if exceeded |

**Server-calculated fields (not accepted in request):** `total_rent`, `total_deposit`, `total_amount`, `balance`, `payment_status`, `receipt_number` (auto-generated), `stage` (defaults to `Enquiry`).

**Response `201`:**
```json
{
  "success": true,
  "data": {
    "id": "bkg_9001",
    "receipt_number": "RCPT-2026-0341",
    "stage": "Enquiry",
    "total_rent": 8000.00,
    "total_deposit": 4000.00,
    "total_amount": 11600.00,
    "amount_paid": 0,
    "balance": 11600.00,
    "payment_status": "Unpaid"
  }
}
```

**Response `409` (availability conflict):**
```json
{
  "success": false,
  "error": {
    "code": "PRODUCT_UNAVAILABLE",
    "message": "Product LHG-0042 is already booked for an overlapping date range.",
    "details": [
      {
        "product_code": "LHG-0042",
        "conflicting_booking": {
          "id": "bkg_8890",
          "receipt_number": "RCPT-2026-0299",
          "customer_name": "Meera Iyer",
          "pick_up_date": "2026-09-14",
          "return_date": "2026-09-18"
        }
      }
    ]
  }
}
```

---

### 4.3 `GET /bookings/{booking_id}`
**Auth:** staff+

**Response `200`:** full booking object including `line_items[]`, `payments[]` (see §5), and `audit_log[]` (user, action, timestamp).

---

### 4.4 `PATCH /bookings/{booking_id}`
**Auth:** staff+ (most fields) / manager+ (stage → Booking Confirmed with conflict override; pricing overrides beyond max discount)

**Description:** Partial update. Common uses: advancing `stage`, editing `special_note`/`alterations`, adjusting dates (re-triggers availability check).

**Request Body (example — advancing stage):**
```json
{ "stage": "Ready for Pickup" }
```

**Response `200`:** updated booking object.

**Response `409`:** if changing `pick_up_date`/`return_date` creates a new conflict.

---

### 4.5 `DELETE /bookings/{booking_id}`
**Auth:** manager+

**Description:** Not a hard delete — sets `stage` to `Cancelled/Full Refund`, which releases the product's date range back to available (PRD §9). Use `PATCH .../cancel` (§4.6) in preference; this endpoint is retained for admin cleanup only.

**Response `204`.**

---

### 4.6 `POST /bookings/{booking_id}/cancel`
**Auth:** staff+

**Request Body:**
```json
{ "reason": "Customer changed event date" }
```

**Response `200`:** booking object with `stage: "Cancelled/Full Refund"`. All line-item date ranges are immediately excluded from availability conflict checks.

> **Note:** if the customer is postponing rather than cancelling outright, use `PATCH /bookings/{booking_id}` with `{ "stage": "Postponed/Credit Note" }` instead — this also releases the date range but issues a credit note rather than a refund (see §12 error table and diagrams doc §4 for the full stage lifecycle).

---

## 5. Payments

Payments are sub-records of a Booking, supporting multiple partial payments (advance + balance at pickup, etc.) per PRD §8.

### 5.1 `GET /bookings/{booking_id}/payments`
**Auth:** staff+

**Response `200`:**
```json
{
  "success": true,
  "data": [
    {
      "id": "pay_3001",
      "amount": 5000.00,
      "paid_via": "UPI",
      "upi_name": "Ananya Rao",
      "paid_on": "2026-08-17T09:10:00Z",
      "recorded_by": "usr_1029"
    }
  ]
}
```

---

### 5.2 `POST /bookings/{booking_id}/payments`
**Auth:** staff+

**Description:** Records a new payment against a booking. Server recalculates `amount_paid`, `balance`, and `payment_status` on the parent booking automatically.

**Headers:** `Idempotency-Key` recommended (see §1.8).

**Request Body:**
```json
{
  "amount": 5000.00,
  "paid_via": "UPI",
  "upi_name": "Ananya Rao"
}
```

| Field | Type | Required | Notes |
|---|---|---|---|
| `amount` | number | Yes | Must be > 0; server rejects if it would push `amount_paid` above `total_amount` (returns `422` unless `allow_overpayment: true` is passed, e.g. for deposit adjustments) |
| `paid_via` | enum | Yes | `Cash`, `UPI`, `Card`, `Bank Transfer` |
| `upi_name` | string | Conditional | Required if `paid_via = UPI` |

**Response `201`:**
```json
{
  "success": true,
  "data": {
    "payment": { "id": "pay_3001", "amount": 5000.00, "paid_via": "UPI" },
    "booking_summary": {
      "total_amount": 11600.00,
      "amount_paid": 5000.00,
      "balance": 6600.00,
      "payment_status": "Partially Paid"
    }
  }
}
```

---

### 5.3 `DELETE /bookings/{booking_id}/payments/{payment_id}`
**Auth:** manager+

**Description:** Reverses/voids a payment entry (e.g. entered in error). Recalculates booking balance.

**Response `204`.**

---

## 6. Products (Catalog)

### 6.1 `GET /products`
**Auth:** staff+ (read) 

**Query Params:**

| Param | Type | Description |
|---|---|---|
| `category` | string | Filter by Category |
| `sub_category` | string | Filter by Sub Category |
| `brand` | string | Filter by Brand |
| `status` | enum | `Active`, `Under Maintenance`, `Retired`, `Lost/Damaged` |
| `q` | string | Search by Name or Product Code |
| `page`, `limit`, `sort` | — | See §1.5 |

**Response `200`:**
```json
{
  "success": true,
  "data": [
    {
      "id": "prod_7001",
      "product_code": "LHG-0042",
      "name": "Red Silk Bridal Lehenga",
      "brand": "Manish Malhotra Studio",
      "category": "Lehenga",
      "sub_category": "Bridal",
      "product_price": 45000.00,
      "discount_on_mrp_percent": 10,
      "rent_amount": 8000.00,
      "deposit_amount": 4000.00,
      "unit_tax_percent": 5,
      "max_discount_percent": 15,
      "status": "Active",
      "size": "M",
      "image_url": "https://cdn.rentaloutfitcrm.com/products/prod_7001.jpg"
    }
  ],
  "meta": { "page": 1, "limit": 25, "total_records": 1, "total_pages": 1 }
}
```

---

### 6.2 `POST /products`
**Auth:** manager+

**Request Body:**
```json
{
  "name": "Red Silk Bridal Lehenga",
  "brand": "Manish Malhotra Studio",
  "category": "Lehenga",
  "sub_category": "Bridal",
  "product_price": 45000.00,
  "discount_on_mrp_percent": 10,
  "rent_amount": 8000.00,
  "deposit_amount": 4000.00,
  "unit_tax_percent": 5,
  "max_discount_percent": 15,
  "description": "Fabric: Silk. Color: Red/Gold. Includes dupatta and belt.",
  "size": "M",
  "product_code": null
}
```

| Field | Type | Required | Notes |
|---|---|---|---|
| `name` | string | Yes | |
| `brand` | string | No | |
| `category` | string | Yes | |
| `sub_category` | string | No | |
| `product_price` | number | No | Reference MRP/retail value |
| `discount_on_mrp_percent` | number | No | |
| `rent_amount` | number | Yes | |
| `deposit_amount` | number | Yes | |
| `unit_tax_percent` | number | No | |
| `max_discount_percent` | number | No | Ceiling enforced at booking time |
| `description` | string | No | |
| `size` | string | No | |
| `product_code` | string | No | Auto-generated with configurable prefix if omitted (e.g. `LHG-0043`) |

**Response `201`:** created product object (as §6.1 item shape).

**Response `422`:** if `product_code` supplied is not unique.

---

### 6.3 `GET /products/{product_id}`
**Auth:** staff+

**Response `200`:** full product object, plus a `booking_history_summary` (total times rented, last rented date).

---

### 6.4 `PATCH /products/{product_id}`
**Auth:** manager+

**Description:** Updates catalog master data. Note: does **not** retroactively change pricing on existing bookings — bookings store a pricing snapshot at time of creation (PRD §9).

**Response `200`:** updated product object.

---

### 6.5 `DELETE /products/{product_id}`
**Auth:** admin

**Description:** Blocked (`409`) if the product has any active bookings — i.e. any stage other than `Return Received/Deposit Pending`, `Successful Leads`, `Postponed/Credit Note`, or `Cancelled/Full Refund`. Recommend using `PATCH` to set `status: "Retired"` instead.

**Response `204`.**

---

### 6.6 `POST /products/import`
**Auth:** manager+

**Description:** Bulk import/update via CSV upload (`multipart/form-data`), per PRD §7.2.

**Request:** `Content-Type: multipart/form-data`, field `file` = CSV.

**Response `200`:**
```json
{
  "success": true,
  "data": {
    "rows_processed": 120,
    "rows_created": 95,
    "rows_updated": 20,
    "rows_failed": 5,
    "errors": [
      { "row": 34, "issue": "Duplicate product_code: LHG-0042" }
    ]
  }
}
```

---

## 7. Product Availability

Implements the availability logic defined in PRD §7.3.

### 7.1 `GET /availability/check`
**Auth:** staff+

**Description:** Checks whether a specific product (or all products in a category) is free for a given date range. This is the endpoint called inline during the New Booking flow (§4.2) before allowing `Booking Confirmed` stage.

**Query Params:**

| Param | Type | Required | Notes |
|---|---|---|---|
| `product_code` | string | Yes (or `category`) | Specific unit to check |
| `category` | string | No | If provided instead of `product_code`, returns availability across all products in that category |
| `pick_up_date` | date | Yes | |
| `return_date` | date | Yes | Must be ≥ `pick_up_date` |
| `buffer_days` | integer | No | Overrides the global turnaround buffer (default from Settings) |

**Example:** `GET /availability/check?product_code=LHG-0042&pick_up_date=2026-09-12&return_date=2026-09-15`

**Response `200` (available):**
```json
{
  "success": true,
  "data": {
    "product_code": "LHG-0042",
    "requested_range": { "pick_up_date": "2026-09-12", "return_date": "2026-09-15" },
    "available": true,
    "conflicts": []
  }
}
```

**Response `200` (not available):**
```json
{
  "success": true,
  "data": {
    "product_code": "LHG-0042",
    "requested_range": { "pick_up_date": "2026-09-12", "return_date": "2026-09-15" },
    "available": false,
    "conflicts": [
      {
        "booking_id": "bkg_8890",
        "receipt_number": "RCPT-2026-0299",
        "customer_name": "Meera Iyer",
        "pick_up_date": "2026-09-14",
        "return_date": "2026-09-18",
        "stage": "Booking Confirmed"
      }
    ],
    "next_available_date": "2026-09-19"
  }
}
```

> Bookings with `stage` in `Return Received/Deposit Pending`, `Successful Leads`, `Postponed/Credit Note`, or `Cancelled/Full Refund` are excluded from conflict evaluation, per the availability logic in PRD §7.3.3. All other stages (`Enquiry`, `Booking Confirmed`, `Pickup Pending`, `Ready for Pickup`, `Pick Up Done`, `Return Pending`) are treated as active holds on the product. See the diagrams doc §5 for the full decision logic, including the open question on whether `Enquiry` should block.

---

### 7.2 `GET /availability/calendar`
**Auth:** staff+

**Description:** Returns a timeline of all booked date ranges for a product (or category) within a date window, powering the Gantt-style calendar view (PRD §7.3.2).

**Query Params:**

| Param | Type | Required | Notes |
|---|---|---|---|
| `product_code` | string | Yes (or `category`) | |
| `from` | date | Yes | Window start |
| `to` | date | Yes | Window end |

**Response `200`:**
```json
{
  "success": true,
  "data": {
    "product_code": "LHG-0042",
    "window": { "from": "2026-09-01", "to": "2026-09-30" },
    "bookings": [
      {
        "booking_id": "bkg_8890",
        "receipt_number": "RCPT-2026-0299",
        "pick_up_date": "2026-09-14",
        "return_date": "2026-09-18",
        "stage": "Booking Confirmed"
      },
      {
        "booking_id": "bkg_9002",
        "receipt_number": "RCPT-2026-0350",
        "pick_up_date": "2026-09-22",
        "return_date": "2026-09-25",
        "stage": "Ready for Pickup"
      }
    ]
  }
}
```

---

## 8. Reports

### 8.1 `GET /reports/revenue`
**Auth:** manager+

**Query Params:** `from` (date), `to` (date), `group_by` (`day`|`week`|`month`|`category`|`source`)

**Response `200`:**
```json
{
  "success": true,
  "data": {
    "group_by": "month",
    "series": [
      { "period": "2026-07", "revenue": 245000.00, "bookings_count": 38 },
      { "period": "2026-08", "revenue": 312500.00, "bookings_count": 46 }
    ]
  }
}
```

---

### 8.2 `GET /reports/balance-due`
**Auth:** manager+

**Description:** Aged balance-due report (PRD §7.4).

**Query Params:** `bucket` (optional filter: `0-7`, `8-30`, `30+`)

**Response `200`:**
```json
{
  "success": true,
  "data": [
    {
      "booking_id": "bkg_9001",
      "receipt_number": "RCPT-2026-0341",
      "customer_name": "Ananya Rao",
      "balance": 6600.00,
      "days_outstanding": 5,
      "aged_bucket": "0-7"
    }
  ],
  "meta": { "total_outstanding": 6600.00 }
}
```

---

### 8.3 `GET /reports/follow-ups`
**Auth:** staff+

**Query Params:** `range` (`today`|`this_week`|`overdue`)

**Response `200`:** array of bookings with `follow_up` date in range, same shape as §4.1 list items.

---

### 8.4 `GET /reports/product-performance`
**Auth:** manager+

**Query Params:** `from`, `to`, `order` (`most_booked`|`least_booked`), `limit`

**Response `200`:**
```json
{
  "success": true,
  "data": [
    { "product_code": "LHG-0042", "name": "Red Silk Bridal Lehenga", "times_booked": 14, "revenue_generated": 96000.00 }
  ]
}
```

---

## 9. Settings

### 9.1 `GET /settings`
**Auth:** admin

**Response `200`:**
```json
{
  "success": true,
  "data": {
    "stage_pipeline": ["Enquiry", "Booking Confirmed", "Pickup Pending", "Ready for Pickup", "Pick Up Done", "Return Pending", "Return Received/Deposit Pending", "Successful Leads", "Postponed/Credit Note", "Cancelled/Full Refund"],
    "sources": ["Walk-in", "Instagram", "Facebook", "Referral", "Google", "Other"],
    "availability_buffer_days": 1,
    "product_code_prefix_by_category": { "Lehenga": "LHG", "Sherwani": "SHW" }
  }
}
```

### 9.2 `PATCH /settings`
**Auth:** admin

**Request Body:** any subset of the fields in §9.1.

**Response `200`:** updated settings object.

---

## 10. Users (Staff Accounts)

### 10.1 `GET /users`
**Auth:** admin

**Response `200`:** list of `{ id, name, email, role, active }`.

### 10.2 `POST /users`
**Auth:** admin

**Request Body:**
```json
{ "name": "Priya Sharma", "email": "priya@shop.com", "role": "staff" }
```
**Response `201`:** created user (invite email triggered to set password).

### 10.3 `PATCH /users/{user_id}`
**Auth:** admin

**Request Body:** e.g. `{ "role": "manager", "active": false }`

**Response `200`:** updated user object.

---

## 11. Webhooks (optional / Phase 2)

For integrations (e.g. WhatsApp/SMS reminders per PRD Roadmap Phase 3), the API can emit events to a configured endpoint:

| Event | Trigger |
|---|---|
| `booking.created` | New booking created |
| `booking.stage_changed` | Stage transitions (e.g. → `Ready for Pickup`, → `Return Received/Deposit Pending`) |
| `payment.recorded` | New payment recorded |
| `booking.balance_due_reminder` | Follow-up date reached with balance > 0 |

**Payload shape:**
```json
{
  "event": "booking.stage_changed",
  "booking_id": "bkg_9001",
  "data": { "from_stage": "Pickup Pending", "to_stage": "Ready for Pickup" },
  "timestamp": "2026-08-17T09:20:00Z"
}
```

---

## 12. Error Code Reference

| Code | HTTP Status | Meaning |
|---|---|---|
| `VALIDATION_ERROR` | 400 | Request body failed field-level validation |
| `UNAUTHORIZED` | 401 | Missing/invalid/expired token |
| `FORBIDDEN` | 403 | Role lacks permission for this action |
| `NOT_FOUND` | 404 | Resource does not exist |
| `DUPLICATE_CUSTOMER` | 409 | Matching phone/ID card number already exists |
| `PRODUCT_UNAVAILABLE` | 409 | Date range conflicts with an existing active booking |
| `HAS_ACTIVE_BOOKINGS` | 409 | Attempted delete on a product/customer with active bookings |
| `DISCOUNT_EXCEEDS_MAX` | 422 | `discount_percent` exceeds product's `max_discount_percent` |
| `OVERPAYMENT_NOT_ALLOWED` | 422 | Payment would exceed `total_amount` without `allow_overpayment` |
| `RATE_LIMITED` | 429 | Too many requests |
| `SERVER_ERROR` | 500 | Unexpected failure |

---

## 13. Status & Enum Reference

Single source of truth for every enum used across the API.

**Booking `stage`** (pipeline, ordered):
1. `Enquiry`
2. `Booking Confirmed`
3. `Pickup Pending`
4. `Ready for Pickup`
5. `Pick Up Done`
6. `Return Pending`
7. `Return Received/Deposit Pending`
8. `Successful Leads`
9. `Postponed/Credit Note` — exit branch, releases the product; customer retains a credit note rather than a refund
10. `Cancelled/Full Refund` — exit branch, releases the product; full refund issued

See the diagrams document (`Rental-Outfit-CRM-API-Diagrams.md`, §4) for the full state transition diagram.

**`payment_status`** (auto-calculated, never set directly): `Unpaid` / `Partially Paid` / `Paid`

**Product `status`**: `Active` / `Under Maintenance` / `Retired` / `Lost/Damaged`

**`paid_via`**: `Cash` / `UPI` / `Card` / `Bank Transfer`

**`source`**: `Walk-in` / `Instagram` / `Facebook` / `Referral` / `Google` / `Other`

**`id_card_type`**: `Aadhaar` / `PAN` / `Driving License` / `Passport` / `Voter ID`

**User `role`**: `admin` / `manager` / `staff`

---

## 14. Field-Level Business Rules (Quick Reference)

| Rule | Enforced At |
|---|---|
| `balance = total_amount - amount_paid` | Server-calculated, read-only in all responses |
| `payment_status` derived from `balance` (`Unpaid` / `Partially Paid` / `Paid`) | Server-calculated |
| `total_amount = (total_rent + total_deposit) + tax - discount` | Server-calculated on booking create/update |
| `return_date >= pick_up_date` | `POST/PATCH /bookings` |
| No overlapping active booking on same `product_code` | `POST /bookings` (stage → Booking Confirmed), `PATCH /bookings/{id}` on date change |
| `discount_percent <= product.max_discount_percent` | `POST/PATCH /bookings` line items |
| `Return Received/Deposit Pending`, `Successful Leads`, `Postponed/Credit Note`, `Cancelled/Full Refund` bookings excluded from availability conflicts | `GET /availability/check`, `GET /availability/calendar` |
| `product_code` and `receipt_number` unique | `POST /products`, auto-gen on `POST /bookings` |

---

## 15. Open Items for Engineering (carried from PRD)

1. Confirm whether `ID Card Type` and `Phone Number` are distinct fields (assumed yes in this spec).
2. Confirm unit-level Product Code strategy if the same design exists as multiple physical pieces (affects `GET /availability/check` semantics).
3. Deposit refund workflow is not yet modeled — no endpoint currently exists for partial deposit deduction/refund; to be added in Phase 2/3.
4. Confirm default `availability_buffer_days` value with the business (currently defaulted to `1`).
