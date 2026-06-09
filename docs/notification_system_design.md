# Stage 1: Notification System — REST API Design & Contract

> **Audience:** Frontend Engineers  
> **Purpose:** Defines the full REST API surface for the notification platform — endpoints, JSON schemas, headers, error contracts, and the real-time delivery mechanism.  
> **Base URL:** `https://api.yourapp.com/v1`

---

## 1. Core Actions Identified

| # | Action | Description |
|---|--------|-------------|
| 1 | **Fetch notifications** | Retrieve paginated list of notifications for the authenticated user |
| 2 | **Fetch single notification** | Retrieve full detail of one notification by ID |
| 3 | **Mark as read** | Mark one notification as read |
| 4 | **Mark all as read** | Bulk-mark all unread notifications as read |
| 5 | **Delete notification** | Permanently remove a single notification |
| 6 | **Get unread count** | Lightweight poll endpoint for badge counts |
| 7 | **Update preferences** | Set per-channel / per-type delivery preferences |
| 8 | **Get preferences** | Retrieve current notification preferences |
| 9 | **Real-time subscription** | Subscribe to live notification events via SSE |

---

## 2. Shared Conventions

### 2.1 Authentication Header
Every request **must** include:

```
Authorization: Bearer <access_token>
Content-Type: application/json
Accept: application/json
X-Request-ID: <uuid-v4>          // client-generated; echoed in response for tracing
```

### 2.2 Standard Response Envelope

All responses are wrapped in a consistent envelope:

```json
{
  "success": true,
  "data": { },
  "meta": { },
  "error": null
}
```

### 2.3 Standard Error Shape

```json
{
  "success": false,
  "data": null,
  "meta": null,
  "error": {
    "code": "NOTIFICATION_NOT_FOUND",
    "message": "The requested notification does not exist.",
    "details": [],
    "request_id": "3f9a1b2c-..."
  }
}
```

### 2.4 Common HTTP Status Codes

| Code | Meaning |
|------|---------|
| `200` | OK — successful read |
| `204` | No Content — successful write with no body |
| `400` | Bad Request — validation failure |
| `401` | Unauthorized — missing or invalid token |
| `403` | Forbidden — authenticated but not allowed |
| `404` | Not Found |
| `422` | Unprocessable Entity — semantic validation error |
| `429` | Too Many Requests |
| `500` | Internal Server Error |

### 2.5 Notification Object Schema (Canonical)

This is the base shape returned in all notification responses:

```json
{
  "id": "notif_01HZ9X4K2P7VBQR3W5FMGD6TN",
  "type": "COMMENT" | "MENTION" | "SYSTEM" | "ALERT" | "REMINDER" | "PROMOTION",
  "title": "John Doe commented on your post",
  "body": "Great article! I totally agree with your point about...",
  "is_read": false,
  "is_archived": false,
  "priority": "LOW" | "MEDIUM" | "HIGH" | "URGENT",
  "channel": "IN_APP" | "EMAIL" | "PUSH" | "SMS",
  "actor": {
    "id": "usr_789",
    "name": "John Doe",
    "avatar_url": "https://cdn.yourapp.com/avatars/usr_789.jpg"
  },
  "resource": {
    "type": "POST" | "COMMENT" | "TASK" | "INVOICE" | "SYSTEM",
    "id": "post_456",
    "url": "/posts/456"
  },
  "metadata": {},
  "created_at": "2026-06-09T08:30:00Z",
  "read_at": null,
  "expires_at": null
}
```

---

## 3. Endpoints

---

### 3.1 List Notifications

Fetch a paginated, filterable list of notifications for the logged-in user.

```
GET /notifications
```

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `page` | integer | No | Page number, default `1` |
| `per_page` | integer | No | Items per page, default `20`, max `100` |
| `status` | string | No | `unread` \| `read` \| `all` (default `all`) |
| `type` | string | No | Filter by notification type e.g. `COMMENT,MENTION` |
| `priority` | string | No | `LOW` \| `MEDIUM` \| `HIGH` \| `URGENT` |
| `since` | ISO 8601 | No | Only return notifications after this timestamp |

**Request Headers:**
```http
GET /v1/notifications?status=unread&per_page=20&page=1 HTTP/1.1
Authorization: Bearer eyJhbGci...
X-Request-ID: 3f9a1b2c-4d5e-6f7a-8b9c-0d1e2f3a4b5c
Accept: application/json
```

**Response — 200 OK:**
```json
{
  "success": true,
  "data": [
    {
      "id": "notif_01HZ9X4K2P7VBQR3W5FMGD6TN",
      "type": "COMMENT",
      "title": "John Doe commented on your post",
      "body": "Great article! I totally agree...",
      "is_read": false,
      "is_archived": false,
      "priority": "MEDIUM",
      "channel": "IN_APP",
      "actor": {
        "id": "usr_789",
        "name": "John Doe",
        "avatar_url": "https://cdn.yourapp.com/avatars/usr_789.jpg"
      },
      "resource": {
        "type": "POST",
        "id": "post_456",
        "url": "/posts/456"
      },
      "metadata": {},
      "created_at": "2026-06-09T08:30:00Z",
      "read_at": null,
      "expires_at": null
    }
  ],
  "meta": {
    "current_page": 1,
    "per_page": 20,
    "total_items": 84,
    "total_pages": 5,
    "unread_count": 12
  },
  "error": null
}
```

---

### 3.2 Get Single Notification

```
GET /notifications/:id
```

**Request Headers:**
```http
GET /v1/notifications/notif_01HZ9X4K2P7VBQR3W5FMGD6TN HTTP/1.1
Authorization: Bearer eyJhbGci...
X-Request-ID: a1b2c3d4-e5f6-7890-abcd-ef1234567890
Accept: application/json
```

**Response — 200 OK:**
```json
{
  "success": true,
  "data": {
    "id": "notif_01HZ9X4K2P7VBQR3W5FMGD6TN",
    "type": "COMMENT",
    "title": "John Doe commented on your post",
    "body": "Great article! I totally agree with your point about distributed systems.",
    "is_read": false,
    "is_archived": false,
    "priority": "MEDIUM",
    "channel": "IN_APP",
    "actor": {
      "id": "usr_789",
      "name": "John Doe",
      "avatar_url": "https://cdn.yourapp.com/avatars/usr_789.jpg"
    },
    "resource": {
      "type": "POST",
      "id": "post_456",
      "url": "/posts/456"
    },
    "metadata": {
      "comment_preview": "Great article! I totally agree..."
    },
    "created_at": "2026-06-09T08:30:00Z",
    "read_at": null,
    "expires_at": null
  },
  "meta": null,
  "error": null
}
```

**Response — 404 Not Found:**
```json
{
  "success": false,
  "data": null,
  "meta": null,
  "error": {
    "code": "NOTIFICATION_NOT_FOUND",
    "message": "No notification found with the given ID.",
    "details": [],
    "request_id": "a1b2c3d4-..."
  }
}
```

---

### 3.3 Mark Single Notification as Read

```
PATCH /notifications/:id/read
```

**Request Headers:**
```http
PATCH /v1/notifications/notif_01HZ9X4K2P7VBQR3W5FMGD6TN/read HTTP/1.1
Authorization: Bearer eyJhbGci...
X-Request-ID: b2c3d4e5-f6a7-8901-bcde-f12345678901
Content-Type: application/json
```

**Request Body:** _(empty — no body required)_

**Response — 200 OK:**
```json
{
  "success": true,
  "data": {
    "id": "notif_01HZ9X4K2P7VBQR3W5FMGD6TN",
    "is_read": true,
    "read_at": "2026-06-09T09:15:22Z"
  },
  "meta": null,
  "error": null
}
```

---

### 3.4 Mark All Notifications as Read

```
POST /notifications/read-all
```

**Request Headers:**
```http
POST /v1/notifications/read-all HTTP/1.1
Authorization: Bearer eyJhbGci...
X-Request-ID: c3d4e5f6-a7b8-9012-cdef-123456789012
Content-Type: application/json
```

**Request Body (optional — scope the bulk action):**
```json
{
  "type": "COMMENT",
  "before": "2026-06-09T09:00:00Z"
}
```

**Response — 200 OK:**
```json
{
  "success": true,
  "data": {
    "marked_read_count": 12
  },
  "meta": null,
  "error": null
}
```

---

### 3.5 Delete Notification

```
DELETE /notifications/:id
```

**Request Headers:**
```http
DELETE /v1/notifications/notif_01HZ9X4K2P7VBQR3W5FMGD6TN HTTP/1.1
Authorization: Bearer eyJhbGci...
X-Request-ID: d4e5f6a7-b8c9-0123-defa-234567890123
```

**Response — 204 No Content** _(empty body)_

**Response — 404 Not Found:**
```json
{
  "success": false,
  "data": null,
  "meta": null,
  "error": {
    "code": "NOTIFICATION_NOT_FOUND",
    "message": "No notification found with the given ID.",
    "details": [],
    "request_id": "d4e5f6a7-..."
  }
}
```

---

### 3.6 Get Unread Count

Lightweight endpoint designed for badge polling. Returns only the count — no notification payloads.

```
GET /notifications/unread-count
```

**Request Headers:**
```http
GET /v1/notifications/unread-count HTTP/1.1
Authorization: Bearer eyJhbGci...
X-Request-ID: e5f6a7b8-c9d0-1234-efab-345678901234
Accept: application/json
```

**Response — 200 OK:**
```json
{
  "success": true,
  "data": {
    "unread_count": 12,
    "by_type": {
      "COMMENT": 5,
      "MENTION": 3,
      "SYSTEM": 2,
      "ALERT": 2
    }
  },
  "meta": {
    "last_checked_at": "2026-06-09T09:18:00Z"
  },
  "error": null
}
```

> **Frontend note:** Poll this endpoint every 30–60 seconds as a fallback when the real-time SSE connection is unavailable. Do **not** poll faster than every 15 seconds.

---

### 3.7 Get Notification Preferences

```
GET /notifications/preferences
```

**Request Headers:**
```http
GET /v1/notifications/preferences HTTP/1.1
Authorization: Bearer eyJhbGci...
X-Request-ID: f6a7b8c9-d0e1-2345-fabc-456789012345
Accept: application/json
```

**Response — 200 OK:**
```json
{
  "success": true,
  "data": {
    "user_id": "usr_123",
    "channels": {
      "in_app": true,
      "email": true,
      "push": false,
      "sms": false
    },
    "types": {
      "COMMENT": {
        "in_app": true,
        "email": true,
        "push": false,
        "sms": false
      },
      "MENTION": {
        "in_app": true,
        "email": true,
        "push": true,
        "sms": false
      },
      "SYSTEM": {
        "in_app": true,
        "email": true,
        "push": false,
        "sms": false
      },
      "ALERT": {
        "in_app": true,
        "email": true,
        "push": true,
        "sms": true
      },
      "REMINDER": {
        "in_app": true,
        "email": false,
        "push": false,
        "sms": false
      },
      "PROMOTION": {
        "in_app": false,
        "email": false,
        "push": false,
        "sms": false
      }
    },
    "quiet_hours": {
      "enabled": true,
      "start": "22:00",
      "end": "08:00",
      "timezone": "Asia/Kolkata"
    },
    "digest": {
      "enabled": false,
      "frequency": "DAILY" | "WEEKLY",
      "time": "08:00"
    },
    "updated_at": "2026-05-01T10:00:00Z"
  },
  "meta": null,
  "error": null
}
```

---

### 3.8 Update Notification Preferences

```
PUT /notifications/preferences
```

**Request Headers:**
```http
PUT /v1/notifications/preferences HTTP/1.1
Authorization: Bearer eyJhbGci...
X-Request-ID: a7b8c9d0-e1f2-3456-abcd-567890123456
Content-Type: application/json
```

**Request Body** _(partial updates supported — send only changed keys)_:
```json
{
  "channels": {
    "push": true
  },
  "types": {
    "PROMOTION": {
      "in_app": false,
      "email": false,
      "push": false,
      "sms": false
    }
  },
  "quiet_hours": {
    "enabled": true,
    "start": "23:00",
    "end": "07:00",
    "timezone": "Asia/Kolkata"
  }
}
```

**Response — 200 OK:**
```json
{
  "success": true,
  "data": {
    "updated": true,
    "updated_at": "2026-06-09T09:22:00Z"
  },
  "meta": null,
  "error": null
}
```

---

## 4. Real-Time Notification Mechanism

### 4.1 Chosen Approach: Server-Sent Events (SSE)

| Factor | SSE | WebSocket | Long Polling |
|--------|-----|-----------|--------------|
| Direction | Server → Client only | Bidirectional | Server → Client |
| Complexity | Low | High | Medium |
| Auto-reconnect | Native (browser) | Manual | Manual |
| Works over HTTP/1.1 | Yes | Requires upgrade | Yes |
| Proxy / firewall friendly | Yes | Sometimes blocked | Yes |
| Best for | Notification feeds | Chat / gaming | Legacy fallback |

Notifications are a **one-way** server-push use case — SSE is the right fit.

---

### 4.2 SSE Subscription Endpoint

```
GET /notifications/stream
```

**Request Headers:**
```http
GET /v1/notifications/stream HTTP/1.1
Authorization: Bearer eyJhbGci...
Accept: text/event-stream
Cache-Control: no-cache
X-Request-ID: b8c9d0e1-f2a3-4567-bcde-678901234567
```

**Response Headers (server):**
```http
HTTP/1.1 200 OK
Content-Type: text/event-stream
Cache-Control: no-cache
Connection: keep-alive
X-Accel-Buffering: no
Access-Control-Allow-Origin: https://yourapp.com
```

---

### 4.3 SSE Event Types & Payloads

Each SSE message follows the format:

```
event: <event_type>
id: <event_id>
data: <json_payload>
retry: 5000

```
_(blank line terminates the event)_

---

#### Event: `notification.new`
Fired when a new notification arrives for the user.

```
event: notification.new
id: evt_01HZ9XABCDEF1234567890
data: {"id":"notif_01HZ9X4K2P7VBQR3W5FMGD6TN","type":"MENTION","title":"Sarah mentioned you in a comment","body":"Hey @you, what do you think about this approach?","is_read":false,"priority":"HIGH","actor":{"id":"usr_456","name":"Sarah Lee","avatar_url":"https://cdn.yourapp.com/avatars/usr_456.jpg"},"resource":{"type":"COMMENT","id":"cmt_789","url":"/posts/123#comment-789"},"created_at":"2026-06-09T09:25:00Z"}
retry: 5000

```

---

#### Event: `notification.read`
Fired when a notification is marked read (e.g. on another device/tab).

```
event: notification.read
id: evt_01HZ9XABCDEF1234567891
data: {"id":"notif_01HZ9X4K2P7VBQR3W5FMGD6TN","is_read":true,"read_at":"2026-06-09T09:26:00Z"}
retry: 5000

```

---

#### Event: `notification.deleted`
Fired when a notification is deleted.

```
event: notification.deleted
id: evt_01HZ9XABCDEF1234567892
data: {"id":"notif_01HZ9X4K2P7VBQR3W5FMGD6TN"}
retry: 5000

```

---

#### Event: `notification.count`
Periodic count sync to keep badge accurate (emitted every 60 seconds or after any change).

```
event: notification.count
id: evt_01HZ9XABCDEF1234567893
data: {"unread_count":11,"by_type":{"COMMENT":4,"MENTION":4,"SYSTEM":2,"ALERT":1}}
retry: 5000

```

---

#### Event: `ping`
Heartbeat emitted every 30 seconds to keep the connection alive through proxies.

```
event: ping
id: evt_01HZ9XABCDEF1234567894
data: {"timestamp":"2026-06-09T09:25:30Z"}
retry: 5000

```

---

### 4.4 Frontend Integration Guide (JavaScript)

```javascript
// 1. Open the SSE stream after login
const token = getAccessToken();

const eventSource = new EventSource(
  `https://api.yourapp.com/v1/notifications/stream`,
  {
    // NOTE: EventSource does not support custom headers natively.
    // Pass token as a query param OR use a wrapper library (see note below).
    // Recommended: use the `@microsoft/fetch-event-source` library which
    // supports Authorization headers.
  }
);

// 2. Handle new notifications
eventSource.addEventListener('notification.new', (e) => {
  const notification = JSON.parse(e.data);
  displayToast(notification);          // show in-app toast
  incrementBadgeCount();               // update bell icon
  prependToNotificationList(notification);
});

// 3. Sync read state across tabs/devices
eventSource.addEventListener('notification.read', (e) => {
  const { id, read_at } = JSON.parse(e.data);
  markNotificationReadInUI(id, read_at);
  decrementBadgeCount();
});

// 4. Sync deletions
eventSource.addEventListener('notification.deleted', (e) => {
  const { id } = JSON.parse(e.data);
  removeNotificationFromUI(id);
});

// 5. Badge count sync
eventSource.addEventListener('notification.count', (e) => {
  const { unread_count } = JSON.parse(e.data);
  setBadgeCount(unread_count);
});

// 6. Handle errors + reconnect
eventSource.onerror = (err) => {
  console.error('SSE connection error:', err);
  // Browser auto-reconnects after the `retry` delay (5000ms).
  // Fall back to polling /notifications/unread-count if offline.
};

// 7. Clean up on logout
function onLogout() {
  eventSource.close();
}
```

> **Token passing note:** Because the browser's native `EventSource` API does not support `Authorization` headers, use one of:
> - **Query parameter:** `?token=<access_token>` _(only if the connection is HTTPS and short-lived tokens are used)_
> - **Cookie-based auth** for the SSE endpoint only
> - **`@microsoft/fetch-event-source`** — a drop-in replacement that supports full header control

---

### 4.5 Reconnection & Fallback Strategy

```
┌─────────────────────────────────────────────────┐
│              Client Startup (logged in)         │
└───────────────────┬─────────────────────────────┘
                    │
                    ▼
          Open SSE /notifications/stream
                    │
          ┌─────────┴──────────┐
          │ Connection OK       │ Connection FAILED
          │                     │
          ▼                     ▼
   Listen for events     Poll GET /notifications/
   (live updates)        unread-count every 30s
          │                     │
          │  Reconnect after     │
          └──── retry delay ─────┘
               (5 seconds,
               browser-native)
```

---

## 5. Endpoint Summary

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/notifications` | List notifications (paginated, filtered) |
| `GET` | `/notifications/:id` | Get single notification |
| `PATCH` | `/notifications/:id/read` | Mark one notification as read |
| `POST` | `/notifications/read-all` | Mark all notifications as read |
| `DELETE` | `/notifications/:id` | Delete a notification |
| `GET` | `/notifications/unread-count` | Get unread badge count |
| `GET` | `/notifications/preferences` | Get user notification preferences |
| `PUT` | `/notifications/preferences` | Update user notification preferences |
| `GET` | `/notifications/stream` | SSE stream for real-time events |

---

## 6. Design Decisions & Notes for Frontend

1. **Optimistic UI:** For `PATCH /:id/read` and `DELETE /:id`, update the UI immediately on user action and roll back only if the API returns a non-2xx response.

2. **Cursor vs. offset pagination:** This design uses offset/page-based pagination for simplicity. If the notification list grows very large, request a switch to cursor-based pagination (`?cursor=<last_id>`).

3. **`X-Request-ID` header:** Always generate and send this UUID per request — it allows backend engineers to trace requests across logs end-to-end. Echo it in bug reports.

4. **SSE token expiry:** If the access token expires mid-stream, the server will close the connection with `401`. The client should detect this, refresh the token, and re-open the stream.

5. **Notification TTL:** Notifications with a non-null `expires_at` should be hidden from the UI after that timestamp without requiring a delete call.

6. **`metadata` field:** This is a flexible JSON object. Content varies by notification type — consult the backend team for type-specific metadata schemas as they are added.

---

*Document version: 1.0 — June 9, 2026*

---

---

# Stage 2: Persistent Storage — Database Design, Scalability & Queries

> **Purpose:** Defines the storage layer that backs the Stage 1 REST API — database choice rationale, full schema, scalability problems and their solutions, and the exact queries each endpoint executes.

---

## 1. Storage Choice: PostgreSQL (Primary) + Redis (Cache Layer)

### 1.1 Why PostgreSQL?

| Criteria | Evaluation |
|----------|------------|
| **Data structure** | Notifications are structured, relational records with predictable fields — a perfect fit for relational schemas |
| **ACID compliance** | Critical for `mark as read` and `delete` operations — partial writes must never leave data inconsistent |
| **JSONB support** | The `metadata` and `actor`/`resource` fields are semi-structured; PostgreSQL's native `JSONB` column stores them efficiently with indexing support |
| **Rich query support** | Filtering by `type`, `priority`, `status`, date ranges, and pagination are all first-class SQL operations |
| **Partitioning** | PostgreSQL supports declarative table partitioning by range (e.g. `created_at`) — essential for managing billions of rows at scale |
| **Maturity & ecosystem** | Battle-tested, excellent Spring Boot / JPA integration, strong tooling |

**Alternatives considered and rejected:**

- **MongoDB:** Semi-structured `metadata` field could suggest NoSQL, but the rest of the schema is rigidly relational. Using MongoDB would sacrifice joins and transactions for a flexibility we only need in one column — JSONB gives us that flexibility within PostgreSQL.
- **Cassandra:** Excellent for write-heavy time-series at extreme scale, but operationally complex and poor for the ad-hoc filtering patterns our APIs require (by type, priority, read status simultaneously).

### 1.2 Why Redis as a Cache Layer?

Redis sits alongside PostgreSQL to serve two specific needs:

| Use case | Detail |
|----------|--------|
| **Unread count cache** | `GET /notifications/unread-count` is called very frequently (badge polling + post-read updates). Computing `COUNT(*)` on millions of rows on every request is expensive. Redis stores a per-user counter that is incremented/decremented atomically. |
| **SSE session registry** | Tracks which users have active SSE connections on which server instances — required for horizontally scaled deployments to route push events to the right server. |

---

## 2. Database Schema

### 2.1 Entity Relationship Overview

```
users (external — referenced by ID only)
  │
  ├──< notifications (core table)
  │         │
  │         └── partitioned by created_at (monthly)
  │
  ├──< notification_preferences (one row per user)
  │
  └──< notification_type_preferences (per user per type)
```

---

### 2.2 Table: `notifications`

This is the core table. Every row is one notification delivered to one recipient.

```sql
CREATE TABLE notifications (
    id                VARCHAR(36)     NOT NULL,                  -- e.g. notif_01HZ9X4K...
    recipient_id      VARCHAR(36)     NOT NULL,                  -- user who receives this
    actor_id          VARCHAR(36),                               -- user who triggered it (nullable for SYSTEM)
    actor_name        VARCHAR(255),
    actor_avatar_url  TEXT,

    type              VARCHAR(50)     NOT NULL,                  -- COMMENT | MENTION | SYSTEM | ALERT | REMINDER | PROMOTION
    priority          VARCHAR(20)     NOT NULL DEFAULT 'MEDIUM', -- LOW | MEDIUM | HIGH | URGENT
    channel           VARCHAR(20)     NOT NULL DEFAULT 'IN_APP', -- IN_APP | EMAIL | PUSH | SMS

    title             VARCHAR(512)    NOT NULL,
    body              TEXT            NOT NULL,

    resource_type     VARCHAR(50),                               -- POST | COMMENT | TASK | INVOICE | SYSTEM
    resource_id       VARCHAR(36),
    resource_url      TEXT,

    metadata          JSONB           NOT NULL DEFAULT '{}',

    is_read           BOOLEAN         NOT NULL DEFAULT FALSE,
    is_archived       BOOLEAN         NOT NULL DEFAULT FALSE,
    is_deleted        BOOLEAN         NOT NULL DEFAULT FALSE,    -- soft delete flag

    read_at           TIMESTAMPTZ,
    expires_at        TIMESTAMPTZ,
    created_at        TIMESTAMPTZ     NOT NULL DEFAULT NOW(),
    updated_at        TIMESTAMPTZ     NOT NULL DEFAULT NOW(),

    PRIMARY KEY (id, created_at)                                 -- composite PK required for partitioning
) PARTITION BY RANGE (created_at);
```

**Monthly partitions (create ahead of time or automate with pg_partman):**

```sql
CREATE TABLE notifications_2026_06
    PARTITION OF notifications
    FOR VALUES FROM ('2026-06-01') TO ('2026-07-01');

CREATE TABLE notifications_2026_07
    PARTITION OF notifications
    FOR VALUES FROM ('2026-07-01') TO ('2026-08-01');

-- Continue for each month...
```

---

### 2.3 Table: `notification_preferences`

One row per user — stores global channel toggles, quiet hours, and digest settings.

```sql
CREATE TABLE notification_preferences (
    user_id             VARCHAR(36)     PRIMARY KEY,

    -- Global channel toggles
    channel_in_app      BOOLEAN         NOT NULL DEFAULT TRUE,
    channel_email       BOOLEAN         NOT NULL DEFAULT TRUE,
    channel_push        BOOLEAN         NOT NULL DEFAULT FALSE,
    channel_sms         BOOLEAN         NOT NULL DEFAULT FALSE,

    -- Quiet hours
    quiet_hours_enabled BOOLEAN         NOT NULL DEFAULT FALSE,
    quiet_hours_start   TIME,                                    -- e.g. '22:00:00'
    quiet_hours_end     TIME,                                    -- e.g. '08:00:00'
    quiet_hours_tz      VARCHAR(100)    NOT NULL DEFAULT 'UTC',  -- IANA timezone

    -- Digest
    digest_enabled      BOOLEAN         NOT NULL DEFAULT FALSE,
    digest_frequency    VARCHAR(20),                             -- DAILY | WEEKLY
    digest_time         TIME,                                    -- e.g. '08:00:00'

    created_at          TIMESTAMPTZ     NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ     NOT NULL DEFAULT NOW()
);
```

---

### 2.4 Table: `notification_type_preferences`

Per-user, per-type channel overrides. Only rows that differ from the global default need to exist.

```sql
CREATE TABLE notification_type_preferences (
    user_id         VARCHAR(36)     NOT NULL,
    notification_type VARCHAR(50)   NOT NULL,                   -- COMMENT | MENTION | SYSTEM | ALERT | REMINDER | PROMOTION

    channel_in_app  BOOLEAN         NOT NULL DEFAULT TRUE,
    channel_email   BOOLEAN         NOT NULL DEFAULT TRUE,
    channel_push    BOOLEAN         NOT NULL DEFAULT FALSE,
    channel_sms     BOOLEAN         NOT NULL DEFAULT FALSE,

    updated_at      TIMESTAMPTZ     NOT NULL DEFAULT NOW(),

    PRIMARY KEY (user_id, notification_type)
);
```

---

### 2.5 Indexes

```sql
-- Most critical: fetch unread notifications for a user, newest first
CREATE INDEX idx_notifications_recipient_read_created
    ON notifications (recipient_id, is_read, created_at DESC)
    WHERE is_deleted = FALSE;

-- Filter by type
CREATE INDEX idx_notifications_recipient_type
    ON notifications (recipient_id, type, created_at DESC)
    WHERE is_deleted = FALSE;

-- Filter by priority
CREATE INDEX idx_notifications_recipient_priority
    ON notifications (recipient_id, priority, created_at DESC)
    WHERE is_deleted = FALSE;

-- Expiry cleanup job
CREATE INDEX idx_notifications_expires_at
    ON notifications (expires_at)
    WHERE expires_at IS NOT NULL AND is_deleted = FALSE;

-- JSONB metadata queries (if needed later)
CREATE INDEX idx_notifications_metadata
    ON notifications USING GIN (metadata);
```

---

## 3. Scalability Problems & Solutions

As the platform grows from thousands to millions of users, the following problems will arise:

---

### Problem 1: Table Size — Billions of Rows

**What happens:** With 1M users each receiving 10 notifications/day, the `notifications` table grows by ~10M rows/day — 3.6B rows/year. Full table scans become catastrophically slow.

**Solution: Range Partitioning by `created_at`**

Already built into the schema above. Each monthly partition is a separate physical table. Queries that include a `created_at` filter (e.g. `since=`) touch only the relevant partition(s) — this is called **partition pruning**.

```sql
-- PostgreSQL automatically routes this to only the June 2026 partition
SELECT * FROM notifications
WHERE recipient_id = 'usr_123'
  AND created_at >= '2026-06-01'
  AND is_deleted = FALSE
ORDER BY created_at DESC
LIMIT 20;
```

Old partitions (e.g. older than 90 days) can be **dropped instantly** with `DROP TABLE notifications_2026_03` — far faster than `DELETE`.

---

### Problem 2: Slow Unread Count Queries

**What happens:** `COUNT(*) WHERE is_read = FALSE` scans potentially millions of rows per user on every badge poll.

**Solution: Redis Counter Cache**

Maintain a Redis hash per user:

```
Key:   notif:unread:<user_id>
Value: { "total": 12, "COMMENT": 5, "MENTION": 3, "SYSTEM": 2, "ALERT": 2 }
```

- **On new notification insert:** `HINCRBY notif:unread:usr_123 total 1` and `HINCRBY notif:unread:usr_123 COMMENT 1`
- **On mark as read:** decrement the relevant counters
- **On mark all as read:** `DEL notif:unread:usr_123` and recompute from DB once
- **Cache miss / first login:** compute from DB and populate Redis

This reduces `GET /notifications/unread-count` to a single Redis `HGETALL` — sub-millisecond response.

---

### Problem 3: Slow Pagination at Deep Offsets

**What happens:** `LIMIT 20 OFFSET 2000` requires the DB to scan and discard 2000 rows before returning results — gets progressively slower at deeper pages.

**Solution: Cursor-Based Pagination**

Replace `page` + `offset` with a `cursor` (the `created_at` + `id` of the last seen row):

```sql
-- Instead of OFFSET, use a WHERE clause on the last seen values
SELECT * FROM notifications
WHERE recipient_id = 'usr_123'
  AND is_deleted = FALSE
  AND (created_at, id) < ('2026-06-09T08:30:00Z', 'notif_01HZ...')
ORDER BY created_at DESC, id DESC
LIMIT 20;
```

This is O(log n) regardless of how deep the page is. The API response includes a `next_cursor` token (base64-encoded `created_at:id`) for the client to pass in the next request.

---

### Problem 4: Write Contention on High-Volume Inserts

**What happens:** Burst notification events (e.g. a viral post triggering 100K mention notifications simultaneously) overwhelm the DB with concurrent inserts.

**Solution: Async Write Queue**

Introduce a message queue (e.g. RabbitMQ or Kafka) between the notification producer and the DB writer:

```
Event Producer → Kafka Topic: notifications.pending
                      ↓
              Notification Worker
           (batched DB inserts, Redis updates)
                      ↓
              PostgreSQL + Redis
```

Workers consume from the queue and perform **batch inserts** (`INSERT INTO notifications VALUES (...), (...), (...)`) — far more efficient than one insert per notification. The API remains fast; delivery becomes eventually consistent (typically within milliseconds).

---

### Problem 5: Old Data Accumulating Indefinitely

**What happens:** Users accumulate thousands of old, read notifications that are never cleaned up, bloating storage and slowing queries.

**Solution: TTL-Based Archival & Partition Drops**

Three-tier retention policy:

| Tier | Age | Action |
|------|-----|--------|
| Active | 0–30 days | Fully queryable in hot partitions |
| Archived | 30–90 days | Moved to cheaper cold storage / compressed partition |
| Deleted | 90+ days | Partition dropped entirely |

A nightly cleanup job also hard-deletes soft-deleted rows and expired notifications:

```sql
-- Run nightly via pg_cron
DELETE FROM notifications
WHERE is_deleted = TRUE
   OR (expires_at IS NOT NULL AND expires_at < NOW());
```

---

## 4. Queries Mapped to REST API Endpoints

---

### 4.1 `GET /notifications` — List Notifications

```sql
-- Base query (status=all, no filters)
SELECT
    id, type, priority, channel, title, body,
    actor_id, actor_name, actor_avatar_url,
    resource_type, resource_id, resource_url,
    metadata, is_read, is_archived,
    read_at, expires_at, created_at
FROM notifications
WHERE recipient_id  = :recipient_id
  AND is_deleted    = FALSE
ORDER BY created_at DESC
LIMIT :per_page
OFFSET (:page - 1) * :per_page;

-- With status=unread filter
SELECT ... FROM notifications
WHERE recipient_id  = :recipient_id
  AND is_read       = FALSE
  AND is_deleted    = FALSE
ORDER BY created_at DESC
LIMIT :per_page OFFSET :offset;

-- With type filter (e.g. type=COMMENT,MENTION)
SELECT ... FROM notifications
WHERE recipient_id  = :recipient_id
  AND type          = ANY(:types)        -- pass as array: ARRAY['COMMENT','MENTION']
  AND is_deleted    = FALSE
ORDER BY created_at DESC
LIMIT :per_page OFFSET :offset;

-- With since filter
SELECT ... FROM notifications
WHERE recipient_id  = :recipient_id
  AND created_at    > :since
  AND is_deleted    = FALSE
ORDER BY created_at DESC
LIMIT :per_page OFFSET :offset;

-- Total count for pagination meta (run alongside main query)
SELECT COUNT(*) FROM notifications
WHERE recipient_id  = :recipient_id
  AND is_deleted    = FALSE;
```

---

### 4.2 `GET /notifications/:id` — Get Single Notification

```sql
SELECT
    id, type, priority, channel, title, body,
    actor_id, actor_name, actor_avatar_url,
    resource_type, resource_id, resource_url,
    metadata, is_read, is_archived,
    read_at, expires_at, created_at
FROM notifications
WHERE id            = :notification_id
  AND recipient_id  = :recipient_id        -- security: ensure ownership
  AND is_deleted    = FALSE;
```

---

### 4.3 `PATCH /notifications/:id/read` — Mark Single as Read

```sql
UPDATE notifications
SET
    is_read    = TRUE,
    read_at    = NOW(),
    updated_at = NOW()
WHERE id            = :notification_id
  AND recipient_id  = :recipient_id
  AND is_deleted    = FALSE
  AND is_read       = FALSE               -- no-op if already read
RETURNING id, is_read, read_at;
```

**Redis update (run after successful SQL):**
```
HINCRBY notif:unread:<recipient_id> total -1
HINCRBY notif:unread:<recipient_id> <type> -1
```

---

### 4.4 `POST /notifications/read-all` — Mark All as Read

```sql
-- Without scope filters
UPDATE notifications
SET
    is_read    = TRUE,
    read_at    = NOW(),
    updated_at = NOW()
WHERE recipient_id  = :recipient_id
  AND is_read       = FALSE
  AND is_deleted    = FALSE;

-- With optional type filter
UPDATE notifications
SET
    is_read    = TRUE,
    read_at    = NOW(),
    updated_at = NOW()
WHERE recipient_id  = :recipient_id
  AND is_read       = FALSE
  AND is_deleted    = FALSE
  AND type          = :type;

-- With optional before filter
UPDATE notifications
SET
    is_read    = TRUE,
    read_at    = NOW(),
    updated_at = NOW()
WHERE recipient_id  = :recipient_id
  AND is_read       = FALSE
  AND is_deleted    = FALSE
  AND created_at   <= :before;
```

**Redis update (run after):**
```
DEL notif:unread:<recipient_id>
-- Counter will be lazily recomputed on next unread-count request
```

---

### 4.5 `DELETE /notifications/:id` — Delete Notification

```sql
-- Soft delete (preferred — preserves audit trail)
UPDATE notifications
SET
    is_deleted = TRUE,
    updated_at = NOW()
WHERE id            = :notification_id
  AND recipient_id  = :recipient_id
  AND is_deleted    = FALSE;
```

---

### 4.6 `GET /notifications/unread-count` — Unread Badge Count

**Step 1 — Try Redis first:**
```
HGETALL notif:unread:<recipient_id>
```

**Step 2 — Cache miss: compute from DB and populate Redis:**
```sql
SELECT
    type,
    COUNT(*) AS unread_count
FROM notifications
WHERE recipient_id  = :recipient_id
  AND is_read       = FALSE
  AND is_deleted    = FALSE
GROUP BY type;
```

Then write to Redis:
```
HSET notif:unread:<recipient_id>
     total   12
     COMMENT  5
     MENTION  3
     SYSTEM   2
     ALERT    2
EXPIRE notif:unread:<recipient_id> 3600
```

---

### 4.7 `GET /notifications/preferences` — Get Preferences

```sql
-- Fetch global preferences
SELECT *
FROM notification_preferences
WHERE user_id = :user_id;

-- Fetch per-type overrides
SELECT notification_type, channel_in_app, channel_email, channel_push, channel_sms
FROM notification_type_preferences
WHERE user_id = :user_id;
```

These two results are merged in the application layer into the preferences JSON shape defined in Stage 1.

---

### 4.8 `PUT /notifications/preferences` — Update Preferences

```sql
-- Upsert global preferences (INSERT or UPDATE if exists)
INSERT INTO notification_preferences (
    user_id,
    channel_in_app, channel_email, channel_push, channel_sms,
    quiet_hours_enabled, quiet_hours_start, quiet_hours_end, quiet_hours_tz,
    digest_enabled, digest_frequency, digest_time,
    updated_at
)
VALUES (
    :user_id,
    :channel_in_app, :channel_email, :channel_push, :channel_sms,
    :quiet_hours_enabled, :quiet_hours_start, :quiet_hours_end, :quiet_hours_tz,
    :digest_enabled, :digest_frequency, :digest_time,
    NOW()
)
ON CONFLICT (user_id) DO UPDATE SET
    channel_in_app      = EXCLUDED.channel_in_app,
    channel_email       = EXCLUDED.channel_email,
    channel_push        = EXCLUDED.channel_push,
    channel_sms         = EXCLUDED.channel_sms,
    quiet_hours_enabled = EXCLUDED.quiet_hours_enabled,
    quiet_hours_start   = EXCLUDED.quiet_hours_start,
    quiet_hours_end     = EXCLUDED.quiet_hours_end,
    quiet_hours_tz      = EXCLUDED.quiet_hours_tz,
    digest_enabled      = EXCLUDED.digest_enabled,
    digest_frequency    = EXCLUDED.digest_frequency,
    digest_time         = EXCLUDED.digest_time,
    updated_at          = NOW();

-- Upsert per-type overrides (called once per type in the request body)
INSERT INTO notification_type_preferences (
    user_id, notification_type,
    channel_in_app, channel_email, channel_push, channel_sms,
    updated_at
)
VALUES (
    :user_id, :notification_type,
    :channel_in_app, :channel_email, :channel_push, :channel_sms,
    NOW()
)
ON CONFLICT (user_id, notification_type) DO UPDATE SET
    channel_in_app  = EXCLUDED.channel_in_app,
    channel_email   = EXCLUDED.channel_email,
    channel_push    = EXCLUDED.channel_push,
    channel_sms     = EXCLUDED.channel_sms,
    updated_at      = NOW();
```

---

## 5. Storage Architecture Summary

```
┌─────────────────────────────────────────────────────────┐
│                    REST API Layer                        │
└───────────┬─────────────────────────┬───────────────────┘
            │                         │
            ▼                         ▼
    ┌───────────────┐         ┌───────────────┐
    │     Redis     │         │  PostgreSQL   │
    │               │         │               │
    │ • Unread      │         │ • notifications│
    │   counters    │         │   (partitioned)│
    │ • SSE session │         │ • preferences │
    │   registry    │         │ • type_prefs  │
    └───────────────┘         └───────────────┘
                                      │
                              ┌───────┴────────┐
                              │  Monthly       │
                              │  Partitions    │
                              │  2026_06       │
                              │  2026_07  ...  │
                              └────────────────┘
```

---

*Document version: 2.0 — June 9, 2026*

---

---

# Stage 3: Query Analysis, Indexing Strategy & Optimization

> **Context:** The notifications table has grown to **50,000 students** and **50 lakh (5,000,000) notifications**. A query written three months ago to fetch unread notifications per student is now performing slowly. This section diagnoses the problem, fixes it, and answers related indexing and query design questions.

---

## 1. The Original Query

```sql
Select * from notifications where student_id=1042 and isread=false orderby createdAt desc;
```

---

## 2. Is This Query Accurate?

**Mostly yes — but it has two syntax errors that would prevent it from running at all on a strict SQL engine:**

| Issue | In the query | Correct syntax |
|-------|-------------|----------------|
| Missing space in `ORDER BY` | `orderby` | `ORDER BY` |
| Wrong column name casing | `createdAt` | `created_at` (assuming snake_case schema as defined in Stage 2) |

**Corrected query:**

```sql
SELECT * FROM notifications
WHERE student_id = 1042
  AND isread = false
ORDER BY created_at DESC;
```

This is logically correct — it fetches all unread notifications for a specific student, newest first. However, correctness and performance are two different things, which brings us to the next problem.

---

## 3. Why Is This Query Slow?

With 5,000,000 rows in the table, this query is slow for the following reasons:

### Reason 1: Full Table Scan (No Index)

If no index exists on `student_id` or `isread`, PostgreSQL has no choice but to scan **every single row** in the table — all 5 million of them — to find the ones matching `student_id = 1042 AND isread = false`. This is called a **Sequential Scan (Seq Scan)**.

```
Seq Scan on notifications
  Filter: ((student_id = 1042) AND (isread = false))
  Rows removed by filter: ~4,999,950
```

It reads 5M rows to return perhaps 30. That is the core problem.

### Reason 2: `SELECT *` Fetches Every Column

`SELECT *` forces PostgreSQL to read every column for every matching row — including wide columns like `body` (TEXT), `metadata` (JSONB), and `resource_url`. This increases I/O significantly, especially when the API only needs a subset of fields to render the notification list.

### Reason 3: `ORDER BY created_at DESC` Without a Supporting Index

Even after filtering, sorting the result set requires PostgreSQL to load all matching rows into memory and sort them. Without an index that already provides the data in sorted order, this becomes an in-memory sort operation — expensive as the unread count per student grows.

### Reason 4: No LIMIT Clause

The query fetches **all** unread notifications for the student in one shot. A student could have hundreds of unread notifications. The API only displays 20 at a time — fetching all of them is wasteful.

---

## 4. What Should You Change?

### Fix 1: Add a Composite Index

Create an index that covers all three operations in the query — filter by `student_id`, filter by `isread`, and sort by `created_at`:

```sql
CREATE INDEX idx_notifications_student_unread
ON notifications (student_id, isread, created_at DESC)
WHERE isread = false;
```

This is a **partial composite index** — the `WHERE isread = false` clause means the index only stores unread rows. This makes the index smaller, faster to build, and faster to search because read notifications (the majority over time) are excluded entirely.

**What PostgreSQL does with this index:**

```
Index Scan using idx_notifications_student_unread on notifications
  Index Cond: ((student_id = 1042) AND (isread = false))
  -- Already sorted by created_at DESC — no separate sort step needed
```

It jumps directly to student 1042's unread rows, already in the right order. Zero full table scan.

### Fix 2: Select Only Required Columns

Replace `SELECT *` with only the columns the frontend actually needs:

```sql
SELECT
    id, type, priority, title, body,
    actor_id, actor_name, actor_avatar_url,
    resource_type, resource_id, resource_url,
    created_at, expires_at
FROM notifications
WHERE student_id = 1042
  AND isread = false
ORDER BY created_at DESC
LIMIT 20;
```

This reduces the data read from disk and sent over the network on every request.

### Fix 3: Add a LIMIT Clause

Always paginate. Never fetch all rows in one query:

```sql
-- Page 1
SELECT id, type, priority, title, body, ...
FROM notifications
WHERE student_id = 1042
  AND isread = false
ORDER BY created_at DESC
LIMIT 20 OFFSET 0;

-- Better: cursor-based (as discussed in Stage 2)
SELECT id, type, priority, title, body, ...
FROM notifications
WHERE student_id = 1042
  AND isread = false
  AND created_at < :last_seen_cursor
ORDER BY created_at DESC
LIMIT 20;
```

### The Fully Optimized Query

```sql
SELECT
    id, type, priority, channel,
    title, body,
    actor_id, actor_name, actor_avatar_url,
    resource_type, resource_id, resource_url,
    metadata, created_at, expires_at
FROM notifications
WHERE student_id  = 1042
  AND isread      = false
ORDER BY created_at DESC
LIMIT 20;
```

---

## 5. Computational Cost: Before vs. After

| Metric | Original Query | Optimized Query |
|--------|---------------|-----------------|
| **Scan type** | Sequential Scan (all 5M rows) | Index Scan (~30 rows for that student) |
| **Rows examined** | ~5,000,000 | ~20–50 |
| **Sort operation** | In-memory sort of all matching rows | Pre-sorted by index — no sort needed |
| **Columns fetched** | All columns including heavy TEXT/JSONB | Only required columns |
| **Rows returned** | All unread (unbounded) | 20 (paginated) |
| **Time complexity** | O(n) — linear in table size | O(log n) — index B-tree lookup |
| **Typical execution time** | 800ms–3s at 5M rows | 1ms–5ms |

The index reduces the query from **O(n)** linear scan to **O(log n)** B-tree traversal — performance stops degrading as the table grows.

---

## 6. Should You Add Indexes on Every Column?

**No. This is a common misconception and would actively harm performance.**

Here is why indexing every column is a bad idea:

### It Slows Down Every Write Operation

Every `INSERT`, `UPDATE`, and `DELETE` on the `notifications` table must also update **every index** on that table. With 10+ indexes, inserting one notification becomes 10+ index update operations. On a high-volume notification system receiving thousands of inserts per minute, this write overhead compounds severely.

```
-- Without indexes: 1 write operation
INSERT INTO notifications (...) VALUES (...);

-- With 10 indexes: 11 write operations
INSERT INTO notifications (...) VALUES (...);
-- + update idx_1, idx_2, idx_3 ... idx_10
```

### Most Indexes Would Never Be Used

Columns like `actor_avatar_url`, `resource_url`, `body`, and `title` are never used in `WHERE` clauses or `JOIN` conditions. An index on them wastes disk space and write performance with zero read benefit. PostgreSQL's query planner would ignore them entirely.

### Indexes Consume Significant Disk Space

Each index is a separate B-tree data structure on disk. At 5M rows, a single index can consume hundreds of megabytes. Indexing every column could multiply storage requirements by 5–10x.

### The Right Approach: Index by Query Pattern

Only add an index when you have a concrete query that needs it. The decision checklist:

| Question | If Yes → |
|----------|----------|
| Is this column used in a `WHERE` clause frequently? | Consider indexing |
| Is this column used in `ORDER BY` on large result sets? | Consider indexing |
| Is this column used in a `JOIN` condition? | Consider indexing |
| Is the column high-cardinality (many distinct values)? | Index is more effective |
| Is this a write-heavy table? | Be conservative — fewer indexes |

**Indexes to create for this system (and why):**

```sql
-- 1. Core read query: filter by student + read status + sort
CREATE INDEX idx_notifications_student_unread
ON notifications (student_id, isread, created_at DESC)
WHERE isread = false;

-- 2. Filter by notification type (placement queries, etc.)
CREATE INDEX idx_notifications_student_type
ON notifications (student_id, notification_type, created_at DESC);

-- 3. Expiry cleanup job
CREATE INDEX idx_notifications_expires_at
ON notifications (expires_at)
WHERE expires_at IS NOT NULL;
```

**Indexes NOT to create:**
- On `body`, `title`, `actor_avatar_url`, `resource_url` — never queried in WHERE clauses
- On `actor_name` — low selectivity, not filtered
- On `metadata` (JSONB GIN index) — only if you have specific JSONB key queries, not by default

---

## 7. Placement Notifications in the Last 7 Days

Find all students who received a placement notification in the last 7 days.

### Query

```sql
SELECT DISTINCT student_id
FROM notifications
WHERE notification_type = 'placement'
  AND created_at >= NOW() - INTERVAL '7 days';
```

### With Student Details (if a `students` table exists)

```sql
SELECT
    s.id            AS student_id,
    s.name          AS student_name,
    s.email,
    COUNT(n.id)     AS placement_notification_count,
    MAX(n.created_at) AS latest_notification_at
FROM notifications n
JOIN students s ON s.id = n.student_id
WHERE n.notification_type = 'placement'
  AND n.created_at >= NOW() - INTERVAL '7 days'
GROUP BY s.id, s.name, s.email
ORDER BY latest_notification_at DESC;
```

### With Full Notification Details Per Student

```sql
SELECT
    n.student_id,
    n.id            AS notification_id,
    n.title,
    n.body,
    n.created_at
FROM notifications n
WHERE n.notification_type = 'placement'
  AND n.created_at >= NOW() - INTERVAL '7 days'
ORDER BY n.student_id, n.created_at DESC;
```

### Supporting Index for This Query

```sql
CREATE INDEX idx_notifications_type_created
ON notifications (notification_type, created_at DESC);
```

This index allows PostgreSQL to jump directly to all `placement` rows and then scan only those within the last 7 days — without touching unrelated notification types at all.

**Without this index:** PostgreSQL scans all 5M rows, filters by type and date.
**With this index:** PostgreSQL reads only placement rows in the last 7 days — a tiny fraction of the table.

---

## 8. Stage 3 Summary

| Question | Answer |
|----------|--------|
| Is the original query accurate? | Syntactically broken (`orderby`, wrong casing). Logically correct but severely unoptimized. |
| Why is it slow? | Full table scan across 5M rows, no index, `SELECT *`, no LIMIT |
| What to change? | Partial composite index, select specific columns, add LIMIT/pagination |
| Computational cost improvement | O(n) → O(log n); from ~2s to ~2ms |
| Index every column? | No — destroys write performance, wastes disk, most indexes unused |
| Correct indexing strategy | Index only columns used in WHERE, ORDER BY, and JOIN — by query pattern |

---

*Document version: 3.0 — June 9, 2026*

---

---

# Stage 4: Performance at Scale — Caching, Load Reduction & Delivery Strategies

> **Context:** Notifications are being fetched from the database on every page load for every student. With 50,000 students and 5,000,000 notifications, the database is overwhelmed, causing slow response times and poor user experience. This section diagnoses the architectural problem and proposes a layered set of solutions with honest tradeoff analysis for each.

---

## 1. Diagnosing the Root Problem

Fetching from the database on every page load is the core architectural mistake. Here is what happens at scale:

```
50,000 students × average 5 page loads/hour
= 250,000 DB queries/hour
= ~70 queries/second sustained
```

During peak hours (morning login, placement result announcements) this spikes to several hundred queries per second. Each query hits the disk, competes for DB connections, and adds latency. The database — designed for persistence, not high-frequency reads — becomes the bottleneck.

The solution is not to make the database faster. The solution is to **stop asking the database the same question repeatedly**.

---

## 2. Strategy 1: Server-Side Caching with Redis

### What It Is

Instead of querying PostgreSQL on every request, the application checks Redis first. Redis stores the result of recent notification queries in memory. A DB query only happens on a cache miss.

```
Request → Check Redis
              │
      ┌───────┴────────┐
   HIT (fast)       MISS (slow)
      │                │
  Return cached    Query PostgreSQL
  response         → Store in Redis
  (~1ms)           → Return response
                   (~50ms, once)
```

### Implementation

```
Cache key:   notif:list:<student_id>:unread:page:1
Cache value: JSON array of notification objects
TTL:         60 seconds
```

On every write operation (new notification inserted, marked as read, deleted), the relevant cache key is invalidated:

```
INSERT notification for student_id=1042
  → DEL notif:list:1042:*          (invalidate all pages for this student)
  → HINCRBY notif:unread:1042 total 1

PATCH /notifications/:id/read
  → DEL notif:list:1042:*
  → HINCRBY notif:unread:1042 total -1
```

### Tradeoffs

| Benefit | Cost |
|---------|------|
| Dramatically reduces DB load — cache hit rate of 80–95% is realistic | Adds Redis as an infrastructure dependency to operate and monitor |
| Sub-millisecond response on cache hits | Cache invalidation logic must be maintained — bugs here cause stale data shown to users |
| Scales horizontally — multiple app servers share one Redis | Students could briefly see slightly outdated notifications within the TTL window |
| Unread count badge becomes nearly free (HGETALL on Redis) | Memory cost — Redis stores data in RAM, which is expensive at high volume |

### When Stale Data Is Acceptable

For notification lists, a 30–60 second TTL is generally acceptable — if a notification arrives and the student sees it 45 seconds later instead of instantly, the SSE stream (from Stage 1) will have already pushed it to the UI in real time. The REST API cache is for the initial page load, not for live delivery.

---

## 3. Strategy 2: Client-Side Caching with HTTP Cache Headers

### What It Is

The server instructs the browser to cache the notification response locally. On the next page load, the browser serves the cached response without making a network request at all.

### Implementation

Add these response headers to `GET /notifications`:

```http
HTTP/1.1 200 OK
Cache-Control: private, max-age=30
ETag: "a3f8b2c1d4e5f6a7"
Last-Modified: Tue, 09 Jun 2026 09:00:00 GMT
```

On the next request, the browser sends:

```http
GET /notifications?status=unread HTTP/1.1
If-None-Match: "a3f8b2c1d4e5f6a7"
```

If nothing has changed, the server responds with `304 Not Modified` and an empty body — saving bandwidth and DB load. The browser uses its cached copy.

For the ETag, compute a hash of the student's latest `updated_at` timestamp:

```sql
SELECT MAX(updated_at) FROM notifications
WHERE student_id = :student_id
  AND isread = false;
```

Hash that value → use as ETag. This is a lightweight query compared to fetching all notifications.

### Tradeoffs

| Benefit | Cost |
|---------|------|
| Zero server load on cache hit — request never reaches the application | Only works for the same student on the same browser/device |
| No extra infrastructure required — built into HTTP | `private` cache means CDNs cannot cache it (notifications are user-specific) |
| Reduces network bandwidth significantly | Requires ETag computation logic on the server |
| Works automatically with the browser's native cache | Browser cache can be cleared by the user, defeating the strategy |
| Complements server-side caching — both can work together | `max-age=30` means UI could be 30 seconds stale without a revalidation request |

---

## 4. Strategy 3: Stop Fetching on Every Page Load — Event-Driven Updates

### What It Is

The most impactful change is architectural: **stop polling the database on page load entirely**. Instead, load notifications once on login and keep them updated in real time via the SSE stream already designed in Stage 1.

### Current (Broken) Pattern

```
Page Load → GET /notifications (DB hit)
Page Load → GET /notifications (DB hit)
Page Load → GET /notifications (DB hit)
... repeated indefinitely
```

### Proposed Pattern

```
Login → GET /notifications (one DB hit, result cached in app state)
      → Open SSE /notifications/stream

New notification arrives via SSE
  → Prepend to local state (no DB call)

Student marks as read
  → PATCH /notifications/:id/read (one DB write)
  → Update local state optimistically

Page navigation
  → Serve from local in-memory state (zero DB calls)
```

### Implementation on the Frontend

Maintain a client-side notification store (e.g. Redux, Zustand, or a React context) that is populated once on login and updated only via SSE events and explicit user actions:

```javascript
// On login: single fetch, store in memory
const { data } = await fetch('/v1/notifications?per_page=50');
notificationStore.set(data);

// On SSE event: update store in memory, no new API call
eventSource.addEventListener('notification.new', (e) => {
  const notification = JSON.parse(e.data);
  notificationStore.prepend(notification);   // zero DB calls
});

// On page navigation: read from store, zero API calls
function renderNotificationBell() {
  return notificationStore.getUnreadCount();  // instant, in-memory
}
```

### Tradeoffs

| Benefit | Cost |
|---------|------|
| Eliminates repeated DB hits entirely for page navigation | Requires frontend state management discipline — store must be kept consistent |
| Notifications feel instant — SSE delivers in under 100ms | If SSE connection drops, store becomes stale until reconnect |
| Reduces server load by the largest margin of any strategy | Initial page load is slightly heavier (fetching 50 notifications upfront) |
| Better UX — no loading spinner on every navigation | Multi-tab consistency requires care — tab A marking as read must update tab B via SSE |
| Aligns with how modern SPAs (React, Vue) are designed | Requires frontend team to implement and maintain the store logic |

---

## 5. Strategy 4: Pagination + Lazy Loading (Stop Fetching Everything)

### What It Is

Even when the DB is queried, never fetch all notifications at once. Fetch only what is visible, and load more only when the student scrolls or requests it.

### Implementation

The API already supports pagination (from Stage 1). The frontend must use it:

```
Initial load:  GET /notifications?per_page=20&page=1
Scroll to end: GET /notifications?per_page=20&page=2
Scroll to end: GET /notifications?per_page=20&page=3
```

Better yet — use cursor-based pagination (from Stage 2) so deep pages remain fast:

```
Initial load:  GET /notifications?per_page=20
Next page:     GET /notifications?per_page=20&cursor=<last_id>
```

### Tradeoffs

| Benefit | Cost |
|---------|------|
| Massively reduces data transfer per request | UI must handle incremental loading (infinite scroll or "load more" button) |
| DB queries are cheap — 20 rows vs 500 rows | Total notifications count still requires a COUNT query |
| Works independently of caching — benefits stack | Pagination state must be managed on the frontend |
| Reduces memory pressure on both server and client | Cursor-based pagination makes jumping to a specific page harder |

---

## 6. Strategy 5: Read Replica for Read Queries

### What It Is

PostgreSQL supports streaming replication — a read replica is a live copy of the primary database that receives all writes but handles all read queries. The notification fetch queries are routed to the replica; only writes (insert, update, delete) go to the primary.

```
Write operations  →  Primary DB (PostgreSQL)
                           │
                    Streaming replication
                           │
Read operations   →  Read Replica (PostgreSQL)
```

### Tradeoffs

| Benefit | Cost |
|---------|------|
| Primary DB is completely free from read load | Replication lag — replica can be 10–500ms behind primary |
| Read replica can be scaled independently | Adds infrastructure cost — another database server to operate |
| No application logic changes required — just a connection string change | Application must route reads and writes to different connections |
| Replica can serve as a hot standby for failover | A student who just marked a notification as read might briefly see it as unread on the next fetch (replica lag) |

Replication lag is the key tradeoff to communicate to the team. For notification lists it is generally acceptable — for the unread count badge (where accuracy matters more), continue serving from Redis rather than the replica.

---

## 7. Recommended Implementation Order

Not all strategies should be implemented at once. Apply them in order of impact vs. effort:

| Priority | Strategy | Impact | Effort | When to Apply |
|----------|----------|--------|--------|---------------|
| 1 | **Event-driven updates (SSE + client store)** | Highest | Medium | Immediately — eliminates the root cause |
| 2 | **Redis server-side cache** | High | Medium | Immediately — protects DB from bursts |
| 3 | **Pagination + lazy loading** | High | Low | Immediately — already designed in Stage 1/2 |
| 4 | **HTTP Cache-Control headers** | Medium | Low | Short term — easy win, no new infrastructure |
| 5 | **Read replica** | Medium | High | When DB write load also becomes a bottleneck |

---

## 8. Combined Architecture After All Strategies Applied

```
┌─────────────────────────────────────────────────────────────┐
│                        Student Browser                       │
│                                                             │
│  Notification Store (in-memory)                             │
│  ├── Populated once on login via REST                       │
│  ├── Updated in real time via SSE                           │
│  └── Served instantly on every page navigation              │
└────────────┬────────────────────────┬───────────────────────┘
             │ REST (initial load     │ SSE (real-time
             │ + explicit actions)    │ push events)
             ▼                        ▼
┌─────────────────────────────────────────────────────────────┐
│                      API Server                              │
│                                                             │
│  1. Check Redis cache                                       │
│  2. Cache hit  → return instantly                           │
│  3. Cache miss → query Read Replica → cache result          │
│  4. Writes     → Primary DB → invalidate cache              │
└──────────┬──────────────────────────┬───────────────────────┘
           │ Reads                    │ Writes
           ▼                          ▼
  ┌─────────────────┐      ┌─────────────────────┐
  │   Redis Cache   │      │  PostgreSQL Primary  │
  │                 │      │  (writes only)       │
  │  • Notif lists  │      └──────────┬──────────┘
  │  • Unread counts│                 │ Replication
  └─────────────────┘                 ▼
                             ┌─────────────────────┐
                             │  PostgreSQL Replica  │
                             │  (reads only)        │
                             └─────────────────────┘
```

---

## 9. Stage 4 Summary

| Strategy | Core Idea | Biggest Tradeoff |
|----------|-----------|-----------------|
| Redis server-side cache | Answer from memory, not disk | Cache invalidation complexity |
| HTTP Cache-Control | Browser caches the response | Only works per-device, can go stale |
| Event-driven SSE + client store | Fetch once, update via push | Frontend state management overhead |
| Pagination + lazy loading | Never fetch more than you show | UI must handle incremental loading |
| Read replica | Separate read and write load | Replication lag, extra infrastructure |

The single most impactful change is **stopping the fetch-on-every-page-load pattern** by combining the SSE stream (already designed) with a client-side notification store. Every other strategy reduces DB load at the margins — this one eliminates the root cause.

---

*Document version: 4.0 — June 9, 2026*

---

---

# Stage 5: Bulk Notification Design — Reliability, Failure Handling & Redesign

> **Context:** It is placement season. The HR clicks "Notify All" and 50,000 students must receive an email and an in-app notification simultaneously. The proposed pseudocode implementation has critical shortcomings. This section diagnoses every flaw, answers the design questions, and delivers a redesigned reliable implementation.

---

## 1. The Original Pseudocode

```
function notify_all(student_ids: array, message: string):
    for student_id in student_ids:
        send_email(student_id, message)   # calls Email API
        save_to_db(student_id, message)   # DB insert
        push_to_app(student_id, message)  # SSE push
```

---

## 2. Shortcomings of This Implementation

### Problem 1: Synchronous Loop Over 50,000 Students

The loop processes one student at a time, sequentially. Each iteration makes three blocking calls — an external email API, a database insert, and an SSE push — before moving to the next student.

**Time estimate:**
```
50,000 students
× (email API ~300ms + DB insert ~10ms + SSE push ~5ms)
= 50,000 × 315ms
= ~15,750 seconds ≈ 4.4 hours
```

The HR clicks "Notify All" and waits **four and a half hours** for the function to complete, blocking the entire process in a single thread the entire time. This is completely unacceptable.

---

### Problem 2: No Error Handling — One Failure Corrupts Everything

There is no `try/catch` anywhere. If `send_email` throws an exception for student 5,000, the entire loop crashes. Students 5,001 to 50,000 receive nothing — no email, no DB record, no in-app notification. There is no way to know which students were notified and which were not without manually inspecting logs.

The logs already confirmed this happened: **200 students' emails failed midway** and the system had no recovery mechanism.

---

### Problem 3: DB and Email Are Coupled in the Same Transaction

`save_to_db` and `send_email` are called back to back in the same loop iteration with no coordination. This creates two dangerous inconsistency scenarios:

- **Email sent, DB insert fails:** The student receives an email but there is no notification record in the database. The in-app notification never appears. The student is confused.
- **DB insert succeeds, email fails:** The notification exists in the database but the student never received the email. The system thinks the student was notified; the student was not.

These are **split-brain states** — the email system and the database disagree on what happened.

---

### Problem 4: Email API Rate Limits Will Be Breached

Every email provider (SendGrid, AWS SES, Mailgun) enforces rate limits — typically 100–1,000 emails per second. Hammering the email API with 50,000 requests as fast as the loop runs will trigger rate limit errors (`429 Too Many Requests`), causing widespread failures. The original code has no rate limiting, no backoff, and no retry logic.

---

### Problem 5: SSE Push Is Not Reliable for Bulk Delivery

Calling `push_to_app` per student inside a synchronous loop assumes every student has an active SSE connection at that exact moment. Most students will not — they may be offline, on a different tab, or their connection may have dropped. SSE is a real-time delivery mechanism, not a persistence mechanism. Sending via SSE without saving to the DB first means offline students miss the notification entirely.

---

### Problem 6: No Idempotency — Retrying Causes Duplicates

If the process crashes at student 30,000 and is restarted, the loop starts from the beginning. Students 1–30,000 receive duplicate emails and get duplicate DB records. There is no mechanism to track which students have already been successfully notified.

---

## 3. Should DB Save and Email Send Happen Together?

**No. They must be deliberately decoupled — but coordinated.**

Here is the reasoning:

The database insert and the email send are operations across two different systems — your PostgreSQL database and an external email provider. There is no distributed transaction that spans both. You cannot `COMMIT` a DB row and an email send atomically — one can succeed while the other fails, and there is no rollback for a sent email.

The correct pattern is **"write first, deliver asynchronously"**:

1. **Save to DB first** — this is your source of truth. If the DB write succeeds, the notification officially exists. This operation is fast, reliable, and within your control.
2. **Publish a delivery event to a queue** — do not call the email API directly. Write a job to a message queue (e.g. RabbitMQ, Kafka, Redis Streams). This is also fast and reliable.
3. **A worker reads from the queue and sends the email** — separately, asynchronously, with retry logic, rate limiting, and idempotency checks.

This decoupling means:
- The DB is always the authoritative record of what notifications exist
- Email delivery is best-effort with retries — temporary email API failures do not corrupt the DB
- If the email worker fails, the job stays in the queue and is retried — no data is lost
- The DB and email states can be reconciled at any time by replaying queue events

---

## 4. What Happens to the 200 Failed Emails?

With the original implementation: **they are lost**. No record of the failure, no retry, no way to identify which 200 students were affected without manual log parsing.

With the redesigned implementation:
- Every delivery attempt is recorded in a `notification_delivery_log` table with status `PENDING`, `SENT`, or `FAILED`
- Failed jobs remain in the queue with an exponential backoff retry schedule
- A dead-letter queue (DLQ) catches jobs that fail after maximum retries
- An admin endpoint can query all `FAILED` deliveries and trigger a targeted re-send for exactly those 200 students — no duplicates to the 49,800 who succeeded

---

## 5. Redesigned Implementation

### 5.1 Architecture Overview

```
HR clicks "Notify All"
        │
        ▼
  API Handler
  ├── Validate request
  ├── INSERT bulk_notification_job (DB)
  └── Enqueue 50,000 jobs → Message Queue
        │
        ▼
  Job Queue (e.g. RabbitMQ / Redis Streams)
  [job_1: student_1] [job_2: student_2] ... [job_50000: student_50000]
        │
        ▼
  Worker Pool (N parallel workers)
  Each worker:
  ├── Dequeue one job
  ├── Check idempotency (already processed?)
  ├── save_to_db(student_id, message)
  ├── send_email(student_id, message)  ← with retry + rate limit
  ├── push_to_app(student_id, message) ← SSE if connected
  └── Mark job DONE / FAILED
        │
        ▼
  Dead Letter Queue (for jobs that fail after max retries)
        │
        ▼
  Admin can inspect + re-trigger failed jobs
```

---

### 5.2 Revised Pseudocode

```
# ─────────────────────────────────────────────
# STEP 1: API Handler — called when HR clicks "Notify All"
# Fast, returns immediately. Does NOT send anything.
# ─────────────────────────────────────────────

function handle_notify_all(student_ids: array, message: string):
    job_id = generate_uuid()

    # Record the bulk job in DB for tracking and audit
    save_bulk_job_to_db(job_id, student_ids.length, status="PENDING")

    # Enqueue one lightweight job per student into the message queue
    # This is fast — just writing job metadata, not sending anything
    for student_id in student_ids:
        enqueue(queue="notifications.bulk", payload={
            job_id:     job_id,
            student_id: student_id,
            message:    message,
            attempt:    1
        })

    # Return immediately to the HR user — do not wait for delivery
    return { job_id: job_id, status: "queued", student_count: 50000 }


# ─────────────────────────────────────────────
# STEP 2: Worker — runs in parallel, N instances
# Each worker independently processes one job at a time
# ─────────────────────────────────────────────

function process_notification_job(job):
    student_id = job.student_id
    message    = job.message
    job_id     = job.job_id

    # Idempotency check — skip if already successfully processed
    # Prevents duplicates on retry or restart
    if already_processed(job_id, student_id):
        ack(job)   # remove from queue, do nothing
        return

    db_saved   = false
    email_sent = false

    # ── Save to DB first (source of truth) ──
    try:
        save_to_db(student_id, message, job_id)
        db_saved = true
    catch DatabaseException as e:
        log_failure(job_id, student_id, "DB_INSERT_FAILED", e)
        nack(job, requeue=true)   # return to queue for retry
        return

    # ── Send email (external, fallible) ──
    try:
        send_email_with_retry(
            student_id = student_id,
            message    = message,
            max_retries = 3,
            backoff     = exponential   # 1s, 2s, 4s
        )
        email_sent = true
    catch RateLimitException as e:
        requeue_with_delay(job, delay_seconds=30)   # back off, try later
        return
    catch EmailApiException as e:
        log_failure(job_id, student_id, "EMAIL_FAILED", e)
        # DB record exists — student will see in-app notification
        # Email failure is recorded; admin can re-trigger

    # ── Push real-time in-app notification (best-effort) ──
    # SSE is fire-and-forget — failure here does not affect DB or email
    try:
        push_to_app(student_id, message)
    catch Exception as e:
        log_warning(job_id, student_id, "SSE_PUSH_FAILED", e)
        # Non-fatal: student will see notification on next page load from DB

    # ── Mark job complete ──
    mark_processed(job_id, student_id,
                   db_saved=db_saved,
                   email_sent=email_sent)
    ack(job)


# ─────────────────────────────────────────────
# STEP 3: Dead Letter Queue Handler
# Called for jobs that have failed max_retries times
# ─────────────────────────────────────────────

function handle_dead_letter(job):
    log_permanent_failure(job.job_id, job.student_id)
    alert_admin(job)   # notify the platform team
    # Job is preserved in DLQ — admin can inspect and manually re-trigger


# ─────────────────────────────────────────────
# STEP 4: Re-send endpoint for failed deliveries
# HR or admin triggers this to retry only the failed students
# ─────────────────────────────────────────────

function retry_failed_emails(job_id: string):
    failed_students = query_db(
        "SELECT student_id FROM notification_delivery_log
         WHERE job_id = :job_id AND email_sent = false"
    )
    # Enqueue only the failed students — no duplicates to successful ones
    for student_id in failed_students:
        enqueue(queue="notifications.bulk", payload={
            job_id:     job_id + "_retry",
            student_id: student_id,
            attempt:    2
        })
    return { requeued: failed_students.length }
```

---

### 5.3 Supporting DB Table: `notification_delivery_log`

```sql
CREATE TABLE notification_delivery_log (
    id              VARCHAR(36)     PRIMARY KEY DEFAULT gen_random_uuid(),
    bulk_job_id     VARCHAR(36)     NOT NULL,
    student_id      VARCHAR(36)     NOT NULL,
    db_saved        BOOLEAN         NOT NULL DEFAULT FALSE,
    email_sent      BOOLEAN         NOT NULL DEFAULT FALSE,
    sse_pushed      BOOLEAN         NOT NULL DEFAULT FALSE,
    failure_reason  TEXT,
    attempt_count   INTEGER         NOT NULL DEFAULT 1,
    processed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ     NOT NULL DEFAULT NOW(),

    UNIQUE (bulk_job_id, student_id)   -- idempotency constraint
);
```

---

## 6. Key Design Principles Applied

| Principle | How It Is Applied |
|-----------|------------------|
| **Write-first** | DB is saved before email is sent — DB is always the source of truth |
| **Async decoupling** | Email sending is offloaded to a queue — the API returns instantly |
| **Idempotency** | `UNIQUE (bulk_job_id, student_id)` prevents duplicate processing on retry |
| **Graceful degradation** | SSE failure is non-fatal; email failure is logged but DB record persists |
| **Retry with backoff** | Email failures retry 3 times with exponential backoff before going to DLQ |
| **Observability** | Every outcome is logged — HR can see exactly how many succeeded and failed |
| **Targeted recovery** | Re-send endpoint retries only the failed 200, not all 50,000 |

---

## 7. Original vs. Redesigned: Side-by-Side

| Dimension | Original | Redesigned |
|-----------|----------|------------|
| Delivery time for 50,000 | ~4.4 hours (sequential) | ~2–5 minutes (parallel workers) |
| HR waits for completion | Yes — request blocks | No — returns instantly with job ID |
| 200 email failures | Lost, unrecoverable | Logged, retryable for exactly those 200 |
| DB + email consistency | Split-brain possible | Write-first guarantees DB is source of truth |
| Duplicate notifications on retry | Yes | No — idempotency key prevents duplicates |
| Rate limit handling | None — crashes | Requeue with delay on 429 |
| Offline students miss in-app | Yes | No — DB record persists, loaded on next visit |
| System observable | No | Yes — per-student delivery status tracked |

---

*Document version: 5.0 — June 9, 2026*
