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
