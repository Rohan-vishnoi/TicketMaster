# 🎟️ High-Concurrency Ticket Booking System

An enterprise-grade ticketing platform designed to handle massive traffic spikes (up to 1 million concurrent users) without double-booking or crashing. Built with a modern Angular frontend, a Spring Boot backend, and a robust Redis + PostgreSQL data layer.

## 🚀 Tech Stack
- **Frontend:** Angular (TypeScript)
- **Backend:** Spring Boot (Java)
- **Primary Database:** PostgreSQL (Relational DB for ACID transactions)
- **Caching & Queuing:** Redis (In-memory datastore for extreme concurrency)
- **Architecture:** Client-Server, Stateless REST API, Distributed Locking

## 🏗️ System Architecture

To prevent server overloads and race conditions, the system employs a **Virtual Waiting Room** and a **Two-Tier Locking Strategy**. 

```mermaid
sequenceDiagram
    participant U as User (Angular)
    participant LB as Load Balancer
    participant R as Redis (Queue/Cache)
    participant API as Spring Boot API
    participant DB as PostgreSQL

    U->>LB: 1. Request Ticket
    LB->>R: 2. Add to Waiting Room (Sorted Set)
    loop Polling
        U->>R: Check Position
    end
    R-->>U: 3. Turn to Buy!
    U->>API: 4. Select Seat 'A12'
    API->>R: 5. Acquire Distributed Lock (5 min TTL)
    alt Lock Acquired
        API->>DB: 6. SELECT FOR UPDATE (Pessimistic Lock)
        DB-->>API: Seat locked for transaction
        API->>DB: 7. Update status to BOOKED
        API-->>U: 8. Booking Confirmed
    else Lock Failed
        R-->>API: Seat currently held
        API-->>U: Seat Unavailable
    end
```

## 🧠 Concurrency Strategies Implemented
### 1. The Virtual Waiting Room (Redis)
Instead of hammering the SQL database, users are placed into a Redis Sorted Set queue. The backend throttles traffic, allowing only a safe number of users into the actual booking flow at a time.

### 2. Preventing Double Booking
- **Tier 1 (Redis Distributed Lock):** When a user clicks a seat, Redis instantly places a temporary lock (e.g., 5-minute TTL). If another user clicks the same seat, Redis rejects it in milliseconds without querying the SQL database.
- **Tier 2 (PostgreSQL Pessimistic Locking):** During checkout, Spring Boot executes `SELECT * FROM seats WHERE id = ? FOR UPDATE`. This guarantees ACID compliance at the database level, ensuring the ticket is never sold twice even if the cache fails.

## 🗄️ Database Schema

### PostgreSQL Tables
- **`users`**: `id` (UUID), `email`, `password_hash`
- **`events`**: `id`, `name`, `date`, `venue`
- **`seats`**: `id`, `event_id`, `seat_number`, `status` (AVAILABLE, RESERVED, BOOKED), `price`
- **`bookings`**: `id`, `user_id`, `seat_id`, `booking_time`, `payment_status`

## 📁 Project Structure (Monorepo)
```text
ticket-booking-platform/
│
├── frontend/                     # Angular Application
│   ├── src/app/
│   │   ├── components/           # WaitingRoom, StadiumMap, Checkout
│   │   ├── services/             # TicketService, QueueService
│   │   └── models/               # TS Interfaces
│
└── backend/                      # Spring Boot Application
    ├── src/main/java/com/tickets/
    │   ├── controllers/          # TicketController, QueueController
    │   ├── services/             # BookingService, QueueService (Redis Logic)
    │   ├── repositories/         # Spring Data JPA
    │   └── models/               # JPA Entities (Event, Seat, Booking)
    └── src/main/resources/
        └── application.yml       # DB connection pool, Redis host
```
