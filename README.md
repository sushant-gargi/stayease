<p align="center">
  <h1 align="center">🏨 AirBnB Clone — Backend API</h1>
  <p align="center">
    <strong>A production-grade hotel booking platform built with Spring Boot</strong>
    <br/>
    <sub>JWT Auth · Dynamic Pricing Engine · Stripe Payments · Role-Based Access · PostgreSQL</sub>
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
  <a href="#-tech-stack">Tech Stack</a> ·
  <a href="#-api-overview">API Overview</a> ·
  <a href="#-quick-start">Quick Start</a> ·
  <a href="#-design-highlights">Design Highlights</a>
</p>

---

A **full-featured hotel booking backend** inspired by AirBnB, designed to demonstrate real-world backend engineering skills — secure authentication, dynamic pricing algorithms, payment integration, and clean layered architecture.

This project goes beyond a CRUD API. It implements **inventory management**, **pessimistic locking for concurrent bookings**, **Stripe checkout with webhook-driven confirmation**, and a **Strategy Pattern-based dynamic pricing engine** — the kind of patterns used in production booking systems.

---

## 🏗 Architecture

```
src/main/java/com/codingshuttle/projects/airBnbApp/
│
├── advice/          → Unified API response wrapper + global exception handling
├── config/          → CORS, ModelMapper, Stripe SDK configuration
├── controller/      → REST controllers (Auth, Booking, Hotel, Inventory, Room, User, Webhook)
├── dto/             → Request & response DTOs (clean separation from entities)
├── entity/          → JPA entities (Booking, Hotel, Room, Inventory, Guest, User)
│   └── enums/       → BookingStatus, PaymentStatus, Role, Gender
├── exception/       → Custom exception classes
├── repository/      → Spring Data JPA repositories with custom JPQL queries
├── security/        → JWT filter, JWT service, auth service, security config
├── service/         → Business logic (Booking, Hotel, Inventory, Room, User, Pricing)
├── strategy/        → Dynamic pricing strategies (Decorator Pattern)
└── util/            → AppUtils (SecurityContext helpers)
```

The project follows **Controller → Service Interface → Service Impl → Repository** layering throughout, with DTOs used at every boundary and entities never leaking out of the service layer.

---

## ✨ Features

### 🔐 Security & Authentication
- **JWT access + refresh token** strategy — short-lived access tokens (10 min), long-lived refresh tokens (6 months) stored as HttpOnly cookies
- **Role-based authorization** via Spring Security: `GUEST` and `HOTEL_MANAGER` roles with endpoint-level restrictions
- Password hashing with **BCrypt**
- Custom `JWTAuthFilter` that delegates exception handling to Spring's `HandlerExceptionResolver` for consistent error responses

### 🏨 Hotel & Room Management (Admin)
- Full CRUD on hotels and rooms, **scoped to the authenticated hotel manager** — ownership is verified on every mutating operation
- Hotel activation workflow: activating a hotel initializes a **365-day inventory** for every room
- Rooms carry base price, capacity, total count, amenity metadata, and photo arrays

### 📦 Inventory System
- Per-room, per-date inventory records tracking `totalCount`, `bookedCount`, `reservedCount`, `surgeFactor`, and `closed` status
- **Pessimistic write locking** (`@Lock(LockModeType.PESSIMISTIC_WRITE)`) on inventory reads during booking to prevent double-booking under concurrency
- Bulk JPQL `UPDATE` queries for efficient inventory mutations (reserve → confirm → cancel)
- Separate `HotelMinPrice` table for fast hotel-search queries (pre-computed cheapest price per hotel per day)

### 🔍 Hotel Search
- City + date range + room count search with pagination
- Leverages the `HotelMinPrice` table with an aggregate JPQL query to return average price per hotel over the search window
- Results mapped to `HotelPriceResponseDto` — no entity exposure

### 📅 Booking Lifecycle
The booking follows a well-defined state machine:

```
RESERVED → GUESTS_ADDED → PAYMENTS_PENDING → CONFIRMED
                                           ↘ CANCELLED / EXPIRED
```

- Booking is **reserved** (inventory locked) at initialization — expires in 10 minutes if not completed
- Guests are attached from the user's saved guest list
- Payment session is created via Stripe Checkout
- Stripe **webhook** (`checkout.session.completed`) triggers the final inventory confirmation
- Cancellation triggers an **automatic Stripe refund** via the Refund API

### 💰 Dynamic Pricing Engine (Strategy + Decorator Pattern)

The pricing pipeline chains four decorators over a base price:

| Strategy | Trigger | Multiplier |
|---|---|---|
| `BasePricingStrategy` | Always | Room base price |
| `SurgePricingStrategy` | Admin-set surge factor | `× surgeFactor` |
| `OccupancyPricingStrategy` | >80% rooms booked | `× 1.2` |
| `UrgencyPricingStrategy` | Check-in within 7 days | `× 1.15` |
| `HolidayPricingStrategy` | Holiday dates | `× 1.25` |

Prices are recalculated **hourly** via a `@Scheduled` batch job (`PricingUpdateService`) that processes hotels in pages of 100 and bulk-upserts the `HotelMinPrice` table.

### 💳 Stripe Payment Integration
- Stripe Checkout Session created per booking with customer creation, line item, billing address collection
- Session URL returned to frontend for redirect
- Webhook validates `Stripe-Signature` header before processing events
- On `checkout.session.completed`: booking confirmed + inventory moved from `reserved` to `booked`
- On cancellation: `Session.retrieve()` + `Refund.create()` for automated refunds

### 🧾 Reporting (Admin)
- Per-hotel booking report with configurable date range
- Aggregates: total confirmed bookings, total revenue, average revenue per booking

### 🔁 Global API Consistency
- `GlobalResponseHandler` wraps all successful responses in `ApiResponse<T>` (with timestamp)
- `GlobalExceptionHandler` maps custom and Spring exceptions to consistent `ApiError` payloads
- Swagger/OpenAPI annotations on every endpoint for documentation

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
| API Docs | SpringDoc OpenAPI (Swagger UI) |

---

## 🔌 API Overview

All endpoints are prefixed with `/api/v1`.

### Auth
| Method | Endpoint | Description | Access |
|---|---|---|---|
| `POST` | `/auth/signup` | Register a new user | Public |
| `POST` | `/auth/login` | Login, returns access token + sets refresh cookie | Public |
| `POST` | `/auth/refresh` | Refresh access token using cookie | Public |

### Browse (Public)
| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/hotels/search` | Search hotels by city, dates, room count (paginated) |
| `GET` | `/hotels/{hotelId}/info` | Get hotel details with room prices for a date range |

### Booking (Authenticated)
| Method | Endpoint | Description |
|---|---|---|
| `POST` | `/bookings/init` | Initialize a booking (reserves inventory) |
| `POST` | `/bookings/{id}/addGuests` | Attach guests to booking |
| `POST` | `/bookings/{id}/payments` | Create Stripe Checkout session |
| `GET` | `/bookings/{id}/status` | Poll booking status |
| `POST` | `/bookings/{id}/cancel` | Cancel booking + trigger refund |

### User Profile (Authenticated)
| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/users/profile` | Get current user profile |
| `PATCH` | `/users/profile` | Update name, DOB, gender |
| `GET` | `/users/myBookings` | Get booking history |
| `GET/POST/PUT/DELETE` | `/users/guests` | Manage saved guest profiles |

### Admin — Hotel Manager Only
| Method | Endpoint | Description |
|---|---|---|
| `GET/POST/PUT/DELETE` | `/admin/hotels` | Hotel management |
| `PATCH` | `/admin/hotels/{id}/activate` | Activate hotel (seeds 1-year inventory) |
| `GET/POST/PUT/DELETE` | `/admin/hotels/{id}/rooms` | Room management |
| `GET/PATCH` | `/admin/inventory/rooms/{roomId}` | View and update room inventory |
| `GET` | `/admin/hotels/{id}/bookings` | All bookings for a hotel |
| `GET` | `/admin/hotels/{id}/reports` | Revenue report with date range |

### Webhook
| Method | Endpoint | Description |
|---|---|---|
| `POST` | `/webhook/payment` | Stripe webhook receiver (signature-verified) |

---

## ⚡ Quick Start

### Prerequisites
- Java 17+
- Maven 3.6+
- PostgreSQL 14+
- A [Stripe](https://stripe.com) test account

### 1. Clone & Configure

```bash
git clone <your-repo-url>
cd airBnbApp
```

Create `src/main/resources/application.properties`:

```properties
spring.application.name=airBnbApp

# Database
spring.datasource.url=jdbc:postgresql://localhost:5432/airbnb
spring.datasource.username=YOUR_DB_USERNAME
spring.datasource.password=YOUR_DB_PASSWORD
spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=true

# Server
server.servlet.context-path=/api/v1

# JWT — use a strong secret (min 256-bit)
jwt.secretKey=YOUR_JWT_SECRET_KEY_AT_LEAST_32_CHARS

# Frontend (for Stripe redirect URLs)
frontend.url=http://localhost:3000

# Stripe
stripe.secret.key=${STRIPE_SECRET_KEY}
stripe.webhook.secret=${STRIPE_WEBHOOK_SECRET}
```

### 2. Set Environment Variables

```bash
export STRIPE_SECRET_KEY=sk_test_...
export STRIPE_WEBHOOK_SECRET=whsec_...
```

### 3. Run

```bash
./mvnw spring-boot:run
```

API is available at `http://localhost:8080/api/v1`
Swagger UI at `http://localhost:8080/api/v1/swagger-ui.html`

### 4. Set Up Stripe Webhook (Local Testing)

```bash
# Install Stripe CLI
stripe listen --forward-to localhost:8080/api/v1/webhook/payment
```

---

## 🧠 Design Highlights

### Pessimistic Locking for Concurrent Bookings
Two users booking the same room simultaneously is handled with `PESSIMISTIC_WRITE` locks on the inventory rows during the booking transaction. The `initBooking` query atomically increments `reservedCount` only when availability checks pass — eliminating race conditions without application-level distributed locks.

### Decorator Pattern Pricing Engine
The pricing system uses composable decorators so new pricing rules can be added without touching existing code (Open/Closed Principle). The `PricingUpdateService` runs every hour and updates both `Inventory` prices and the `HotelMinPrice` summary table in a single batch pass.

### Booking Expiry
Bookings that aren't paid within 10 minutes are considered expired (`hasBookingExpired()` check). Inventory reservation is held for this window, then released — keeping the system consistent without requiring a separate cleanup scheduler.

### Consistent API Layer
Every response, success or error, is wrapped in `ApiResponse<T>` via a `ResponseBodyAdvice` interceptor. This means frontend clients always receive the same envelope format, including timestamps.

---

## 📁 Key Entities

```
User ──< Booking >── Hotel
              │
             Room
              │
          Inventory (hotel_id, room_id, date) — UNIQUE constraint
              │
          HotelMinPrice (hotel_id, date)
```

`User` implements Spring Security's `UserDetails`, allowing direct use with the security context throughout the codebase.

---

## 🚀 What I Learned / Demonstrated

- Designing a **multi-role REST API** with fine-grained authorization
- Implementing **pessimistic database locking** for high-concurrency scenarios
- Building a **composable pricing engine** using the Strategy + Decorator patterns
- Integrating **Stripe Checkout** with webhook-driven state transitions
- Using **Spring Scheduling** for background batch processing
- Writing **custom JPQL queries** for aggregate operations (avg price, revenue reports)
- Structuring a Spring Boot project with clean layering and consistent error handling

---

## 📄 License

This project is built for **learning and portfolio purposes**. Feel free to explore, fork, or use it as a reference.
