
# Backend Requirement Specifications

> Version: 1.0 — June 29 2025

This document defines detailed backend requirements for three core feature areas of the property‑rental platform: **User Authentication**, **Property Management**, and **Booking System**. Each feature is described in terms of REST API endpoints, request/response contracts, validation rules, security considerations, and performance targets.

---

## 1  User Authentication

### 1.1  Overview

Handles secure user registration, login, logout, password resets, and token refresh using JSON Web Tokens (JWT). All endpoints reside under `/api/v1/auth` and respond with `application/json`.

### 1.2  Endpoints

| # | Method | Path               | Purpose                                                   |
| - | ------ | ------------------ | --------------------------------------------------------- |
| 1 | `POST` | `/register`        | Create a new user account and send verification email.    |
| 2 | `POST` | `/login`           | Authenticate credentials and issue access+refresh tokens. |
| 3 | `POST` | `/logout`          | Invalidate refresh token (server‑side blacklist).         |
| 4 | `POST` | `/refresh`         | Issue a new access token using a valid refresh token.     |
| 5 | `POST` | `/forgot-password` | Send password‑reset email with one‑time token.            |
| 6 | `POST` | `/reset-password`  | Set a new password via valid reset token.                 |

#### 1.2.1  `POST /register`

```jsonc
Request Body
{
  "fullName": "Alice Johnson",          // string, 3–100 chars
  "email": "alice@example.com",         // valid RFC 5322 address
  "password": "P@ssw0rd123",            // min 8 chars, 1 upper, lower, digit, symbol
  "phone": "+14155550101"               // E.164 optional
}
```

*Validation*

* `email` must be unique (case‑insensitive) across users.
* `password` meets complexity regex `^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)(?=.*[^A-Za-z\d]).{8,}$`.

*Responses*

* **201** Created

  ```json
  { "userId": 1, "message": "Verification email sent" }
  ```
* **400** Validation error list.
* **409** Email already registered.

*(Full request/response examples for other endpoints omitted for brevity but follow the same template.)*

### 1.3  Security & Validation Rules

* Passwords hashed with Argon2id, 16‑byte salt, 4 iterations.
* JWT access token TTL = 15 min; refresh token TTL = 7 days.
* All auth endpoints rate‑limited to **5 requests/min per IP**.
* CSRF protection on cookie‑based tokens (SameSite=Lax).

### 1.4  Performance Criteria

* **P95** latency ≤ 200 ms for `POST /login` under 1 000 RPS.
* Token validation must stay CPU‑bound; target 2 million verifications/min on a 4‑core pod.

---

## 2  Property Management

### 2.1  Overview

Hosts can create, update, list, and archive property listings. Guests can search and view public listings. All endpoints under `/api/v1/properties`.

### 2.2  Endpoints

| # | Method   | Path           | Auth?        | Purpose                |
| - | -------- | -------------- | ------------ | ---------------------- |
| 1 | `POST`   | `/`            | Host         | Create new listing     |
| 2 | `GET`    | `/`            | Public       | List/search listings   |
| 3 | `GET`    | `/:propertyId` | Public       | Fetch property details |
| 4 | `PATCH`  | `/:propertyId` | Host (owner) | Update listing         |
| 5 | `DELETE` | `/:propertyId` | Host (owner) | Soft delete (archive)  |

#### 2.2.1  `POST /`

```jsonc
{
  "title": "Cozy Loft in Accra",         // 5–150 chars
  "description": "...",                  // markdown, ≤ 3 000 chars
  "location": "Accra, Ghana",           // free text or geohash
  "nightlyRate": 120.00,                 // numeric, ≥ 0
  "photos": ["https://cdn…/1.jpg", …]    // ≤ 10 URLs
}
```

*Validation*

* Authenticated user must have role `HOST`.
* `title` unique **per host**.
* `photos` array length ≤ 10; each URL must be HTTPS.

*Performance*

* Listing search (`GET /`) supports pagination (`page`, `limit≤100`), text search (`q`), and geo bounding box.
  **P95** response ≤ 250 ms for a paginated query of 50 results at 2 000 QPS with index coverage on `(location, nightlyRate)`.

---

## 3  Booking System

### 3.1  Overview

Enables guests to check availability, create reservations, cancel them, and list their bookings. All operations under `/api/v1/bookings`.

### 3.2  Endpoints

| # | Method   | Path            | Auth?         | Purpose                 |
| - | -------- | --------------- | ------------- | ----------------------- |
| 1 | `POST`   | `/availability` | Public        | Check if dates are free |
| 2 | `POST`   | `/`             | Guest         | Create booking          |
| 3 | `GET`    | `/`             | Guest         | List own bookings       |
| 4 | `GET`    | `/:bookingId`   | Guest or Host | Booking details         |
| 5 | `DELETE` | `/:bookingId`   | Guest         | Cancel (if allowed)     |

#### 3.2.1  `POST /availability`

```jsonc
{
  "propertyId": 42,
  "checkIn":  "2025-07-10",
  "checkOut": "2025-07-15"
}
```

*Response*

```json
{ "available": true }
```

*Validation*

* `checkOut` > `checkIn`; stay length ≤ 30 days.
* Reject if overlapping booking exists: `(propertyId, date_range)` conflict.

#### 3.2.2  `POST /`

Creates booking and returns payment intent.

```jsonc
{
  "propertyId": 42,
  "checkIn":  "2025-07-10",
  "checkOut": "2025-07-15",
  "guests": 2
}
```

*Response 201*

```json
{
  "bookingId": 11,
  "amount": 600.00,
  "currency": "USD",
  "paymentIntent": "pi_abc123"   // from payment gateway
}
```

*Validation*

* Guest cannot book own property.
* Payment must be completed within **15 minutes** or booking auto‑expires (cron job).

### 3.3  Consistency & Concurrency

* Use **SERIALIZABLE** isolation for `POST /` with retry logic (max 3 attempts) to avoid race conditions.
* Idempotency‑Key header supported to safely retry booking requests.

### 3.4  Performance Criteria

* Availability checks cached (Redis) for 5 minutes; **P99** latency ≤ 150 ms.
* Bookings creation throughput target: **500 TPS** globally with <1 s end‑to‑end including payment‑intent creation.

---

## 4  Cross‑Cutting Non‑Functional Requirements

| Area              | Requirement                                                                                       |
| ----------------- | ------------------------------------------------------------------------------------------------- |
| **Security**      | OWASP Top 10 compliance, HSTS (1 year, includeSubDomains), TLS 1.3 only                           |
| **Observability** | Structured JSON logs, OpenTelemetry traces, Prometheus metrics (`http_request_duration_seconds`)  |
| **Scalability**   | Stateless services with horizontal autoscaling pods; database read replicas for heavy read routes |
| **Rate Limiting** | 100 req/min per IP for public endpoints, 1 000 req/min per authenticated user                     |
| **Backup & DR**   | Point‑in‑time recovery (≤ 15 min RPO), multi‑AZ replicas                                          |

---

*End of Document*
