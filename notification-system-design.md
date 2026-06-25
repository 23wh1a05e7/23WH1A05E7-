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