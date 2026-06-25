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