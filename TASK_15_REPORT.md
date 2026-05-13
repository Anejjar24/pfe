# Task 15 Report — Dockerfiles + Docker Compose (P12)

## Status: DONE

## What Changed and Why

Added production-ready Docker images for both services and wired them into the existing `docker-compose.yml` alongside the infrastructure services already present (postgres, redis, mosquitto).

### New Files

| File | Purpose |
|------|---------|
| `backend/Dockerfile` | Multi-stage build: `node:20-alpine` builder → production image; non-root user |
| `backend/.dockerignore` | Excludes `node_modules`, `dist`, `.env`, logs from build context |
| `frontend/Dockerfile` | Multi-stage build: CRA build with `REACT_APP_*` args → `nginx:1.25-alpine` |
| `frontend/.dockerignore` | Excludes `node_modules`, `build`, `.env`, logs from build context |
| `frontend/nginx.conf` | SPA-aware nginx config: gzip, asset caching, no-cache on `index.html`, security headers |

### Updated Files

**`docker-compose.yml`** — Added `backend` and `frontend` services:
- `backend` waits for `postgres`, `redis`, `mosquitto` via `condition: service_healthy`
- `frontend` waits for `backend` to be started
- All env vars use `${VAR:-default}` so they can be overridden via a `.env` file
- Both services joined to `aquaflow-network`

## Docker Compose Validation

```
docker compose config --quiet
# EXIT:0 — valid (version: warning is cosmetic, ignored by Compose v2)
```

## How to Run the Full Stack

### Option A — All services (production mode)
```bash
cd C:\Users\DELL\Downloads\pfe-project

# Build + start everything
docker compose up --build -d

# Watch logs
docker compose logs -f backend frontend

# Verify
curl http://localhost:3001/api/health   # backend
curl http://localhost:3000              # frontend (nginx)
```

### Option B — Infrastructure only (for local development)
```bash
# Start only postgres + redis + mosquitto
docker compose up postgres redis mosquitto -d

# Then run backend and frontend locally:
# backend: cd backend && npm run start:dev
# frontend: cd frontend && npm start
```

### Option C — Override API URL (e.g. if backend is on different host)
```bash
REACT_APP_API_URL=http://myserver:3001/api \
REACT_APP_WS_URL=http://myserver:3001 \
docker compose up --build frontend -d
```

### Option D — Use a .env file for secrets
Create `C:\Users\DELL\Downloads\pfe-project\.env`:
```env
JWT_SECRET=my-super-secret-32-chars-minimum
JWT_REFRESH_SECRET=my-refresh-secret-32-chars
FRONTEND_URL=http://localhost:3000
REACT_APP_API_URL=http://localhost:3001/api
REACT_APP_WS_URL=http://localhost:3001
```
Then: `docker compose up --build -d`

## Service Map After Task 15

| Service | Container | Port | Depends on |
|---------|-----------|------|------------|
| PostgreSQL | aquaflow-postgres | 5432 | — |
| Redis | aquaflow-redis | 6379 | — |
| Mosquitto | aquaflow-mosquitto | 1883, 9001 | — |
| Backend (NestJS) | aquaflow-backend | 3001 | postgres ✓, redis ✓, mosquitto ✓ |
| Frontend (nginx) | aquaflow-frontend | 3000→80 | backend |

## Build Notes

- **Backend**: production deps only (`npm ci --only=production`); `--legacy-peer-deps` used due to the cache-manager peer dep conflict
- **Frontend**: `REACT_APP_API_URL` and `REACT_APP_WS_URL` baked at build time via `ARG`/`ENV` — override with `docker compose build --build-arg REACT_APP_API_URL=...` or via compose `args:`
- **nginx**: serves from port 80 (mapped to host 3000); gzip on; `index.html` never cached; hashed JS/CSS cached 1 year

## Diff Summary

```
backend/Dockerfile          [NEW]
backend/.dockerignore       [NEW]
frontend/Dockerfile         [NEW]
frontend/.dockerignore      [NEW]
frontend/nginx.conf         [NEW]
docker-compose.yml          [UPDATED — added backend and frontend services]
```
