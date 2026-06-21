<p align="center">
  <h1 align="center">🏨 StayEase</h1>
  <p align="center">
    <strong>Hospitality & Reservation Management Backend</strong>
    <br/>
    <sub>JWT Auth · Role-Based Access · Dynamic Pricing Engine · Stripe Payments · Revenue Analytics · PostgreSQL</sub>
  </p>
</p>

<p align="center">
  <img src="https://img.shields.io/badge/Java-17-ED8B00?style=flat&logo=openjdk&logoColor=white" />
  <img src="https://img.shields.io/badge/Spring%20Boot-3.x-6DB33F?style=flat&logo=springboot&logoColor=white" />
  <img src="https://img.shields.io/badge/PostgreSQL-336791?style=flat&logo=postgresql&logoColor=white" />
  <img src="https://img.shields.io/badge/Stripe-635BFF?style=flat&logo=stripe&logoColor=white" />
  <img src="https://img.shields.io/badge/JWT-Auth-black?style=flat&logo=jsonwebtokens" />
  <img src="https://img.shields.io/badge/License-Educational-blue?style=flat" />
</p>

<p align="center">
  <a href="#-architecture">Architecture</a> ·
  <a href="#-features">Features</a> ·
  <a href="#-api-reference">API Reference</a> ·
  <a href="#-quick-start">Quick Start</a> ·
  <a href="#-design-decisions">Design Decisions</a> ·
  <a href="#-known-gaps">Known Gaps</a>
</p>

---

A Spring Boot backend for a hotel booking platform — built to practise production patterns rather than ship to production. It covers the full lifecycle of a reservation: hotel and room setup, real-time availability search, multi-step booking with guest management, Stripe-powered checkout with webhook confirmation, automatic refunds on cancellation, and per-hotel revenue reporting.

The interesting engineering is in three areas: **pessimistic database locking** to prevent double-booking, a **decorator-chain pricing engine** recalculated hourly by a batch scheduler, and a **two-token JWT strategy** (short-lived access + long-lived refresh stored in HttpOnly cookie).

---

## 🏗 Architecture

```
src/main/java/com/codingshuttle/projects/airBnbApp/
│
├── advice/       → GlobalResponseHandler (ResponseBodyAdvice wraps all responses in ApiResponse<T>)
│                   GlobalExceptionHandler (ResourceNotFound, Auth, JWT, AccessDenied → consistent ApiError)
│
├── config/       → CorsConfig (localhost:3000), MapperConfig (ModelMapper bean), StripeConfig (SDK init)
│
├── controller/   → AuthController, HotelBookingController, HotelBrowseController, HotelController (admin),
│                   InventoryController (admin), RoomAdminController, UserController, WebhookController
│
├── dto/          → Request/response DTOs — entities never cross the service boundary
│
├── entity/       → Booking, Guest, Hotel, HotelContactInfo (@Embedded), HotelMinPrice,
│                   Inventory (unique: hotel_id + room_id + date), Room, User (implements UserDetails)
│   └── enums/    → BookingStatus, PaymentStatus, Role, Gender
│
├── exception/    → ResourceNotFoundException, UnAuthorisedException
│
├── repository/   → 8 repositories; InventoryRepository has 9 custom JPQL queries including
│                   bulk UPDATE, pessimistic locking, and a CASE-based average price query
│
├── security/     → JWTService, JWTAuthFilter (OncePerRequestFilter), AuthService, WebSecurityConfig
│
├── service/      → BookingServiceImpl, CheckoutServiceImpl, GuestServiceImpl, HotelServiceImpl,
│                   InventoryServiceImpl, PricingUpdateService (@Scheduled), RoomServiceImpl, UserServiceImpl
│
├── strategy/     → PricingStrategy interface + BasePricingStrategy, SurgePricingStrategy,
│                   OccupancyPricingStrategy, UrgencyPricingStrategy, HolidayPricingStrategy (stub)
│
└── util/         → AppUtils.getCurrentUser() — pulls authenticated User from SecurityContextHolder
```

The layering is strictly **Controller → Service Interface → ServiceImpl → Repository**. Every controller method delegates to an interface, keeping the controller layer free of business logic.

---

## ✨ Features

### 🔐 Authentication & Security

- **Two-token JWT strategy**: access token expires in 10 minutes; refresh token expires in 6 months. On login, both are issued — the access token is returned in the response body, the refresh token is written to an HttpOnly cookie.
- **Stateless session**: Spring Security is configured with `SessionCreationPolicy.STATELESS`. No server-side session state at all.
- **Token refresh**: `POST /auth/refresh` reads the cookie, validates the refresh token, and issues a new access token — no re-login required.
- **`JWTAuthFilter`** is a `OncePerRequestFilter`. If JWT parsing throws a `JwtException`, it delegates to Spring's `HandlerExceptionResolver` so the error flows through `GlobalExceptionHandler` rather than bypassing it.
- **BCrypt** password hashing via Spring Security's `PasswordEncoder`.
- **Role-based endpoint protection** in `WebSecurityConfig`: `/admin/**` requires `HOTEL_MANAGER`, `/bookings/**` and `/users/**` require authentication, everything else (auth, browse) is public.
- `User` implements `UserDetails` directly, so Spring Security's authentication pipeline works without a separate adapter.

### 🏨 Hotel & Room Management (Admin)

- Full CRUD for hotels (`HotelController`) and rooms (`RoomAdminController`), both scoped to the authenticated hotel manager — ownership is verified with `user.equals(hotel.getOwner())` before every write.
- **Hotel activation workflow**: calling `PATCH /admin/hotels/{id}/activate` sets `active = true` and triggers `inventoryService.initializeRoomForAYear()` for every room in the hotel. This seeds one `Inventory` row per day per room for the next 365 days, starting today.
- Rooms hold `basePrice`, `capacity`, `totalCount`, photo URLs (`TEXT[]`), and amenities (`TEXT[]`).
- Adding a new room to an already-active hotel also triggers inventory initialization for that room.
- Deleting a hotel cascades: rooms' inventories are deleted first, then the rooms, then the hotel.

### 📦 Inventory System

The `Inventory` table is the core of the availability model. Each row represents **one room type, one day**, and tracks:

| Field | Purpose |
|---|---|
| `totalCount` | Capacity set when room was created |
| `bookedCount` | Rooms confirmed and paid |
| `reservedCount` | Rooms held during active (not yet paid) bookings |
| `surgeFactor` | Admin-settable multiplier, applied by `SurgePricingStrategy` |
| `price` | Dynamically computed price, updated hourly by the scheduler |
| `closed` | Admin can block availability for a date range |

A **`UNIQUE` constraint** on `(hotel_id, room_id, date)` enforces one row per room per day at the database level.

**Pessimistic write locking** (`@Lock(LockModeType.PESSIMISTIC_WRITE)`) is applied at three points:
1. `findAndLockAvailableInventory` — during booking initialization, locks the rows before checking availability and incrementing `reservedCount`
2. `findAndLockReservedInventory` — during payment confirmation, locks before moving `reservedCount → bookedCount`; also called during cancellation to lock before decrementing `bookedCount`
3. `getInventoryAndLockBeforeUpdate` — during admin inventory updates (surge factor / close flag), locks before the bulk UPDATE

**Bulk JPQL `UPDATE` statements** avoid loading entities into memory for the three booking transitions:
- `initBooking`: `reservedCount += numberOfRooms` (only if available)
- `confirmBooking`: `reservedCount -= numberOfRooms`, `bookedCount += numberOfRooms`
- `cancelBooking`: `bookedCount -= numberOfRooms`

A separate `HotelMinPrice` table stores the cheapest room price per hotel per day, maintained by the scheduler. This lets hotel search queries hit a small aggregated table rather than joining across the full `Inventory` table.

### 🔍 Hotel Search

`GET /hotels/search` accepts a `HotelSearchRequest` as `@RequestBody` (note: body on a GET — an intentional tradeoff for complex filter objects, acknowledged in [Design Decisions](#-design-decisions)).

The search queries `HotelMinPriceRepository` with a JPQL aggregate:

```sql
SELECT new HotelPriceDto(i.hotel, AVG(i.price))
FROM HotelMinPrice i
WHERE i.hotel.city = :city
  AND i.date BETWEEN :startDate AND :endDate
  AND i.hotel.active = true
GROUP BY i.hotel
```

Results are paginated (default page 0, size 10, configurable in the request DTO).

> **Known gap**: the search query does not filter by `roomsCount` — hotels are returned based on pricing alone, not confirmed availability of the required number of rooms. A correct availability query exists in `InventoryRepository.findHotelsWithAvailableInventory()` but is not wired into the search path. See [Known Gaps](#-known-gaps).

### 📅 Booking State Machine

```
RESERVED → GUESTS_ADDED → PAYMENTS_PENDING → CONFIRMED → CANCELLED
```

| Transition | Triggered by | What happens |
|---|---|---|
| `→ RESERVED` | `POST /bookings/init` | Inventory locked + `reservedCount` incremented; booking amount calculated using current dynamic prices |
| `→ GUESTS_ADDED` | `POST /bookings/{id}/addGuests` | Guest IDs validated against user's guest list and linked via `booking_guest` join table |
| `→ PAYMENTS_PENDING` | `POST /bookings/{id}/payments` | Stripe Checkout Session created; session URL returned to frontend for redirect |
| `→ CONFIRMED` | Stripe webhook `checkout.session.completed` | `reservedCount → bookedCount` in inventory; `paymentSessionId` persisted on booking |
| `→ CANCELLED` | `POST /bookings/{id}/cancel` | Only confirmed bookings can be cancelled; `bookedCount` decremented; Stripe refund initiated automatically |

**Expiry guard**: `hasBookingExpired()` checks `booking.createdAt + 10 minutes < now()`. This is enforced on `addGuests` and `initiatePayments` — if the booking is stale, an `IllegalStateException` is thrown. See [Known Gaps](#-known-gaps) for what this does and doesn't do.

**Amount calculation**: booking price is computed at reservation time using `pricingService.calculateTotalPrice(inventoryList)`, which runs the full pricing pipeline over each day's inventory and sums the results. This amount is stored on the booking and passed to Stripe — so the price shown to the user reflects the dynamic price at the moment of reservation.

### 💰 Dynamic Pricing Engine

The pricing pipeline is implemented as a **Decorator chain** over a `PricingStrategy` interface. Each strategy wraps the one before it, so the execution order matters:

```
BasePricingStrategy
  └── SurgePricingStrategy        (× surgeFactor, admin-settable per date range)
        └── OccupancyPricingStrategy   (× 1.2 when bookedCount / totalCount > 0.8)
              └── UrgencyPricingStrategy    (× 1.15 when check-in is within 7 days)
                    └── HolidayPricingStrategy  (× 1.25 — stub, see below)
```

This follows the **Open/Closed Principle**: adding a new pricing rule means creating a new strategy class and inserting it into the chain — no existing strategy is modified.

**`PricingUpdateService`** recalculates prices for every hotel every hour via `@Scheduled(cron = "0 0 * * * *")`. It processes hotels in pages of 100 to avoid loading everything into memory at once:
1. For each hotel, fetches all inventory rows from today + 1 year
2. Runs `calculateDynamicPricing()` on each row and saves all updated `Inventory` rows in bulk (`saveAll`)
3. Groups updated inventory prices by date, takes the minimum per day, and bulk-upserts `HotelMinPrice`

> **Holiday strategy**: `HolidayPricingStrategy` always applies the 1.25× multiplier because `isTodayHoliday` is hardcoded to `true` (pending integration with a holiday calendar API). This means all prices currently include the holiday markup every day. This is a known stub — see [Known Gaps](#-known-gaps).

### 💳 Stripe Payment Integration

`CheckoutServiceImpl.getCheckoutSession()`:
1. Creates a Stripe `Customer` with the user's name and email
2. Creates a `Session` in `PAYMENT` mode with billing address collection required
3. Sets line item: quantity 1, currency INR, unit amount = `booking.amount × 100` (paise)
4. Line item name = `{hotelName} : {roomType}`, description = `Booking ID: {id}`
5. Both success and cancel URLs point to `{frontendUrl}/payments/{bookingId}/status`
6. Saves the session ID on the booking before returning the session URL

`WebhookController.capturePayments()`:
- Validates the `Stripe-Signature` header via `Webhook.constructEvent(payload, sigHeader, endpointSecret)`
- On `checkout.session.completed`: confirms booking, promotes inventory from reserved → booked
- Unhandled event types are logged as `WARN` but not errored (deliberate — graceful degradation)

`cancelBooking()` (service layer):
- Only `CONFIRMED` bookings can be cancelled
- Calls `Session.retrieve(paymentSessionId)` then `Refund.create()` with the session's `paymentIntent`
- Inventory `bookedCount` is decremented

### 🧾 Revenue Reporting (Admin)

`GET /admin/hotels/{id}/reports?startDate=&endDate=` (defaults: last 30 days if not provided) returns:

```json
{
  "bookingCount": 14,
  "totalRevenue": 182000.00,
  "avgRevenue": 13000.00
}
```

Only `CONFIRMED` bookings are counted. Revenue is calculated in-memory by streaming the booking list — no aggregate SQL query. `avgRevenue` uses `RoundingMode.HALF_UP`.

### 🔁 Global API Envelope

`GlobalResponseHandler` implements `ResponseBodyAdvice<Object>` and wraps every response body in `ApiResponse<T>` unless the route contains `/v3/api-docs` or `/actuator`. This means all API responses share the same shape:

```json
{
  "timeStamp": "2024-11-01T10:30:00",
  "data": { ... },
  "error": null
}
```

`GlobalExceptionHandler` maps four exception types:
| Exception | HTTP Status |
|---|---|
| `ResourceNotFoundException` | 404 |
| `AuthenticationException` | 401 |
| `JwtException` | 401 |
| `AccessDeniedException` | 403 |

The generic `Exception` handler is present in the code but **commented out** — uncaught exceptions will produce Spring's default error response rather than an `ApiError`. See [Known Gaps](#-known-gaps).

---

## 🛠 Tech Stack

| Layer | Technology |
|---|---|
| Language | Java 17 |
| Framework | Spring Boot 3.x |
| Security | Spring Security + JJWT |
| Persistence | Spring Data JPA + Hibernate |
| Database | PostgreSQL |
| Payments | Stripe Java SDK |
| Mapping | ModelMapper |
| Build | Maven |
| API Docs | SpringDoc OpenAPI (`@Operation` annotations on every endpoint) |

---

## 🌐 API Reference

All endpoints are prefixed with `/api/v1`.

### Auth
| Method | Endpoint | Auth | Description |
|---|---|---|---|
| `POST` | `/auth/signup` | Public | Register; new users get `GUEST` role |
| `POST` | `/auth/login` | Public | Returns `accessToken` in body; `refreshToken` in HttpOnly cookie |
| `POST` | `/auth/refresh` | Cookie | Issues new access token from refresh cookie |

### Browse (Public)
| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/hotels/search` | Search by city + date range + room count (body, paginated) |
| `GET` | `/hotels/{hotelId}/info` | Hotel detail with per-room average price for a date range |

### Booking (Authenticated Guest)
| Method | Endpoint | Description |
|---|---|---|
| `POST` | `/bookings/init` | Reserve rooms; locks inventory, calculates price |
| `POST` | `/bookings/{id}/addGuests` | Attach guest IDs (must be in user's guest list) |
| `POST` | `/bookings/{id}/payments` | Create Stripe session; returns redirect URL |
| `GET` | `/bookings/{id}/status` | Poll booking status |
| `POST` | `/bookings/{id}/cancel` | Cancel + automatic Stripe refund (CONFIRMED only) |

### User & Guests (Authenticated)
| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/users/profile` | Get own profile |
| `PATCH` | `/users/profile` | Update name, DOB, gender |
| `GET` | `/users/myBookings` | All bookings for current user |
| `GET` | `/users/guests` | List saved guests |
| `POST` | `/users/guests` | Add a guest |
| `PUT` | `/users/guests/{id}` | Update a guest |
| `DELETE` | `/users/guests/{id}` | Remove a guest |

### Admin — Hotel Manager Only (`HOTEL_MANAGER` role)
| Method | Endpoint | Description |
|---|---|---|
| `POST` | `/admin/hotels` | Create hotel (inactive by default) |
| `GET` | `/admin/hotels` | All hotels owned by current manager |
| `GET` | `/admin/hotels/{id}` | Get hotel by ID |
| `PUT` | `/admin/hotels/{id}` | Update hotel |
| `DELETE` | `/admin/hotels/{id}` | Delete hotel + rooms + inventories |
| `PATCH` | `/admin/hotels/{id}/activate` | Activate + seed 365-day inventory |
| `POST` | `/admin/hotels/{id}/rooms` | Add room |
| `GET` | `/admin/hotels/{id}/rooms` | List rooms |
| `GET` | `/admin/hotels/{id}/rooms/{roomId}` | Get room |
| `PUT` | `/admin/hotels/{id}/rooms/{roomId}` | Update room |
| `DELETE` | `/admin/hotels/{id}/rooms/{roomId}` | Delete room + inventories |
| `GET` | `/admin/inventory/rooms/{roomId}` | View all inventory for a room |
| `PATCH` | `/admin/inventory/rooms/{roomId}` | Update surge factor + closed status for a date range |
| `GET` | `/admin/hotels/{id}/bookings` | All bookings for a hotel |
| `GET` | `/admin/hotels/{id}/reports` | Revenue report (optional date range, defaults to last 30 days) |

### Webhook
| Method | Endpoint | Description |
|---|---|---|
| `POST` | `/webhook/payment` | Stripe webhook; validates `Stripe-Signature` header |

---

## ⚡ Quick Start

### Prerequisites
- Java 17+
- Maven 3.6+
- PostgreSQL 14+
- Stripe test account

### 1. Clone

```bash
git clone https://github.com/sushant-gargi/stayease.git
cd stayease
```

### 2. Configure

Create `src/main/resources/application.properties` (the committed file contains test credentials — replace everything):

```properties
spring.application.name=StayEase

spring.datasource.url=jdbc:postgresql://localhost:5432/stayease
spring.datasource.username=YOUR_DB_USERNAME
spring.datasource.password=YOUR_DB_PASSWORD
spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.format_sql=true

server.servlet.context-path=/api/v1

# Min 32 characters, high entropy
jwt.secretKey=YOUR_STRONG_JWT_SECRET_KEY_HERE

# For Stripe redirect URLs after payment
frontend.url=http://localhost:3000

stripe.secret.key=${STRIPE_SECRET_KEY}
stripe.webhook.secret=${STRIPE_WEBHOOK_SECRET}
```

```bash
export STRIPE_SECRET_KEY=sk_test_...
export STRIPE_WEBHOOK_SECRET=whsec_...
```

### 3. Run

```bash
./mvnw spring-boot:run
```

- API: `http://localhost:8080/api/v1`
- Swagger UI: `http://localhost:8080/api/v1/swagger-ui.html`

### 4. Test Stripe Webhooks Locally

```bash
stripe listen --forward-to localhost:8080/api/v1/webhook/payment
```

Copy the webhook signing secret printed by the CLI into `STRIPE_WEBHOOK_SECRET`.

---

## 🧠 Design Decisions

**`GET /hotels/search` uses `@RequestBody` instead of query parameters.** A search with city, start date, end date, room count, page, and size is unwieldy as a query string. Using a request body allows a structured DTO. The tradeoff is that GET-with-body is technically non-standard (some proxies strip it). A production alternative would be `POST /hotels/search` or Spring's `@ParameterObject` with `Pageable`.

**`HotelMinPrice` as a pre-computed search index.** Rather than joining `Inventory` across hundreds of rows for every search request, a scheduled job maintains a summary table with the minimum price per hotel per day. Search queries hit this small table with a simple GROUP BY. The cost is eventual consistency — prices in search results reflect the last hourly batch run, not real-time inventory.

**Pessimistic locking over optimistic locking.** The booking window is short (10 minutes). Under contention, `PESSIMISTIC_WRITE` locks rows for the duration of the transaction rather than relying on version-based retry loops. This is simpler to reason about for a booking system where the user-facing failure ("room no longer available") is acceptable but silent data corruption (double-booking) is not.

**Decorator pattern for pricing strategies.** Each pricing rule is encapsulated in its own class, wrapping the previous one. New rules can be added by creating a new strategy class and inserting it into the chain in `PricingService.calculateDynamicPricing()` — no existing code changes. This also makes unit testing individual rules straightforward since each strategy only needs a mock of the inner one.

**Price locked at reservation, not payment.** `booking.amount` is set at `initBooking` time using the current dynamic prices. This amount is what gets passed to Stripe. The user pays the price they saw when they initiated the booking, even if the hourly recalculation runs between reservation and payment.

**Inventory seeded per-save, not in bulk.** `initializeRoomForAYear()` calls `inventoryRepository.save()` in a loop (366 iterations). A `saveAll()` would be more efficient. This is a known performance gap noted for future improvement.

---

## 🗄 Domain Model

```
User (app_user)
 ├──< Hotel (owner → User)
 │     └──< Room
 │           ├──< Inventory [hotel_id, room_id, date — UNIQUE]
 │           └── HotelMinPrice [hotel_id, date]
 └──< Booking (user, hotel, room)
       └──< Guest (via booking_guest join table)
```

`User` implements `UserDetails` (Spring Security) and overrides `equals/hashCode` on ID, making ownership checks (`user.equals(hotel.getOwner())`) safe and consistent throughout the codebase.

---

## 📄 License

Built for learning and portfolio purposes. Free to explore, fork, or use as reference.
