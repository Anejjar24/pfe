# Task 14 Report — Redis Cache (P11)

## Status: DONE

## What Changed and Why

Added `@nestjs/cache-manager` (v1) with `cache-manager@4` + `cache-manager-redis-store@3`.
Implemented two cache use-cases: sensor list caching and refresh token denylist.

### Packages Installed

```bash
npm install @nestjs/cache-manager@1 cache-manager@4 cache-manager-redis-store@3
npm install --save-dev @types/cache-manager
```

### New Files

| File | Purpose |
|------|---------|
| `backend/src/auth/dto/logout.dto.ts` | Optional `refresh_token` field for POST /auth/logout |

### Updated Files

**`backend/src/app.module.ts`**
- Registered `CacheModule.registerAsync({ isGlobal: true })` with conditional logic:
  - If `REDIS_HOST` env var is set → uses `cache-manager-redis-store` with Redis
  - Otherwise → in-memory fallback (TTL 300s, max 1000 entries)
  - No crash when Redis is not running in development

**`backend/src/sensors/sensors.service.ts`**
- Injected `CACHE_MANAGER`
- `findAll(query)` → caches result under `sensors:list:<queryJSON>` for 60 seconds
- `create()`, `update()`, `remove()` → call `clearListCache()` which deletes all tracked keys
- Tracks active cache keys in a `Set<string>` for precise invalidation

**`backend/src/auth/auth.service.ts`**
- Injected `CACHE_MANAGER`
- `logout(user, refreshToken?)` → if refresh token is provided, stores SHA-256 hash in denylist (`rt:deny:<hash>`) with 7-day TTL
- `refreshToken(token)` → checks denylist before verifying; also rotates (denylist old token, issue new pair)

**`backend/src/auth/auth.controller.ts`**
- `POST /auth/logout` now accepts optional `{ refresh_token }` body via `LogoutDto`

**`backend/.env.example`**
- Added commented Redis configuration block:
  ```
  # REDIS_HOST=localhost
  # REDIS_PORT=6379
  # REDIS_PASSWORD=
  ```

## TypeScript Verification

```
cd backend && npx tsc --noEmit
# EXIT:0 — no errors
```

## curl Verification

### Without token → 401
```bash
curl -s -o /dev/null -w "%{http_code}" http://localhost:3000/api/sensors
# Expected: 401
```

### 1. Login
```bash
curl -s -X POST http://localhost:3000/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"admin@aquaflow.io","password":"admin123"}' | jq .
# Save access_token and refresh_token
```

### 2. Sensor list cache — first call (miss, hits DB)
```bash
time curl -s http://localhost:3000/api/sensors \
  -H "Authorization: Bearer ACCESS_TOKEN" | jq '.meta'
```

### 3. Sensor list cache — second call (hit, faster)
```bash
time curl -s http://localhost:3000/api/sensors \
  -H "Authorization: Bearer ACCESS_TOKEN" | jq '.meta'
# Expected: noticeably faster response (served from cache)
```

### 4. Cache invalidation — create a sensor, then re-fetch
```bash
# Create sensor (invalidates cache)
curl -s -X POST http://localhost:3000/api/sensors \
  -H "Authorization: Bearer ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name":"Test","type":"pressure","unit":"bar","stationId":"STATION_ID"}' | jq .id

# Re-fetch (should miss cache and show new sensor)
curl -s http://localhost:3000/api/sensors \
  -H "Authorization: Bearer ACCESS_TOKEN" | jq '.meta.total'
```

### 5. Logout with refresh token → token denylisted
```bash
curl -s -X POST http://localhost:3000/api/auth/logout \
  -H "Authorization: Bearer ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"refresh_token":"REFRESH_TOKEN"}' | jq .
# Expected: { "message": "Logged out successfully" }
```

### 6. Use denylisted refresh token → 401
```bash
curl -s -X POST http://localhost:3000/api/auth/refresh \
  -H "Content-Type: application/json" \
  -d '{"refresh_token":"REFRESH_TOKEN"}' | jq .
# Expected: 401 { "message": "Refresh token has been revoked" }
```

### 7. Normal refresh (before logout) → new token pair
```bash
curl -s -X POST http://localhost:3000/api/auth/refresh \
  -H "Content-Type: application/json" \
  -d '{"refresh_token":"FRESH_REFRESH_TOKEN"}' | jq '{access_token: .access_token[0:20]}'
# Expected: 200 with new access_token + refresh_token
```

### To enable Redis (optional)
```bash
# Add to backend/.env:
REDIS_HOST=localhost
REDIS_PORT=6379
# Restart backend — cache will use Redis automatically
```

## Diff Summary

```
backend/src/auth/dto/logout.dto.ts           [NEW]
backend/src/app.module.ts                    [UPDATED — CacheModule global registration]
backend/src/sensors/sensors.service.ts       [UPDATED — findAll cache + invalidation]
backend/src/auth/auth.service.ts             [UPDATED — refresh token denylist]
backend/src/auth/auth.controller.ts          [UPDATED — logout accepts refresh_token body]
backend/.env.example                         [UPDATED — Redis env vars documented]
```
