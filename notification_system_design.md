Notification System — Design Document
Project: Vehicle Scheduling + Notification System Backend Status: Complete & Submission Ready

Table of Contents
Stage 1 — API Design
Stage 2 — Database Design
Stage 3 — Query Optimization
Stage 4 — Performance Improvements
Stage 5 — Bulk Notifications System
Stage 6 — Priority Inbox
Stage 1 — API Design
Overview
The system exposes a RESTful API over HTTP. All endpoints except /healthz require a Bearer token in the Authorization header.

Base URL: http://localhost:8080/api

Authentication Header (required on all protected routes):

Authorization: Bearer admin-token-secret-2024

Endpoints
GET /healthz
Description: Public health check — no token required.

Response:

{
  "status": "ok"
}

GET /vehicles
Description: Fetch all vehicle tasks (TaskID, Duration, Impact).

Request Headers:

Authorization: Bearer admin-token-secret-2024

Sample Response:

{
  "count": 20,
  "vehicles": [
    { "taskId": "T001", "duration": 8,  "impact": 120 },
    { "taskId": "T002", "duration": 5,  "impact": 80  },
    { "taskId": "T003", "duration": 12, "impact": 200 }
  ]
}

GET /depots
Description: Fetch all depots with their mechanic hour capacity.

Request Headers:

Authorization: Bearer admin-token-secret-2024

Sample Response:

{
  "count": 3,
  "depots": [
    { "depotId": "D001", "name": "Northern Depot", "location": "Manchester", "mechanicHours": 40 },
    { "depotId": "D002", "name": "Southern Depot", "location": "Bristol",    "mechanicHours": 35 },
    { "depotId": "D003", "name": "Eastern Depot",  "location": "Norwich",    "mechanicHours": 25 }
  ]
}

GET /schedule/optimize
Description: Run 0/1 Knapsack DP to select vehicles that maximise impact within total mechanic hours.

Request Headers:

Authorization: Bearer admin-token-secret-2024

Sample Response:

{
  "algorithm": "0/1 Knapsack (Dynamic Programming, space-optimised 1-D table)",
  "capacity": {
    "total": 100,
    "used": 100,
    "remaining": 0,
    "utilizationPercent": 100
  },
  "result": {
    "selectedTaskIds": ["T002", "T003", "T005", "T006", "T007", "T009", "T011", "T013", "T015", "T019"],
    "totalImpact": 1620,
    "totalDuration": 100,
    "itemsConsidered": 20,
    "itemsSelected": 10
  }
}

POST /schedule/optimize
Description: Run the knapsack with custom depot/vehicle filters or a capacity override.

Request Headers:

Authorization: Bearer admin-token-secret-2024
Content-Type: application/json

Sample Request Body:

{
  "depotIds": ["D001", "D002"],
  "capacityOverride": 60
}

Sample Response:

{
  "result": {
    "selectedTaskIds": ["T003", "T005", "T009", "T015"],
    "totalImpact": 795,
    "totalDuration": 44,
    "itemsSelected": 4
  }
}

GET /notifications
Description: List all notifications with optional filters.

Query Parameters: priority, type, userId, read, limit, offset

Request Headers:

Authorization: Bearer admin-token-secret-2024

Sample Response:

{
  "total": 20,
  "count": 5,
  "notifications": [
    {
      "id": "N001",
      "userId": "U1",
      "title": "Engine Overhaul Alert",
      "message": "Vehicle T005 requires immediate engine overhaul.",
      "type": "alert",
      "priority": "critical",
      "read": false,
      "createdAt": "2024-05-01T08:00:00Z"
    }
  ]
}

GET /notifications/inbox
Description: Priority inbox — top 10 unread notifications sorted by priority then recency.

Request Headers:

Authorization: Bearer admin-token-secret-2024

Sample Response:

{
  "count": 10,
  "inbox": [
    { "id": "N015", "priority": "critical", "title": "Emergency Task Queued", "read": false },
    { "id": "N012", "priority": "critical", "title": "Tyre Replacement Overdue", "read": false },
    { "id": "N018", "priority": "high",     "title": "Compliance Deadline", "read": false }
  ]
}

POST /notifications
Description: Create a single notification.

Request Headers:

Authorization: Bearer admin-token-secret-2024
Content-Type: application/json

Sample Request Body:

{
  "userId": "U1",
  "title": "Brake Check Overdue",
  "message": "Vehicle T007 brake check is overdue by 3 days.",
  "type": "warning",
  "priority": "high"
}

Sample Response (201 Created):

{
  "id": "N021",
  "userId": "U1",
  "title": "Brake Check Overdue",
  "type": "warning",
  "priority": "high",
  "read": false,
  "createdAt": "2024-05-02T10:00:00Z"
}

POST /notifications/bulk
Description: Create multiple notifications in one request.

Sample Request Body:

{
  "items": [
    { "userId": "U1", "title": "Alert 1", "message": "...", "type": "alert",   "priority": "critical" },
    { "userId": "U2", "title": "Info 1",  "message": "...", "type": "info",    "priority": "low"      }
  ]
}

Sample Response (201 Created):

{
  "created": 2,
  "notifications": [ { "id": "N022" }, { "id": "N023" } ]
}

PATCH /notifications/:id/read
Description: Mark a single notification as read.

Sample Response:

{
  "id": "N001",
  "read": true,
  "readAt": "2024-05-02T10:05:00Z"
}

PATCH /notifications/bulk-read
Description: Mark multiple notifications as read in one call.

Sample Request Body:

{ "ids": ["N001", "N002", "N003"] }

Sample Response:

{
  "updated": 3,
  "failed": []
}

Real-Time Notifications — WebSocket Design
For live notification delivery without polling, the system can use WebSockets:

Technology: Socket.IO or native ws library on the same Express server
Flow:
Client connects: ws://localhost:8080/notifications/live
Server authenticates the token on connection handshake
On POST /notifications, server emits event to the target user's socket room
Client receives notification instantly without polling
Client  ──── connect (token) ────►  Server
Server  ◄─── join room "user:U1" ──  Server
[New notification created for U1]
Server  ──── emit("notification", {...}) ──►  Client

Alternative (simpler): Server-Sent Events (SSE) via GET /notifications/stream — one-way, works over plain HTTP, no library needed.

Stage 2 — Database Design
Schema
students Table
Column	Type	Description
id	SERIAL PRIMARY KEY	Unique student ID
name	VARCHAR(100) NOT NULL	Full name
email	VARCHAR(150) UNIQUE NOT NULL	Email address
createdAt	TIMESTAMPTZ DEFAULT NOW()	Record creation time
CREATE TABLE students (
  id         SERIAL PRIMARY KEY,
  name       VARCHAR(100)        NOT NULL,
  email      VARCHAR(150) UNIQUE NOT NULL,
  created_at TIMESTAMPTZ         NOT NULL DEFAULT NOW()
);

notifications Table
Column	Type	Description
id	TEXT PRIMARY KEY	Unique notification ID
studentId	INTEGER REFERENCES students(id)	Target student
title	TEXT NOT NULL	Short heading
message	TEXT NOT NULL	Full notification body
type	TEXT NOT NULL	alert, info, warning, success, system
priority	TEXT NOT NULL	critical, high, medium, low
isRead	BOOLEAN DEFAULT FALSE	Read status
createdAt	TIMESTAMPTZ DEFAULT NOW()	When created
readAt	TIMESTAMPTZ	When marked read (nullable)
metadata	JSONB	Optional extra payload
CREATE TABLE notifications (
  id          TEXT        PRIMARY KEY DEFAULT gen_random_uuid(),
  student_id  INTEGER     NOT NULL REFERENCES students(id),
  title       TEXT        NOT NULL,
  message     TEXT        NOT NULL,
  type        TEXT        NOT NULL CHECK (type IN ('alert','info','warning','success','system')),
  priority    TEXT        NOT NULL CHECK (priority IN ('critical','high','medium','low')),
  is_read     BOOLEAN     NOT NULL DEFAULT FALSE,
  created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  read_at     TIMESTAMPTZ,
  metadata    JSONB
);

Indexes
-- Main index: supports inbox query (filter + sort in one scan)
CREATE INDEX idx_student_read_created
  ON notifications (student_id, is_read, created_at DESC)
  WHERE is_read = FALSE;

This composite index supports the most common query pattern:

Filter by student_id ✓
Filter by is_read = FALSE ✓
Sort by created_at DESC ✓
Sample SQL Queries
Fetch unread notifications for a student:

SELECT *
FROM   notifications
WHERE  student_id = 1042
  AND  is_read    = FALSE
ORDER BY created_at DESC
LIMIT 10;

Insert a new notification:

INSERT INTO notifications (student_id, title, message, type, priority)
VALUES (1042, 'Exam Result Published', 'Your result for CS301 is now available.', 'info', 'high');

Mark a notification as read:

UPDATE notifications
SET    is_read  = TRUE,
       read_at  = NOW()
WHERE  id = 'N001';

Stage 3 — Query Optimization
The Query
SELECT *
FROM   notifications
WHERE  student_id = 1042
  AND  is_read    = FALSE
ORDER BY created_at DESC;

Is the Query Correct?
Yes — the query logic is correct. It correctly filters unread notifications for a specific student and returns them newest-first.

Why Is It Slow (Without an Index)?
Problem	Explanation
Full table scan	Without an index, the database reads every row in the table to find matching rows — O(n)
Sorting cost	After filtering, all matching rows must be sorted by created_at, adding extra CPU time
Large dataset	With millions of rows, even a small percentage match means thousands of rows scanned and sorted
Solution — Add a Composite Index
CREATE INDEX idx_student_read_created
  ON notifications (student_id, is_read, created_at DESC);

How This Helps
Without Index	With Index
Full table scan — O(n)	B-tree index lookup — O(log n)
Sorts all matched rows	Index already ordered by created_at DESC
Slow on large tables	Constant speed regardless of table size
The database jumps directly to rows where student_id = 1042 AND is_read = FALSE
Results are already sorted by created_at DESC inside the index — no extra sort step needed
Why You Should NOT Index Every Column
Adding an index on every column seems like it would speed up all queries — but it causes serious problems:

Problem	Explanation
Slower inserts/updates	Every write must update all indexes — more work per row
High memory/disk usage	Each index is a separate data structure stored on disk
Query planner confusion	Too many indexes can cause the planner to choose a suboptimal index
Rule of thumb: Only index columns used in WHERE, ORDER BY, or JOIN clauses in your most frequent queries.

Stage 4 — Performance Improvements
1. Caching with Redis
How it works:

Store the result of the inbox query in Redis with a short TTL (e.g., 60 seconds)
On every request, check Redis first — if found, return instantly without hitting the database
Request → Check Redis → Cache HIT  → Return cached result (fast)
                     → Cache MISS → Query DB → Store in Redis → Return result

Tradeoff:

Dramatically reduces DB load
Data can be stale for up to 60 seconds (cache consistency issue)
2. Pagination (limit + offset)
How it works:

Never return all notifications at once
Return a fixed page size (e.g., 10 or 20) and let the client request the next page
GET /notifications?limit=10&offset=0   → Page 1
GET /notifications?limit=10&offset=10  → Page 2

Tradeoff:

Small, predictable response sizes
Requires multiple API calls to load all data; offset gets slower on very large tables
3. Lazy Loading
How it works:

Load only what is visible on screen initially
Fetch more data only when the user scrolls down (infinite scroll) or clicks "Load more"
Tradeoff:

Faster initial page load, lower bandwidth
More complex client-side logic; harder to jump to a specific page
4. Avoid Fetching All Notifications on Every Request
Problem: Fetching all 10,000 notifications on every page load is wasteful.

Solutions:

Use the /inbox endpoint (returns only top 10 unread) for the dashboard view
Use /notifications?limit=20 for the full list view
Only fetch new notifications since the last known ID (delta fetch)
Stage 5 — Bulk Notifications System
Problems with the Naive Approach
# BAD — naive sequential implementation
function notify_all(student_ids, message):
    for student_id in student_ids:
        save_to_db(student_id, message)    # Blocks until DB write completes
        send_email(student_id, message)    # Blocks until email sent
        send_push(student_id, message)     # Blocks until push sent

Problem	Impact
Sequential processing	Sending to 10,000 students takes 10,000× single-send time
No retry mechanism	If email fails for student 500, notification is lost permanently
DB and email tightly coupled	A slow email provider blocks DB writes for everyone
No visibility	No way to check progress or failed jobs
Improved Design — Queue-Based Architecture
Core idea: Separate the "create job" step from the "process job" step using a message queue (BullMQ / Kafka / RabbitMQ).

API Request
    │
    ▼
Enqueue jobs (fast — just writes to queue)
    │
    ▼
Queue (BullMQ / Kafka)
    │
    ├──► Worker 1 ──► save_to_db + send_email + push_notification
    ├──► Worker 2 ──► save_to_db + send_email + push_notification
    └──► Worker 3 ──► save_to_db + send_email + push_notification

Improved Pseudocode
# Step 1: API handler — just enqueues jobs, returns immediately
function notify_all(student_ids, message):
    for student_id in student_ids:
        enqueue_job({ student_id, message, retries: 0 })
    return { status: "queued", count: len(student_ids) }
# Step 2: Worker — runs independently, processes one job at a time
worker():
    while true:
        job = get_next_job_from_queue()
        try:
            save_to_db(job.student_id, job.message)
            send_email(job.student_id, job.message)
            push_notification(job.student_id, job.message)
            mark_job_complete(job)
        except Exception as e:
            if job.retries < 3:
                requeue_job(job, delay = 2 ^ job.retries seconds)  # Exponential backoff
            else:
                mark_job_failed(job, reason = e)
                alert_admin(job)

Benefits of the Queue-Based Design
Benefit	Explanation
Parallel processing	Multiple workers process jobs simultaneously
Retry logic	Failed jobs are retried automatically with exponential backoff
Decoupled	DB writes and email sends are independent — one failure doesn't block the other
Observable	Queue dashboard shows pending, active, completed, and failed jobs
Non-blocking API	The API returns instantly; processing happens in the background
Stage 6 — Priority Inbox
Overview
The priority inbox shows a user their most important unread notifications — not just the newest ones.

Priority Weights
Notifications are ranked by category importance:

Priority Level	Weight	Example
critical	4	Engine failure, fuel critical, emergency task
high	3	Compliance deadline, capacity warning, battery check
medium	2	Dispatch confirmed, optimisation complete
low	1	Shift start, record update, info messages
Sorting Logic
The inbox is sorted by two criteria in order:

Priority weight (descending) — critical items always appear first
Recency (descending) — within the same priority, newest comes first
Sort key: (priority_weight DESC, created_at DESC)

Example output:

#1  CRITICAL  Emergency Task Queued        (just now)
#2  CRITICAL  Tyre Replacement Overdue     (30 mins ago)
#3  HIGH      Compliance Deadline          (1 hour ago)
#4  HIGH      Cost Threshold Exceeded      (2 hours ago)
#5  MEDIUM    New Vehicle Assigned         (3 hours ago)

Algorithm — Min Heap for Top-K
To efficiently find the top 10 priority notifications from a stream of N incoming notifications:

Algorithm: Use a Min Heap of size k = 10

For each incoming notification:
    If heap.size < 10:
        heap.push(notification)
    Else if notification.priority > heap.min().priority:
        heap.pop()
        heap.push(notification)
Return heap contents sorted descending

Time Complexity: O(n log k)

n = total number of notifications
k = inbox size (10)
Each insert/remove from a heap of size 10 costs O(log 10) ≈ constant
Much faster than sorting all notifications: O(n log n)
Approach	Time Complexity	Good For
Sort all, take top 10	O(n log n)	Small datasets
Min Heap (size 10)	O(n log k)	Large datasets, real-time streams
Handling New Incoming Notifications
When a new notification arrives in real time:

Check its priority — if it is higher than the lowest-priority item currently in the inbox, replace it
Emit via WebSocket — push the updated inbox to the connected client immediately
Invalidate cache — if Redis is caching the inbox, delete the key so the next request fetches fresh data
New notification arrives (priority = critical)
    │
    ├── priority > current inbox minimum? → YES
    │       │
    │       └── Remove lowest item from inbox
    │           Add new notification
    │           Emit updated inbox via WebSocket to user
    │           Invalidate Redis cache key "inbox:userId"
    │
    └── priority <= minimum? → discard (inbox unchanged)