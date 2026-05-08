# AquaFlow: Project Setup Guide

## Overview

This guide provides step-by-step instructions to set up the development environment and prepare the project for Phase 1 implementation.

---

## Prerequisites

### System Requirements
- **Node.js**: 18.x or higher
- **npm**: 9.x or higher
- **Docker**: 20.10+
- **Docker Compose**: 2.x+
- **Git**: Latest version
- **RAM**: Minimum 4GB (8GB recommended)
- **Disk Space**: Minimum 5GB free

### Verify Prerequisites

```powershell
# Check Node.js version
node --version

# Check npm version
npm --version

# Check Docker version
docker --version

# Check Docker Compose version
docker-compose --version

# Check Git version
git --version
```

---

## Step 1: Initial Project Preparation

### 1.1 Initialize Git (if not already done)

```powershell
cd C:\Users\DELL\Downloads\pfe-project

# Initialize git repository
git init

# Create .gitignore
@"
# Dependencies
node_modules/
package-lock.json
yarn.lock

# Build artifacts
build/
dist/
backend/dist/
frontend/build/

# Environment variables
.env
.env.local
.env.*.local

# IDE
.vscode/
.idea/
*.swp
*.swo
*~

# OS
.DS_Store
Thumbs.db

# Logs
logs/
*.log
npm-debug.log*
yarn-debug.log*

# Database
*.sqlite
*.sqlite3

# Docker
.docker/

# Cache
.cache/
.eslintcache

# Testing
coverage/
.nyc_output/
"@ | Out-File -Encoding UTF8 -FilePath ".gitignore"

# Add files
git add .
git commit -m "Initial AquaFlow project setup

Co-Authored-By: Oz <oz-agent@warp.dev>"
```

### 1.2 Create Directory Structure

```powershell
# Create backend directories
New-Item -ItemType Directory -Path "backend/src/database/entities" -Force
New-Item -ItemType Directory -Path "backend/src/database/migrations" -Force
New-Item -ItemType Directory -Path "backend/src/database/seeds" -Force
New-Item -ItemType Directory -Path "backend/src/common/types" -Force
New-Item -ItemType Directory -Path "backend/src/common/guards" -Force
New-Item -ItemType Directory -Path "backend/src/common/decorators" -Force
New-Item -ItemType Directory -Path "backend/src/auth/strategies" -Force
New-Item -ItemType Directory -Path "backend/src/auth/dto" -Force
New-Item -ItemType Directory -Path "backend/config" -Force

# Create frontend directories
New-Item -ItemType Directory -Path "frontend/src/modules" -Force
New-Item -ItemType Directory -Path "frontend/src/store/slices" -Force
New-Item -ItemType Directory -Path "frontend/src/components/Common" -Force
New-Item -ItemType Directory -Path "frontend/src/components/DataDisplay" -Force
New-Item -ItemType Directory -Path "frontend/src/components/Forms" -Force
New-Item -ItemType Directory -Path "frontend/src/components/Layout" -Force
New-Item -ItemType Directory -Path "frontend/src/styles" -Force
```

---

## Step 2: Backend Setup

### 2.1 Install Backend Dependencies

```powershell
cd backend

# Install production dependencies
npm install @nestjs/common@^10.4.15
npm install @nestjs/core@^10.4.15
npm install @nestjs/platform-express@^10.4.15
npm install @nestjs/websockets@^10.4.15
npm install @nestjs/platform-socket.io@^10.4.15
npm install @nestjs/jwt@^11.0.0
npm install @nestjs/passport@^10.0.3
npm install typeorm@^0.3.17
npm install pg@^8.11.3
npm install mqtt@^5.3.1
npm install bcrypt@^5.1.1
npm install passport-jwt@^4.0.1
npm install class-validator@^0.14.1
npm install class-transformer@^0.5.1
npm install reflect-metadata@^0.2.2
npm install rxjs@^7.8.1

# Install dev dependencies
npm install -D typescript@^5.7.2
npm install -D @types/node@^20.17.10
npm install -D @types/bcrypt@^5.0.2
npm install -D @nestjs/cli@^10.4.9
npm install -D @nestjs/schematics@^10.2.3
npm install -D eslint@^8.56.0
npm install -D @typescript-eslint/eslint-plugin@^6.19.1
npm install -D @typescript-eslint/parser@^6.19.1

cd ..
```

### 2.2 Create Backend Configuration Files

**backend/tsconfig.json**:
```json
{
  "compilerOptions": {
    "module": "commonjs",
    "target": "ES2021",
    "lib": ["ES2021"],
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "declaration": true,
    "allowSyntheticDefaultImports": true,
    "experimentalDecorators": true,
    "emitDecoratorMetadata": true,
    "moduleResolution": "node"
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist", "test", "**/*spec.ts"]
}
```

### 2.3 Create Environment File

**backend/.env**:
```env
# Database
DATABASE_HOST=localhost
DATABASE_PORT=5432
DATABASE_NAME=aquaflow
DATABASE_USER=postgres
DATABASE_PASSWORD=postgres
DATABASE_URL=postgresql://postgres:postgres@localhost:5432/aquaflow

# JWT
JWT_SECRET=your-development-secret-key-change-in-production
JWT_EXPIRATION=3600
JWT_REFRESH_SECRET=your-refresh-secret-key
JWT_REFRESH_EXPIRATION=604800

# MQTT
MQTT_BROKER_URL=mqtt://localhost:1883
MQTT_USERNAME=admin
MQTT_PASSWORD=password
MQTT_PROTOCOL=mqtt
MQTT_PORT=1883

# Server
PORT=3000
NODE_ENV=development

# Frontend
FRONTEND_URL=http://localhost:3000

# Socket.io
SOCKET_PORT=3001

# Mail (optional)
MAIL_HOST=smtp.gmail.com
MAIL_PORT=587
MAIL_USER=your-email@gmail.com
MAIL_PASS=your-app-password
MAIL_FROM=noreply@aquaflow.local
```

---

## Step 3: Frontend Setup

### 3.1 Install Frontend Dependencies

```powershell
cd frontend

# Install core dependencies
npm install react@18.2.0
npm install react-dom@18.2.0
npm install react-router-dom@6.21.1

# Install state management
npm install @reduxjs/toolkit@1.9.7
npm install react-redux@8.1.3

# Install styling
npm install tailwindcss@3.4.1
npm install -D postcss@8.4.32
npm install -D autoprefixer@10.4.17

# Install UI/Chart libraries
npm install recharts@2.10.4
npm install framer-motion@10.16.16
npm install react-leaflet@4.2.3
npm install leaflet@1.9.4

# Install real-time
npm install socket.io-client@4.7.2

# Install HTTP client
npm install axios@1.6.4

# Install forms & validation
npm install react-hook-form@7.50.0
npm install zod@3.22.4

# Install dev dependencies
npm install -D @types/react@18.2.43
npm install -D @types/react-dom@18.2.17
npm install -D vite@5.0.8
npm install -D @vitejs/plugin-react@4.2.1

cd ..
```

### 3.2 Create Frontend Environment File

**frontend/.env**:
```env
REACT_APP_API_URL=http://localhost:3000/api
REACT_APP_WS_URL=ws://localhost:3001
REACT_APP_ENVIRONMENT=development
```

### 3.3 Configure TailwindCSS

**frontend/tailwind.config.js**:
```javascript
module.exports = {
  content: [
    "./index.html",
    "./src/**/*.{js,jsx,ts,tsx}",
  ],
  theme: {
    extend: {
      colors: {
        primary: '#3b82f6',
        secondary: '#0f766e',
      },
    },
  },
  plugins: [],
}
```

---

## Step 4: Docker Setup

### 4.1 Create Docker Compose File

**docker-compose.yml**:
```yaml
version: '3.8'

services:
  postgres:
    image: postgres:15-alpine
    container_name: aquaflow_postgres
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: aquaflow
    ports:
      - '5432:5432'
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - aquaflow

  redis:
    image: redis:7-alpine
    container_name: aquaflow_redis
    ports:
      - '6379:6379'
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - aquaflow

  mosquitto:
    image: eclipse-mosquitto:latest
    container_name: aquaflow_mqtt
    ports:
      - '1883:1883'
      - '9001:9001'
    volumes:
      - mosquitto_data:/mosquitto/data
      - mosquitto_logs:/mosquitto/log
    environment:
      - MOSQUITTO_USERNAME=admin
      - MOSQUITTO_PASSWORD=password
    healthcheck:
      test: ["CMD", "mosquitto_sub", "-h", "localhost", "-t", "$SYS/#", "-C", "1"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - aquaflow

volumes:
  postgres_data:
    driver: local
  redis_data:
    driver: local
  mosquitto_data:
    driver: local
  mosquitto_logs:
    driver: local

networks:
  aquaflow:
    driver: bridge
```

### 4.2 Start Docker Services

```powershell
# Navigate to project root
cd C:\Users\DELL\Downloads\pfe-project

# Start services
docker-compose up -d

# Verify services are running
docker-compose ps

# View logs
docker-compose logs -f

# Stop services (when needed)
# docker-compose down
```

---

## Step 5: Verify Setup

### 5.1 Test Backend Installation

```powershell
cd backend

# Test TypeScript compilation
npx tsc --version

# Verify NestJS CLI
npx @nestjs/cli@latest --version

cd ..
```

### 5.2 Test Frontend Installation

```powershell
cd frontend

# Verify React installation
npm list react

# Verify Node version compatibility
node --version

cd ..
```

### 5.3 Test Database Connection

```powershell
# Test PostgreSQL connection
docker-compose exec postgres psql -U postgres -d aquaflow -c "SELECT version();"

# Test Redis connection
docker-compose exec redis redis-cli ping

# Test MQTT connection
# Use an MQTT client tool or subscribe to test topic
docker-compose exec mosquitto mosquitto_sub -h localhost -t test/topic
```

---

## Step 6: Start Development Servers

### 6.1 Terminal 1: Backend

```powershell
cd backend
npm run start:dev
```

### 6.2 Terminal 2: Frontend

```powershell
cd frontend
npm start
```

### 6.3 Access Application

- **Frontend**: http://localhost:3000
- **Backend API**: http://localhost:3000/api
- **WebSocket**: ws://localhost:3001
- **PostgreSQL**: localhost:5432
- **Redis**: localhost:6379
- **MQTT Broker**: localhost:1883

---

## Step 7: Initial Data Setup

### 7.1 Create Database Seed Script

**backend/src/database/seeds/seed.ts**:
```typescript
import { NestFactory } from '@nestjs/core';
import { AppModule } from '../../app.module';

async function seed() {
  const app = await NestFactory.createApplicationContext(AppModule);
  
  // TODO: Add seeding logic here
  // Create default users, stations, sensors, etc.
  
  await app.close();
}

seed()
  .then(() => console.log('Database seeded successfully'))
  .catch((error) => {
    console.error('Seeding failed:', error);
    process.exit(1);
  });
```

### 7.2 Add Seed Script to package.json

```json
{
  "scripts": {
    "seed": "ts-node src/database/seeds/seed.ts"
  }
}
```

---

## Step 8: IDE Configuration

### 8.1 VS Code Extensions (Recommended)

- ES7+ React/Redux/React-Native snippets
- Prettier - Code formatter
- ESLint
- Thunder Client (for API testing)
- Docker

### 8.2 VS Code Settings

**backend/.vscode/settings.json**:
```json
{
  "editor.defaultFormatter": "esbenp.prettier-vscode",
  "editor.formatOnSave": true,
  "[typescript]": {
    "editor.defaultFormatter": "esbenp.prettier-vscode"
  }
}
```

---

## Troubleshooting

### Port Already in Use

```powershell
# Find process using port 3000 (backend)
Get-NetTCPConnection -LocalPort 3000 | Select-Object OwningProcess

# Find process using port 3001 (frontend)
Get-NetTCPConnection -LocalPort 3001 | Select-Object OwningProcess

# Kill process (replace PID with actual process ID)
Stop-Process -Id <PID> -Force
```

### Docker Issues

```powershell
# Check Docker daemon
docker info

# Rebuild containers
docker-compose down
docker-compose up -d --build

# View container logs
docker-compose logs <service-name>

# Connect to container
docker-compose exec <service-name> sh
```

### Dependencies Installation Fails

```powershell
# Clear npm cache
npm cache clean --force

# Delete node_modules and reinstall
rm -r node_modules package-lock.json
npm install
```

### Database Connection Error

```powershell
# Check if PostgreSQL is running
docker-compose ps postgres

# Verify connection string
docker-compose exec postgres psql -U postgres -c "\l"

# Check environment variables
cat backend/.env | grep DATABASE
```

---

## Next Steps

After completing setup:

1. ✅ Verify all services are running
2. ✅ Test database connectivity
3. ✅ Review AQUAFLOW_ARCHITECTURE.md
4. ✅ Create initial TypeORM entities
5. ✅ Implement authentication module (Phase 1)
6. ✅ Follow IMPLEMENTATION_ROADMAP.md Phase 1 tasks

---

## Development Workflow

### Daily Development

```powershell
# 1. Start Docker services
docker-compose up -d

# 2. Start backend development server
cd backend && npm run start:dev

# 3. In another terminal, start frontend
cd frontend && npm start

# 4. Access application
# Frontend: http://localhost:3000
# API: http://localhost:3000/api
```

### Making Changes

```powershell
# Frontend changes: Auto-reload with npm start
# Backend changes: Auto-reload with npm run start:dev
# Database schema: Create migration, run with npm run typeorm migration:run

# Commit changes
git add .
git commit -m "feat: description

Co-Authored-By: Oz <oz-agent@warp.dev>"
```

### Stopping Development

```powershell
# Stop development servers (Ctrl+C in each terminal)

# Stop Docker services
docker-compose down

# Optional: Remove volumes to reset database
# docker-compose down -v
```

---

## Performance Tips

- Use Redux DevTools for state debugging
- Monitor API response times in browser Network tab
- Check Docker container resource usage: `docker stats`
- Use browser Performance tab for frontend optimization
- Monitor database queries with TypeORM logging enabled

---

## Security Notes (Development Only)

- ⚠️ Change JWT_SECRET before production
- ⚠️ Change MQTT credentials before production
- ⚠️ Never commit .env files
- ⚠️ Use HTTPS in production
- ⚠️ Enable CORS only for trusted domains in production

---

## Getting Help

- Review QUICK_START.md for code examples
- Check AQUAFLOW_ARCHITECTURE.md for design patterns
- Review backend/src/ structure in ARCHITECTURE.md
- Check docker-compose logs for service errors

