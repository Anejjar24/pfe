# Task 17 Report — P14: Notifications Module

## Status: COMPLETE ✅

---

## What Was Built

### Backend

#### `backend/src/notifications/notifications.service.ts` (new)
- `notifyAlertCreated(alert, station?, sensor?)` — creates a `Notification` record, broadcasts `notification-created` WS event to all connected clients, and sends email to all admin users if the alert is critical **and** `SMTP_HOST` is configured. Each per-admin email is wrapped in its own try/catch so a single email failure never aborts the others.
- `findAll(query)` — paginated list with optional `unread=true` filter (`readAt IS NULL`).
- `getUnreadCount()` — returns `{ count: number }` for the navbar badge.
- `markRead(id)` — sets `status = READ` and `readAt = now()`.
- `markAllRead()` — bulk UPDATE + broadcasts `notifications-read-all` WS event.

#### `backend/src/notifications/notifications.controller.ts` (new)
| Method | Route | Description |
|--------|-------|-------------|
| GET | `/notifications` | List with pagination + unread filter |
| GET | `/notifications/unread-count` | Badge count |
| PATCH | `/notifications/read-all` | Mark all read |
| PATCH | `/notifications/:id/read` | Mark one read |

#### `backend/src/notifications/notifications.module.ts` (new)
- `TypeOrmModule.forFeature([Notification, User])`, imports `RealtimeModule`, exports `NotificationsService`.

#### `backend/src/alerts/alerts.service.ts` (updated)
- Injected `NotificationsService`.
- After `save(alert)`: `this.notificationsService.notifyAlertCreated(alert, station, sensor).catch(() => void 0)` — fire-and-forget, never blocks alert creation.

#### `backend/src/alerts/alerts.module.ts` (updated)
- Added `NotificationsModule` to imports array.

---

### Frontend

#### `frontend/src/services/notificationService.js` (new)
Four API helpers using `apiClient`:
- `getNotifications(params)` — `GET /notifications`
- `getUnreadCount()` — `GET /notifications/unread-count`
- `markRead(id)` — `PATCH /notifications/:id/read`
- `markAllRead()` — `PATCH /notifications/read-all`

#### `frontend/src/store/slices/notificationsSlice.js` (new)
**Thunks:** `fetchNotifications`, `fetchUnreadCount`, `markNotificationRead`, `markAllNotificationsRead`

**Reducers (for WebSocket events):**
- `notificationReceived(notification)` — unshift into `items`, increment `unreadCount`
- `allNotificationsCleared()` — reset `unreadCount` to 0

**Selectors:** `selectNotifications`, `selectUnreadCount`, `selectNotificationsLoading`, `selectNotificationsMeta`

#### `frontend/src/store/store.js` (updated)
- Added `notifications: notificationsReducer` to the Redux root reducer.

#### `frontend/src/hooks/useSocket.js` (updated)
Two new WebSocket event handlers wired in:
```js
socket.on('notification-created', (data) => dispatch(notificationReceived(data)));
socket.on('notifications-read-all', ()    => dispatch(allNotificationsCleared()));
```

#### `frontend/src/components/Navbars/AdminNavbar.js` (updated)
- **Bell icon** in the navbar with a live `unreadCount` badge (danger pill, max "99+").
- **Notifications dropdown** (minWidth 340px, max 5 recent items):
  - Header row with badge + "Mark all read" link.
  - Each item shows: severity icon (color-coded), title + message, relative timestamp (`timeAgo`), "Mark read" button for unread items.
  - Unread items highlighted with `bg-lighter`.
  - Footer "View all notifications" link (routes to `/admin/notifications`).
- Dispatches `fetchUnreadCount` + `fetchNotifications({ limit: 8 })` on mount.

---

## Architecture Notes

| Concern | Decision |
|---------|----------|
| Email delivery | Optional — guarded by `SMTP_HOST` env var. No SMTP = no email, no crash. |
| Notification fan-out | Fire-and-forget: `.catch(() => void 0)` so failures never break alert creation. |
| WS broadcast | `RealtimeService.broadcast('notification-created', payload)` — all connected clients. |
| Per-user targeting | `recipient` field on `Notification` entity (value `'all'` for broadcast). Ready to extend to per-user targeting. |
| Badge count | Maintained both from REST (`fetchUnreadCount`) and WS (`notificationReceived` increments locally). |

---

## Build Verification

```
✅ Backend  — npx tsc --noEmit  EXIT:0
✅ Frontend — npx react-scripts build  EXIT:0  (384 kB gzip)
```

---

## Files Changed / Created

### Backend (7 files)
| Path | Change |
|------|--------|
| `backend/src/notifications/notifications.service.ts` | **new** |
| `backend/src/notifications/notifications.controller.ts` | **new** |
| `backend/src/notifications/notifications.module.ts` | **new** |
| `backend/src/alerts/alerts.service.ts` | updated — fire-and-forget notify |
| `backend/src/alerts/alerts.module.ts` | updated — imports NotificationsModule |
| `backend/src/app.module.ts` | updated — registers NotificationsModule |

### Frontend (6 files)
| Path | Change |
|------|--------|
| `frontend/src/services/notificationService.js` | **new** |
| `frontend/src/store/slices/notificationsSlice.js` | **new** |
| `frontend/src/store/store.js` | updated — adds notifications reducer |
| `frontend/src/hooks/useSocket.js` | updated — WS event handlers |
| `frontend/src/components/Navbars/AdminNavbar.js` | updated — bell icon + dropdown |

---

## Next Task

**Task 18 — P15: Testing Infrastructure**
- Backend: `@nestjs/testing`, `jest`, `supertest` — unit tests for services, e2e tests for key endpoints
- Frontend: `@testing-library/react`, `@testing-library/user-event` — component + hook tests
