# Notification System Design
# Stage 1

## Overview
Notification platform for students to receive placement, event and result updates.

## Create Notification
POST /api/v1/notifications

Request:
{
  "title": "Amazon Hiring Drive",
  "message": "Amazon is hiring interns",
  "type": "PLACEMENT"
}

Response:
{
  "notificationId": "n101",
  "status": "created"
}

## Get Notifications
GET /api/v1/notifications

...

## Real-Time Notification Mechanism

Use WebSockets to push notifications instantly to connected users.


# Stage 2

## Database Choice

I recommend PostgreSQL because:

- Structured notification data
- ACID compliance
- Fast indexing
- Easy scaling
- Supports large datasets efficiently

## Database Schema

### Users Table

```sql
CREATE TABLE users (
    user_id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    email VARCHAR(100) UNIQUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### Notifications Table

```sql
CREATE TABLE notifications (
    notification_id SERIAL PRIMARY KEY,
    title VARCHAR(255),
    message TEXT,
    type VARCHAR(50),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### User_Notifications Table

```sql
CREATE TABLE user_notifications (
    id SERIAL PRIMARY KEY,
    user_id INT REFERENCES users(user_id),
    notification_id INT REFERENCES notifications(notification_id),
    is_read BOOLEAN DEFAULT FALSE,
    delivered_at TIMESTAMP
);
```

## Potential Scaling Issues

As notification volume increases:

- Slow queries
- Large table sizes
- Increased storage consumption
- Higher read latency

## Solutions

### Indexing

```sql
CREATE INDEX idx_user_notifications_userid
ON user_notifications(user_id);
```

### Partitioning

Partition notifications by month/year.

### Archiving

Move old notifications to archive tables.

### Caching

Use Redis for frequently accessed notifications.

## Sample Queries

### Create Notification

```sql
INSERT INTO notifications(title,message,type)
VALUES('Amazon Hiring Drive',
'Amazon is hiring interns',
'PLACEMENT');
```

### Get User Notifications

```sql
SELECT n.*
FROM notifications n
JOIN user_notifications un
ON n.notification_id = un.notification_id
WHERE un.user_id = 1;
```

### Mark Notification Read

```sql
UPDATE user_notifications
SET is_read = TRUE
WHERE user_id = 1
AND notification_id = 101;
```


# Stage 3

## Query Analysis

Original Query:

```sql
SELECT *
FROM notifications
WHERE studentID = 1042
  AND isRead = false
ORDER BY createdAt ASC;
```

### Is the query accurate?

Yes, it fetches unread notifications for a student.

However, it is inefficient for a table containing:

- 50,000 students
- 5,000,000 notifications

### Why is it slow?

1. Full table scan if indexes are missing.
2. SELECT * fetches unnecessary columns.
3. Sorting large datasets using ORDER BY.
4. Large number of unread notification records.

### Optimized Query

```sql
SELECT notification_id,
       title,
       message,
       created_at
FROM notifications
WHERE student_id = 1042
  AND is_read = FALSE
ORDER BY created_at DESC
LIMIT 50;
```

### Recommended Index

```sql
CREATE INDEX idx_notifications_student_read_created
ON notifications(student_id, is_read, created_at);
```

### Likely Computational Cost

Without Index:
- O(N) table scan
- Expensive sorting

With Composite Index:
- Approximately O(log N)
- Faster filtering and sorting

## Should We Add Indexes on Every Column?

No.

Reasons:

- Consumes extra storage.
- Slows INSERT and UPDATE operations.
- Many indexes may never be used.
- Query planner may become less efficient.

Indexes should only be created for frequently searched, filtered, joined, or sorted columns.

## Students Who Received Placement Notifications in Last 7 Days

Assuming notification_type contains:
- Event
- Result
- Placement

Query:

```sql
SELECT DISTINCT student_id
FROM notifications
WHERE notification_type = 'Placement'
  AND created_at >= CURRENT_DATE - INTERVAL '7 days';

# Stage 4

## Problem

Notifications are fetched from the database on every page load for every student.

Issues:

- High database load
- Increased response time
- Poor user experience
- Scalability challenges as users grow

## Proposed Solutions

### 1. Pagination

Instead of loading all notifications:

```sql
SELECT *
FROM notifications
WHERE student_id = 1042
ORDER BY created_at DESC
LIMIT 20 OFFSET 0;
```

Benefits:
- Reduces database load
- Faster response times

Tradeoff:
- Additional API calls for more data

---

### 2. Redis Caching

Store recently accessed notifications in Redis.

Flow:

User Request → Redis Cache → Database (only on cache miss)

Benefits:
- Very low latency
- Reduces database queries

Tradeoff:
- Additional infrastructure
- Cache invalidation complexity

---

### 3. Real-Time Notifications using WebSockets

Instead of fetching notifications on every page refresh, push notifications instantly.

Benefits:
- Real-time updates
- Better user experience
- Fewer repeated database reads

Tradeoff:
- Persistent connections
- More server memory usage

---

### 4. Notification Count Endpoint

Instead of loading full notifications:

```http
GET /api/v1/notifications/unread-count
```

Response:

```json
{
  "unreadCount": 5
}
```

Benefits:
- Minimal data transfer
- Faster dashboard loading

Tradeoff:
- Requires separate API

---

### 5. Database Indexing

```sql
CREATE INDEX idx_notifications_student_created
ON notifications(student_id, created_at);
```

Benefits:
- Faster filtering
- Faster sorting

Tradeoff:
- Additional storage
- Slower writes

---

### 6. Background Processing

Use message queues such as RabbitMQ or Kafka for notification delivery.

Benefits:
- Handles spikes in traffic
- Improves reliability

Tradeoff:
- Increased architectural complexity

## Recommended Architecture

Client
↓
WebSocket Server
↓
Redis Cache
↓
Notification Service
↓
PostgreSQL

This architecture minimizes database load, improves response times, and supports large-scale notification delivery.


# Stage 5

## Problems in Current Implementation

```python
for student_id in student_ids:
    send_email(student_id, message)
    save_to_db(student_id, message)
    push_to_app(student_id, message)
```

Issues:

1. Sequential processing is very slow for 50,000 students.
2. If email fails midway, processing becomes inconsistent.
3. No retry mechanism.
4. No fault tolerance.
5. High API response time.
6. Database writes block notification delivery.

## Failure Scenario

If email delivery fails for 200 students:

- Some students receive notifications.
- Some students receive emails.
- Some students receive neither.
- System enters inconsistent state.

## Improved Architecture

### Step 1: Save Notification Once

```sql
INSERT INTO notifications(title, message, notification_type)
VALUES('Placement Drive', 'Amazon Hiring', 'Placement');
```

### Step 2: Publish Event to Queue

Use:

- RabbitMQ
- Apache Kafka

```text
HR Request
    ↓
Notification Service
    ↓
Message Queue
    ↓
Worker Services
```

### Step 3: Parallel Workers

Workers process batches independently:

```text
Worker 1 → Email
Worker 2 → In-App Notification
Worker 3 → Analytics
```

## Retry Mechanism

Failed deliveries should be retried.

Example:

- Retry 1 after 1 minute
- Retry 2 after 5 minutes
- Retry 3 after 15 minutes

Failed messages move to Dead Letter Queue (DLQ).

## Database and Email Execution

These should NOT happen sequentially.

Instead:

1. Save notification metadata.
2. Push job to queue.
3. Return success immediately.
4. Background workers send emails and app notifications.

## Benefits

- Faster response time
- High scalability
- Fault tolerance
- Easier monitoring
- Handles 50,000+ students efficiently

## Recommended Architecture

HR Portal
    ↓
Notification Service
    ↓
Kafka / RabbitMQ
    ↓
Email Workers + App Notification Workers
    ↓
PostgreSQL + Redis Cache

This architecture ensures reliable and scalable notification delivery for large user volumes.