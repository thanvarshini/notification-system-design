# Notification System Design

> A backend system implementing intelligent vehicle scheduling, real-time notifications, and a priority-based inbox — built for performance and scalability.

---

## Overview

This project is a backend notification system that combines algorithmic scheduling with a robust REST API and real-time communication layer. It supports priority-based inbox management, bulk notification dispatch, and WebSocket-driven live updates — designed with production-grade performance optimizations.

---

## Features

- **Vehicle Scheduling (0/1 Knapsack)** — Optimally assigns vehicles to delivery slots based on capacity and priority constraints using the classic 0/1 Knapsack dynamic programming algorithm.
- **Notification REST APIs** — Full CRUD support for creating, reading, updating, and deleting notifications via clean, versioned endpoints.
- **Priority Inbox** — Surfaces the top 10 notifications per user, ranked by a weighted combination of priority level and recency.
- **Real-time Updates** — WebSocket integration delivers instant push notifications to connected clients without polling.
- **Bulk Notification System** — Queue-based architecture supports sending notifications to large user groups efficiently and reliably.

---

## Tech Stack

| Layer | Technology |
|---|---|
| Runtime | Node.js |
| Framework | Express.js |
| Database | PostgreSQL |
| Real-time | WebSockets (ws) |
| Caching | Redis |
| Queue | Bull / BullMQ |
| Package Manager | pnpm |
| Language | TypeScript |

---

## Project Structure

```
notification-system/
├── src/
│   ├── controllers/        # Route handler logic
│   ├── routes/             # API route definitions
│   ├── services/           # Business logic layer
│   ├── algorithms/         # Knapsack scheduling logic
│   ├── websocket/          # WebSocket server setup
│   ├── queues/             # Bull queue workers and producers
│   ├── middlewares/        # Auth, error handling, validation
│   ├── config/             # DB, Redis, environment config
│   └── index.ts            # App entry point
├── prisma/                 # Database schema and migrations
├── .env.example
├── package.json
└── README.md
```

---

## API Overview

| Method | Endpoint | Description |
|---|---|---|
| `POST` | `/api/notifications` | Create a new notification |
| `GET` | `/api/notifications/:userId` | Fetch all notifications for a user |
| `GET` | `/api/notifications/:userId/inbox` | Get top 10 priority notifications |
| `PATCH` | `/api/notifications/:id/read` | Mark a notification as read |
| `DELETE` | `/api/notifications/:id` | Delete a notification |
| `POST` | `/api/notifications/bulk` | Send bulk notifications via queue |
| `POST` | `/api/schedule/vehicles` | Run Knapsack scheduling algorithm |
| `WS` | `ws://localhost:8080` | Real-time notification stream |

---

## How to Run the Project

### Prerequisites

- Node.js v18+
- PostgreSQL
- Redis
- pnpm

### Installation

```bash
# 1. Clone the repository
git clone https://github.com/your-username/notification-system.git
cd notification-system

# 2. Install dependencies
pnpm install

# 3. Set up environment variables
cp .env.example .env
# Fill in your DB, Redis, and port values in .env

# 4. Run database migrations
pnpm run migrate

# 5. Start the development server
pnpm run dev
```

The server will start at `http://localhost:3000` and the WebSocket server at `ws://localhost:8080`.

---

## Sample Output

Screenshots demonstrating the following are included in the `/screenshots` directory:

- Priority Inbox API response (top 10 notifications)
- Bulk notification dispatch via queue
- Real-time WebSocket notification delivery
- Vehicle scheduling output from the Knapsack algorithm
- Paginated notification list with filters

---

## System Design Highlights

### Indexing
Composite indexes on `(user_id, priority, created_at)` in PostgreSQL ensure fast lookups for inbox and history queries.

### Caching
Redis caches the priority inbox per user with a short TTL. Cache is invalidated automatically on new notification creation or status update.

### Queue-based Processing
Bulk notifications are dispatched through a BullMQ queue with configurable concurrency and retry logic — preventing database overload and ensuring delivery guarantees.

---

## Author

**Karthick**
*Backend Developer — Company Assignment Submission*

> This project was developed as part of a backend engineering assignment. All design decisions prioritize scalability, maintainability, and real-world applicability.
