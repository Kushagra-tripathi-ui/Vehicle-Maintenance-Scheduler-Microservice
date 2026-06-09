\# Stage 1: Notification System — REST API Design \& Contract



> \*\*Audience:\*\* Frontend Engineers  

> \*\*Purpose:\*\* Defines the full REST API surface for the notification platform — endpoints, JSON schemas, headers, error contracts, and the real-time delivery mechanism.  

> \*\*Base URL:\*\* `https://api.yourapp.com/v1`



\---



\## 1. Core Actions Identified



| # | Action | Description |

|---|--------|-------------|

| 1 | \*\*Fetch notifications\*\* | Retrieve paginated list of notifications for the authenticated user |

| 2 | \*\*Fetch single notification\*\* | Retrieve full detail of one notification by ID |

| 3 | \*\*Mark as read\*\* | Mark one notification as read |

| 4 | \*\*Mark all as read\*\* | Bulk-mark all unread notifications as read |

| 5 | \*\*Delete notification\*\* | Permanently remove a single notification |

| 6 | \*\*Get unread count\*\* | Lightweight poll endpoint for badge counts |

| 7 | \*\*Update preferences\*\* | Set per-channel / per-type delivery preferences |

| 8 | \*\*Get preferences\*\* | Retrieve current notification preferences |

| 9 | \*\*Real-time subscription\*\* | Subscribe to live notification events via SSE |



\---



\## 2. Shared Conventions



\### 2.1 Authentication Header

Every request \*\*must\*\* include:



```

Authorization: Bearer <access\_token>

Content-Type: application/json

Accept: application/json

X-Request-ID: <uuid-v4>          // client-generated; echoed in response for tracing

```



\### 2.2 Standard Response Envelope



All responses are wrapped in a consistent envelope:



```json

{

&#x20; "success": true,

&#x20; "data": { },

&#x20; "meta": { },

&#x20; "error": null

}

```



\### 2.3 Standard Error Shape



```json

{

&#x20; "success": false,

&#x20; "data": null,

&#x20; "meta": null,

&#x20; "error": {

&#x20;   "code": "NOTIFICATION\_NOT\_FOUND",

&#x20;   "message": "The requested notification does not exist.",

&#x20;   "details": \[],

&#x20;   "request\_id": "3f9a1b2c-..."

&#x20; }

}

```



\### 2.4 Common HTTP Status Codes



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



\### 2.5 Notification Object Schema (Canonical)



This is the base shape returned in all notification responses:



```json

{

&#x20; "id": "notif\_01HZ9X4K2P7VBQR3W5FMGD6TN",

&#x20; "type": "COMMENT" | "MENTION" | "SYSTEM" | "ALERT" | "REMINDER" | "PROMOTION",

&#x20; "title": "John Doe commented on your post",

&#x20; "body": "Great article! I totally agree with your point about...",

&#x20; "is\_read": false,

&#x20; "is\_archived": false,

&#x20; "priority": "LOW" | "MEDIUM" | "HIGH" | "URGENT",

&#x20; "channel": "IN\_APP" | "EMAIL" | "PUSH" | "SMS",

&#x20; "actor": {

&#x20;   "id": "usr\_789",

&#x20;   "name": "John Doe",

&#x20;   "avatar\_url": "https://cdn.yourapp.com/avatars/usr\_789.jpg"

&#x20; },

&#x20; "resource": {

&#x20;   "type": "POST" | "COMMENT" | "TASK" | "INVOICE" | "SYSTEM",

&#x20;   "id": "post\_456",

&#x20;   "url": "/posts/456"

&#x20; },

&#x20; "metadata": {},

&#x20; "created\_at": "2026-06-09T08:30:00Z",

&#x20; "read\_at": null,

&#x20; "expires\_at": null

}

```



\---



\## 3. Endpoints



\---



\### 3.1 List Notifications



Fetch a paginated, filterable list of notifications for the logged-in user.



```

GET /notifications

```



\*\*Query Parameters:\*\*



| Parameter | Type | Required | Description |

|-----------|------|----------|-------------|

| `page` | integer | No | Page number, default `1` |

| `per\_page` | integer | No | Items per page, default `20`, max `100` |

| `status` | string | No | `unread` \\| `read` \\| `all` (default `all`) |

| `type` | string | No | Filter by notification type e.g. `COMMENT,MENTION` |

| `priority` | string | No | `LOW` \\| `MEDIUM` \\| `HIGH` \\| `URGENT` |

| `since` | ISO 8601 | No | Only return notifications after this timestamp |



\*\*Request Headers:\*\*

```http

GET /v1/notifications?status=unread\&per\_page=20\&page=1 HTTP/1.1

Authorization: Bearer eyJhbGci...

X-Request-ID: 3f9a1b2c-4d5e-6f7a-8b9c-0d1e2f3a4b5c

Accept: application/json

```



\*\*Response — 200 OK:\*\*

```json

{

&#x20; "success": true,

&#x20; "data": \[

&#x20;   {

&#x20;     "id": "notif\_01HZ9X4K2P7VBQR3W5FMGD6TN",

&#x20;     "type": "COMMENT",

&#x20;     "title": "John Doe commented on your post",

&#x20;     "body": "Great article! I totally agree...",

&#x20;     "is\_read": false,

&#x20;     "is\_archived": false,

&#x20;     "priority": "MEDIUM",

&#x20;     "channel": "IN\_APP",

&#x20;     "actor": {

&#x20;       "id": "usr\_789",

&#x20;       "name": "John Doe",

&#x20;       "avatar\_url": "https://cdn.yourapp.com/avatars/usr\_789.jpg"

&#x20;     },

&#x20;     "resource": {

&#x20;       "type": "POST",

&#x20;       "id": "post\_456",

&#x20;       "url": "/posts/456"

&#x20;     },

&#x20;     "metadata": {},

&#x20;     "created\_at": "2026-06-09T08:30:00Z",

&#x20;     "read\_at": null,

&#x20;     "expires\_at": null

&#x20;   }

&#x20; ],

&#x20; "meta": {

&#x20;   "current\_page": 1,

&#x20;   "per\_page": 20,

&#x20;   "total\_items": 84,

&#x20;   "total\_pages": 5,

&#x20;   "unread\_count": 12

&#x20; },

&#x20; "error": null

}

```



\---



\### 3.2 Get Single Notification



```

GET /notifications/:id

```



\*\*Request Headers:\*\*

```http

GET /v1/notifications/notif\_01HZ9X4K2P7VBQR3W5FMGD6TN HTTP/1.1

Authorization: Bearer eyJhbGci...

X-Request-ID: a1b2c3d4-e5f6-7890-abcd-ef1234567890

Accept: application/json

```



\*\*Response — 200 OK:\*\*

```json

{

&#x20; "success": true,

&#x20; "data": {

&#x20;   "id": "notif\_01HZ9X4K2P7VBQR3W5FMGD6TN",

&#x20;   "type": "COMMENT",

&#x20;   "title": "John Doe commented on your post",

&#x20;   "body": "Great article! I totally agree with your point about distributed systems.",

&#x20;   "is\_read": false,

&#x20;   "is\_archived": false,

&#x20;   "priority": "MEDIUM",

&#x20;   "channel": "IN\_APP",

&#x20;   "actor": {

&#x20;     "id": "usr\_789",

&#x20;     "name": "John Doe",

&#x20;     "avatar\_url": "https://cdn.yourapp.com/avatars/usr\_789.jpg"

&#x20;   },

&#x20;   "resource": {

&#x20;     "type": "POST",

&#x20;     "id": "post\_456",

&#x20;     "url": "/posts/456"

&#x20;   },

&#x20;   "metadata": {

&#x20;     "comment\_preview": "Great article! I totally agree..."

&#x20;   },

&#x20;   "created\_at": "2026-06-09T08:30:00Z",

&#x20;   "read\_at": null,

&#x20;   "expires\_at": null

&#x20; },

&#x20; "meta": null,

&#x20; "error": null

}

```



\*\*Response — 404 Not Found:\*\*

```json

{

&#x20; "success": false,

&#x20; "data": null,

&#x20; "meta": null,

&#x20; "error": {

&#x20;   "code": "NOTIFICATION\_NOT\_FOUND",

&#x20;   "message": "No notification found with the given ID.",

&#x20;   "details": \[],

&#x20;   "request\_id": "a1b2c3d4-..."

&#x20; }

}

```



\---



\### 3.3 Mark Single Notification as Read



```

PATCH /notifications/:id/read

```



\*\*Request Headers:\*\*

```http

PATCH /v1/notifications/notif\_01HZ9X4K2P7VBQR3W5FMGD6TN/read HTTP/1.1

Authorization: Bearer eyJhbGci...

X-Request-ID: b2c3d4e5-f6a7-8901-bcde-f12345678901

Content-Type: application/json

```



\*\*Request Body:\*\* \_(empty — no body required)\_



\*\*Response — 200 OK:\*\*

```json

{

&#x20; "success": true,

&#x20; "data": {

&#x20;   "id": "notif\_01HZ9X4K2P7VBQR3W5FMGD6TN",

&#x20;   "is\_read": true,

&#x20;   "read\_at": "2026-06-09T09:15:22Z"

&#x20; },

&#x20; "meta": null,

&#x20; "error": null

}

```



\---



\### 3.4 Mark All Notifications as Read



```

POST /notifications/read-all

```



\*\*Request Headers:\*\*

```http

POST /v1/notifications/read-all HTTP/1.1

Authorization: Bearer eyJhbGci...

X-Request-ID: c3d4e5f6-a7b8-9012-cdef-123456789012

Content-Type: application/json

```



\*\*Request Body (optional — scope the bulk action):\*\*

```json

{

&#x20; "type": "COMMENT",

&#x20; "before": "2026-06-09T09:00:00Z"

}

```



\*\*Response — 200 OK:\*\*

```json

{

&#x20; "success": true,

&#x20; "data": {

&#x20;   "marked\_read\_count": 12

&#x20; },

&#x20; "meta": null,

&#x20; "error": null

}

```



\---



\### 3.5 Delete Notification



```

DELETE /notifications/:id

```



\*\*Request Headers:\*\*

```http

DELETE /v1/notifications/notif\_01HZ9X4K2P7VBQR3W5FMGD6TN HTTP/1.1

Authorization: Bearer eyJhbGci...

X-Request-ID: d4e5f6a7-b8c9-0123-defa-234567890123

```



\*\*Response — 204 No Content\*\* \_(empty body)\_



\*\*Response — 404 Not Found:\*\*

```json

{

&#x20; "success": false,

&#x20; "data": null,

&#x20; "meta": null,

&#x20; "error": {

&#x20;   "code": "NOTIFICATION\_NOT\_FOUND",

&#x20;   "message": "No notification found with the given ID.",

&#x20;   "details": \[],

&#x20;   "request\_id": "d4e5f6a7-..."

&#x20; }

}

```



\---



\### 3.6 Get Unread Count



Lightweight endpoint designed for badge polling. Returns only the count — no notification payloads.



```

GET /notifications/unread-count

```



\*\*Request Headers:\*\*

```http

GET /v1/notifications/unread-count HTTP/1.1

Authorization: Bearer eyJhbGci...

X-Request-ID: e5f6a7b8-c9d0-1234-efab-345678901234

Accept: application/json

```



\*\*Response — 200 OK:\*\*

```json

{

&#x20; "success": true,

&#x20; "data": {

&#x20;   "unread\_count": 12,

&#x20;   "by\_type": {

&#x20;     "COMMENT": 5,

&#x20;     "MENTION": 3,

&#x20;     "SYSTEM": 2,

&#x20;     "ALERT": 2

&#x20;   }

&#x20; },

&#x20; "meta": {

&#x20;   "last\_checked\_at": "2026-06-09T09:18:00Z"

&#x20; },

&#x20; "error": null

}

```



> \*\*Frontend note:\*\* Poll this endpoint every 30–60 seconds as a fallback when the real-time SSE connection is unavailable. Do \*\*not\*\* poll faster than every 15 seconds.



\---



\### 3.7 Get Notification Preferences



```

GET /notifications/preferences

```



\*\*Request Headers:\*\*

```http

GET /v1/notifications/preferences HTTP/1.1

Authorization: Bearer eyJhbGci...

X-Request-ID: f6a7b8c9-d0e1-2345-fabc-456789012345

Accept: application/json

```



\*\*Response — 200 OK:\*\*

```json

{

&#x20; "success": true,

&#x20; "data": {

&#x20;   "user\_id": "usr\_123",

&#x20;   "channels": {

&#x20;     "in\_app": true,

&#x20;     "email": true,

&#x20;     "push": false,

&#x20;     "sms": false

&#x20;   },

&#x20;   "types": {

&#x20;     "COMMENT": {

&#x20;       "in\_app": true,

&#x20;       "email": true,

&#x20;       "push": false,

&#x20;       "sms": false

&#x20;     },

&#x20;     "MENTION": {

&#x20;       "in\_app": true,

&#x20;       "email": true,

&#x20;       "push": true,

&#x20;       "sms": false

&#x20;     },

&#x20;     "SYSTEM": {

&#x20;       "in\_app": true,

&#x20;       "email": true,

&#x20;       "push": false,

&#x20;       "sms": false

&#x20;     },

&#x20;     "ALERT": {

&#x20;       "in\_app": true,

&#x20;       "email": true,

&#x20;       "push": true,

&#x20;       "sms": true

&#x20;     },

&#x20;     "REMINDER": {

&#x20;       "in\_app": true,

&#x20;       "email": false,

&#x20;       "push": false,

&#x20;       "sms": false

&#x20;     },

&#x20;     "PROMOTION": {

&#x20;       "in\_app": false,

&#x20;       "email": false,

&#x20;       "push": false,

&#x20;       "sms": false

&#x20;     }

&#x20;   },

&#x20;   "quiet\_hours": {

&#x20;     "enabled": true,

&#x20;     "start": "22:00",

&#x20;     "end": "08:00",

&#x20;     "timezone": "Asia/Kolkata"

&#x20;   },

&#x20;   "digest": {

&#x20;     "enabled": false,

&#x20;     "frequency": "DAILY" | "WEEKLY",

&#x20;     "time": "08:00"

&#x20;   },

&#x20;   "updated\_at": "2026-05-01T10:00:00Z"

&#x20; },

&#x20; "meta": null,

&#x20; "error": null

}

```



\---



\### 3.8 Update Notification Preferences



```

PUT /notifications/preferences

```



\*\*Request Headers:\*\*

```http

PUT /v1/notifications/preferences HTTP/1.1

Authorization: Bearer eyJhbGci...

X-Request-ID: a7b8c9d0-e1f2-3456-abcd-567890123456

Content-Type: application/json

```



\*\*Request Body\*\* \_(partial updates supported — send only changed keys)\_:

```json

{

&#x20; "channels": {

&#x20;   "push": true

&#x20; },

&#x20; "types": {

&#x20;   "PROMOTION": {

&#x20;     "in\_app": false,

&#x20;     "email": false,

&#x20;     "push": false,

&#x20;     "sms": false

&#x20;   }

&#x20; },

&#x20; "quiet\_hours": {

&#x20;   "enabled": true,

&#x20;   "start": "23:00",

&#x20;   "end": "07:00",

&#x20;   "timezone": "Asia/Kolkata"

&#x20; }

}

```



\*\*Response — 200 OK:\*\*

```json

{

&#x20; "success": true,

&#x20; "data": {

&#x20;   "updated": true,

&#x20;   "updated\_at": "2026-06-09T09:22:00Z"

&#x20; },

&#x20; "meta": null,

&#x20; "error": null

}

```



\---



\## 4. Real-Time Notification Mechanism



\### 4.1 Chosen Approach: Server-Sent Events (SSE)



| Factor | SSE | WebSocket | Long Polling |

|--------|-----|-----------|--------------|

| Direction | Server → Client only | Bidirectional | Server → Client |

| Complexity | Low | High | Medium |

| Auto-reconnect | Native (browser) | Manual | Manual |

| Works over HTTP/1.1 | Yes | Requires upgrade | Yes |

| Proxy / firewall friendly | Yes | Sometimes blocked | Yes |

| Best for | Notification feeds | Chat / gaming | Legacy fallback |



Notifications are a \*\*one-way\*\* server-push use case — SSE is the right fit.



\---



\### 4.2 SSE Subscription Endpoint



```

GET /notifications/stream

```



\*\*Request Headers:\*\*

```http

GET /v1/notifications/stream HTTP/1.1

Authorization: Bearer eyJhbGci...

Accept: text/event-stream

Cache-Control: no-cache

X-Request-ID: b8c9d0e1-f2a3-4567-bcde-678901234567

```



\*\*Response Headers (server):\*\*

```http

HTTP/1.1 200 OK

Content-Type: text/event-stream

Cache-Control: no-cache

Connection: keep-alive

X-Accel-Buffering: no

Access-Control-Allow-Origin: https://yourapp.com

```



\---



\### 4.3 SSE Event Types \& Payloads



Each SSE message follows the format:



```

event: <event\_type>

id: <event\_id>

data: <json\_payload>

retry: 5000



```

\_(blank line terminates the event)\_



\---



\#### Event: `notification.new`

Fired when a new notification arrives for the user.



```

event: notification.new

id: evt\_01HZ9XABCDEF1234567890

data: {"id":"notif\_01HZ9X4K2P7VBQR3W5FMGD6TN","type":"MENTION","title":"Sarah mentioned you in a comment","body":"Hey @you, what do you think about this approach?","is\_read":false,"priority":"HIGH","actor":{"id":"usr\_456","name":"Sarah Lee","avatar\_url":"https://cdn.yourapp.com/avatars/usr\_456.jpg"},"resource":{"type":"COMMENT","id":"cmt\_789","url":"/posts/123#comment-789"},"created\_at":"2026-06-09T09:25:00Z"}

retry: 5000



```



\---



\#### Event: `notification.read`

Fired when a notification is marked read (e.g. on another device/tab).



```

event: notification.read

id: evt\_01HZ9XABCDEF1234567891

data: {"id":"notif\_01HZ9X4K2P7VBQR3W5FMGD6TN","is\_read":true,"read\_at":"2026-06-09T09:26:00Z"}

retry: 5000



```



\---



\#### Event: `notification.deleted`

Fired when a notification is deleted.



```

event: notification.deleted

id: evt\_01HZ9XABCDEF1234567892

data: {"id":"notif\_01HZ9X4K2P7VBQR3W5FMGD6TN"}

retry: 5000



```



\---



\#### Event: `notification.count`

Periodic count sync to keep badge accurate (emitted every 60 seconds or after any change).



```

event: notification.count

id: evt\_01HZ9XABCDEF1234567893

data: {"unread\_count":11,"by\_type":{"COMMENT":4,"MENTION":4,"SYSTEM":2,"ALERT":1}}

retry: 5000



```



\---



\#### Event: `ping`

Heartbeat emitted every 30 seconds to keep the connection alive through proxies.



```

event: ping

id: evt\_01HZ9XABCDEF1234567894

data: {"timestamp":"2026-06-09T09:25:30Z"}

retry: 5000



```



\---



\### 4.4 Frontend Integration Guide (JavaScript)



```javascript

// 1. Open the SSE stream after login

const token = getAccessToken();



const eventSource = new EventSource(

&#x20; `https://api.yourapp.com/v1/notifications/stream`,

&#x20; {

&#x20;   // NOTE: EventSource does not support custom headers natively.

&#x20;   // Pass token as a query param OR use a wrapper library (see note below).

&#x20;   // Recommended: use the `@microsoft/fetch-event-source` library which

&#x20;   // supports Authorization headers.

&#x20; }

);



// 2. Handle new notifications

eventSource.addEventListener('notification.new', (e) => {

&#x20; const notification = JSON.parse(e.data);

&#x20; displayToast(notification);          // show in-app toast

&#x20; incrementBadgeCount();               // update bell icon

&#x20; prependToNotificationList(notification);

});



// 3. Sync read state across tabs/devices

eventSource.addEventListener('notification.read', (e) => {

&#x20; const { id, read\_at } = JSON.parse(e.data);

&#x20; markNotificationReadInUI(id, read\_at);

&#x20; decrementBadgeCount();

});



// 4. Sync deletions

eventSource.addEventListener('notification.deleted', (e) => {

&#x20; const { id } = JSON.parse(e.data);

&#x20; removeNotificationFromUI(id);

});



// 5. Badge count sync

eventSource.addEventListener('notification.count', (e) => {

&#x20; const { unread\_count } = JSON.parse(e.data);

&#x20; setBadgeCount(unread\_count);

});



// 6. Handle errors + reconnect

eventSource.onerror = (err) => {

&#x20; console.error('SSE connection error:', err);

&#x20; // Browser auto-reconnects after the `retry` delay (5000ms).

&#x20; // Fall back to polling /notifications/unread-count if offline.

};



// 7. Clean up on logout

function onLogout() {

&#x20; eventSource.close();

}

```



> \*\*Token passing note:\*\* Because the browser's native `EventSource` API does not support `Authorization` headers, use one of:

> - \*\*Query parameter:\*\* `?token=<access\_token>` \_(only if the connection is HTTPS and short-lived tokens are used)\_

> - \*\*Cookie-based auth\*\* for the SSE endpoint only

> - \*\*`@microsoft/fetch-event-source`\*\* — a drop-in replacement that supports full header control



\---



\### 4.5 Reconnection \& Fallback Strategy



```

┌─────────────────────────────────────────────────┐

│              Client Startup (logged in)         │

└───────────────────┬─────────────────────────────┘

&#x20;                   │

&#x20;                   ▼

&#x20;         Open SSE /notifications/stream

&#x20;                   │

&#x20;         ┌─────────┴──────────┐

&#x20;         │ Connection OK       │ Connection FAILED

&#x20;         │                     │

&#x20;         ▼                     ▼

&#x20;  Listen for events     Poll GET /notifications/

&#x20;  (live updates)        unread-count every 30s

&#x20;         │                     │

&#x20;         │  Reconnect after     │

&#x20;         └──── retry delay ─────┘

&#x20;              (5 seconds,

&#x20;              browser-native)

```



\---



\## 5. Endpoint Summary



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



\---



\## 6. Design Decisions \& Notes for Frontend



1\. \*\*Optimistic UI:\*\* For `PATCH /:id/read` and `DELETE /:id`, update the UI immediately on user action and roll back only if the API returns a non-2xx response.



2\. \*\*Cursor vs. offset pagination:\*\* This design uses offset/page-based pagination for simplicity. If the notification list grows very large, request a switch to cursor-based pagination (`?cursor=<last\_id>`).



3\. \*\*`X-Request-ID` header:\*\* Always generate and send this UUID per request — it allows backend engineers to trace requests across logs end-to-end. Echo it in bug reports.



4\. \*\*SSE token expiry:\*\* If the access token expires mid-stream, the server will close the connection with `401`. The client should detect this, refresh the token, and re-open the stream.



5\. \*\*Notification TTL:\*\* Notifications with a non-null `expires\_at` should be hidden from the UI after that timestamp without requiring a delete call.



6\. \*\*`metadata` field:\*\* This is a flexible JSON object. Content varies by notification type — consult the backend team for type-specific metadata schemas as they are added.



\---



\*Document version: 1.0 — June 9, 2026\*

s

