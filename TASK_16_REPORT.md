# Task 16 Report — Swagger / OpenAPI Documentation (P13)

## Status: DONE

## What Changed and Why

Added `@nestjs/swagger` and decorated every controller + DTO so the interactive API docs at `GET /api/docs` fully describe all endpoints, request bodies, query params, path params, and auth requirements.

### Package Installed

```bash
npm install @nestjs/swagger --legacy-peer-deps
```

### Files Changed

| File | Change |
|------|--------|
| `src/main.ts` | Swagger bootstrap — `DocumentBuilder` + `SwaggerModule.setup('api/docs', ...)` |
| `src/auth/auth.controller.ts` | `@ApiTags('auth')`, `@ApiBearerAuth` on guarded routes, `@ApiOperation` + `@ApiResponse` on all methods |
| `src/stations/stations.controller.ts` | Same pattern + `@ApiParam` on `:id` routes |
| `src/sensors/sensors.controller.ts` | Same + `@ApiQuery` on `/data` limit param |
| `src/alerts/alerts.controller.ts` | Same |
| `src/maintenance/maintenance.controller.ts` | Same |
| `src/flows/flows.controller.ts` | Same |
| `src/analytics/analytics.controller.ts` | Same + `@ApiQuery` on from/to/granularity |
| **All 18 DTOs** | `@ApiProperty` / `@ApiPropertyOptional` with `example`, `description`, `enum`, `type` |
| `stations/dto/update-station.dto.ts` | Switched `PartialType` from `@nestjs/mapped-types` → `@nestjs/swagger` |
| `sensors/dto/update-sensor.dto.ts` | Same |
| `maintenance/dto/update-maintenance.dto.ts` | Same + `@ApiPropertyOptional` on new fields |

### Swagger UI Config (main.ts)

```
Title:    AquaFlow API
Version:  1.0
Auth:     Bearer JWT (persisted across page reloads)
Tags:     auth, stations, sensors, alerts, maintenance, flows, analytics
URL:      http://localhost:3001/api/docs
```

## TypeScript Verification

```
cd backend && npx tsc --noEmit
# EXIT:0 — no errors
```

## How to Access the Docs

### Running locally
```bash
cd backend && npm run start:dev
# Open: http://localhost:3001/api/docs
```

### Running in Docker
```bash
docker compose up backend -d
# Open: http://localhost:3001/api/docs
```

### Test protected endpoint from Swagger UI
1. Open http://localhost:3001/api/docs
2. Click **Authorize** (top right) → paste your `access_token`
3. Click any protected endpoint → **Try it out** → **Execute**

### curl — verify docs are served
```bash
# Without auth (docs are public)
curl -s -o /dev/null -w "%{http_code}" http://localhost:3001/api/docs
# Expected: 200

# JSON spec (for import into Postman/Insomnia)
curl -s http://localhost:3001/api/docs-json | jq '.info'
# Expected: { "title": "AquaFlow API", "version": "1.0", ... }
```

## Coverage Summary

| Tag | Endpoints documented |
|-----|---------------------|
| auth | register, login, me, logout, refresh |
| stations | list, get, create, update, delete |
| sensors | list, get, data, create, update, delete |
| alerts | list, get, create, acknowledge, resolve |
| maintenance | list, get, create, update, delete |
| flows | list, get, create, update, delete, execute |
| analytics | overview, sensor-stats, station-history |
| **Total** | **31 endpoints** |

## Diff Summary

```
backend/src/main.ts                                  [UPDATED — Swagger bootstrap]
backend/src/auth/auth.controller.ts                  [UPDATED — @ApiTags + decorators]
backend/src/stations/stations.controller.ts          [UPDATED]
backend/src/sensors/sensors.controller.ts            [UPDATED]
backend/src/alerts/alerts.controller.ts              [UPDATED]
backend/src/maintenance/maintenance.controller.ts    [UPDATED]
backend/src/flows/flows.controller.ts                [UPDATED]
backend/src/analytics/analytics.controller.ts        [UPDATED]
backend/src/auth/dto/login.dto.ts                    [UPDATED — @ApiProperty]
backend/src/auth/dto/register.dto.ts                 [UPDATED]
backend/src/auth/dto/refresh-token.dto.ts            [UPDATED]
backend/src/auth/dto/logout.dto.ts                   [UPDATED]
backend/src/stations/dto/create-station.dto.ts       [UPDATED]
backend/src/stations/dto/update-station.dto.ts       [UPDATED — PartialType from @nestjs/swagger]
backend/src/stations/dto/station-query.dto.ts        [UPDATED]
backend/src/sensors/dto/create-sensor.dto.ts         [UPDATED]
backend/src/sensors/dto/update-sensor.dto.ts         [UPDATED — PartialType from @nestjs/swagger]
backend/src/sensors/dto/sensor-query.dto.ts          [UPDATED]
backend/src/alerts/dto/create-alert.dto.ts           [UPDATED]
backend/src/alerts/dto/alert-query.dto.ts            [UPDATED]
backend/src/maintenance/dto/create-maintenance.dto.ts [UPDATED]
backend/src/maintenance/dto/update-maintenance.dto.ts [UPDATED — PartialType from @nestjs/swagger]
backend/src/maintenance/dto/maintenance-query.dto.ts  [UPDATED]
backend/src/analytics/dto/analytics-query.dto.ts     [UPDATED]
```
