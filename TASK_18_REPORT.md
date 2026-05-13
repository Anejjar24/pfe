# Task 18 Report — P15: Testing Infrastructure

## Status: COMPLETE ✅

---

## Test Results

### Backend — 31 unit tests, all passing

```
Test Suites: 3 passed, 3 total
Tests:       31 passed, 31 total
Time:        ~17 s
```

### Frontend — 28 tests, all passing

```
Test Suites: 2 passed, 2 total
Tests:       28 passed, 28 total
Time:        ~6 s
```

---

## What Was Built

### Backend Infrastructure

#### `backend/jest.config.js` (new)
- `ts-jest` transform for `.ts` files, `tsconfig.json` used as-is
- Test match: `src/**/*.spec.ts` (unit) + `test/**/*.e2e-spec.ts` (e2e)
- `moduleNameMapper` for `src/` alias resolution

#### `backend/package.json` (updated)
Four new scripts:
| Script | Description |
|--------|-------------|
| `npm test` | Run all unit tests |
| `npm run test:watch` | Watch mode |
| `npm run test:cov` | Coverage report |
| `npm run test:e2e` | E2E tests only |

New devDependencies installed:
- `jest` + `ts-jest` + `@types/jest` — test runner with TypeScript support
- `supertest` + `@types/supertest` — HTTP integration tests

---

### Backend Unit Tests

#### `backend/src/auth/auth.service.spec.ts` (new) — 13 tests
Tests all critical `AuthService` methods with fully mocked dependencies:

| Group | Tests |
|-------|-------|
| `validateUser` | returns user when active, throws if missing, throws if inactive |
| `login` | returns tokens + strips password, 401 on missing user, 401 on wrong password, 401 on disabled account |
| `register` | 409 on duplicate email, creates user + returns tokens |
| `logout` | returns success message, denylists refresh token |
| `refreshToken` | 401 on denylisted token, rotates tokens + denylists old |

Key pattern: mock cache manager as `jest.fn() as jest.MockedFunction<any>` with default behaviours set in `beforeEach`, allowing per-test overrides.

#### `backend/src/alerts/alerts.service.spec.ts` (new) — 9 tests

| Group | Tests |
|-------|-------|
| `create` | creates alert, broadcasts WS, fires notification (fire-and-forget), survives notification rejection, 404 for bad stationId, attaches station |
| `findAll` | returns paginated response, empty list |
| `findOne` | returns alert, 404 on missing |

Key assertion: even when `notifyAlertCreated` rejects, `create` still resolves — validating the fire-and-forget pattern.

#### `backend/src/notifications/notifications.service.spec.ts` (new) — 9 tests

| Group | Tests |
|-------|-------|
| `notifyAlertCreated` | saves broadcast record, broadcasts WS event, skips email when no SMTP_HOST |
| `getUnreadCount` | returns `{ count: N }` |
| `markRead` | 404 on missing, sets readAt + status = READ |
| `markAllRead` | broadcasts `notifications-read-all`, returns `{ updated: N }` |

---

### Backend E2E Test

#### `backend/test/auth.e2e-spec.ts` (new) — smoke tests
Starts the full NestJS app in-process with `TypeORM DataSource` overridden by an in-memory stub (no real DB needed):

| Test | Expectation |
|------|-------------|
| `POST /auth/login` empty body | 400 |
| `POST /auth/login` invalid email | 400 |
| `POST /auth/login` wrong credentials | 401 |
| `POST /auth/register` missing password | 400 |
| `GET /notifications` without JWT | 401 |
| `GET /alerts` without JWT | 401 |
| `GET /stations` without JWT | 401 |

---

### Frontend Infrastructure

New devDependencies installed:
- `@testing-library/react` — component rendering + queries
- `@testing-library/jest-dom` — custom matchers (`toBeInTheDocument`, etc.)
- `@testing-library/user-event` — user interaction simulation
- `@testing-library/dom` — DOM query utilities (peer dep)

#### `frontend/src/setupTests.js` (new)
```js
import '@testing-library/jest-dom';
```
CRA auto-discovers this file — no `package.json` config needed.

---

### Frontend Tests

#### `frontend/src/store/slices/__tests__/notificationsSlice.test.js` (new) — 17 tests
Pure reducer and selector tests — zero network calls, zero React.

| Group | Tests |
|-------|-------|
| Reducer — initial state | matches expected shape |
| `notificationReceived` | prepends item, increments unreadCount, increments meta.total |
| `allNotificationsCleared` | resets unreadCount to 0, leaves items intact |
| `fetchNotifications` thunk | pending → loading, fulfilled → items+meta, rejected → error |
| `fetchUnreadCount` thunk | fulfilled → sets unreadCount |
| `markNotificationRead` thunk | updates item in list, decrements count, floor at 0 |
| `markAllNotificationsRead` thunk | sets readAt on all items, resets unreadCount |
| Selectors | selectNotifications, selectUnreadCount, selectNotificationsLoading, selectNotificationsMeta |

#### `frontend/src/components/Navbars/__tests__/AdminNavbar.test.jsx` (new) — 11 tests
Component tests with a pre-loaded Redux store stub. Network thunks are mocked at the module boundary so no HTTP calls fire.

| Test | Assertion |
|------|-----------|
| Renders without crashing | ✅ |
| Shows user display name | "Admin User" in DOM |
| No badge when unreadCount = 0 | badge element absent |
| Badge shows correct count | "3" present when unreadCount = 3 |
| Badge caps at "99+" | unreadCount 150 → "99+" |
| "No notifications" when empty | message present |
| Renders notification titles | item titles in DOM |
| "Mark all read" when unread > 0 | link present |
| No "Mark all read" when count = 0 | link absent |
| Per-item "Mark read" only for unread | exactly 1 button |
| "View all notifications" link | always present |

---

## Files Changed / Created

### Backend (5 files)
| Path | Change |
|------|--------|
| `backend/jest.config.js` | **new** |
| `backend/package.json` | updated — test scripts added |
| `backend/src/auth/auth.service.spec.ts` | **new** — 13 unit tests |
| `backend/src/alerts/alerts.service.spec.ts` | **new** — 9 unit tests |
| `backend/src/notifications/notifications.service.spec.ts` | **new** — 9 unit tests |
| `backend/test/auth.e2e-spec.ts` | **new** — 7 e2e smoke tests |

### Frontend (4 files)
| Path | Change |
|------|--------|
| `frontend/src/setupTests.js` | **new** |
| `frontend/src/store/slices/__tests__/notificationsSlice.test.js` | **new** — 17 tests |
| `frontend/src/components/Navbars/__tests__/AdminNavbar.test.jsx` | **new** — 11 tests |

---

## Coverage Summary

| Layer | Test files | Tests | Result |
|-------|-----------|-------|--------|
| Backend unit | 3 | 31 | ✅ All pass |
| Backend e2e | 1 | 7 (run separately) | ✅ suite created |
| Frontend | 2 | 28 | ✅ All pass |
| **Total** | **6** | **59** | **✅** |

---

## Next Step

The core AquaFlow implementation is now complete through Task 18. All planned development tasks (P1–P15) have been delivered.
