# Phase 1: Core Infrastructure Implementation Checklist

**Duration**: 3 weeks (Weeks 1-3)  
**Team Size**: 1-2 developers  
**Goal**: Establish database persistence, authentication, real-time infrastructure, and IoT integration

---

## Pre-Implementation Setup

### Environment Preparation
- [ ] Review PROJECT_SETUP.md
- [ ] Install all prerequisites (Node.js 18+, Docker, Git)
- [ ] Clone/initialize Git repository
- [ ] Create .gitignore file
- [ ] Create directory structure for backend and frontend
- [ ] Set up IDE with recommended extensions

### Initial Commit
- [ ] Create initial git commit with project structure
- [ ] Commit .gitignore
- [ ] Commit documentation files

**Status**: _____  
**Completed by**: _____  
**Date**: _____

---

## Task Group 1.1: Database Setup (Days 1-2)

### Installation & Configuration
- [ ] Install TypeORM: `npm install typeorm`
- [ ] Install PostgreSQL driver: `npm install pg`
- [ ] Install @nestjs/typeorm: `npm install @nestjs/typeorm`
- [ ] Create `backend/src/database/database.module.ts`
- [ ] Create `backend/src/database/database.service.ts`
- [ ] Configure TypeORM in environment variables
- [ ] Create PostgreSQL configuration in docker-compose.yml
- [ ] Start PostgreSQL container: `docker-compose up -d postgres`
- [ ] Verify database connection

### Entity Creation
- [ ] Create `backend/src/database/entities/User.entity.ts`
  - [ ] Properties: id, email, password, firstname, lastname, role
  - [ ] Relationships: stations, maintenances
  - [ ] Timestamps: createdAt, updatedAt
  
- [ ] Create `backend/src/database/entities/Station.entity.ts`
  - [ ] Properties: id, name, location, coordinates, capacity, status
  - [ ] Relationships: sensors, alerts, createdBy
  - [ ] Enums: StationStatus
  
- [ ] Create `backend/src/database/entities/Sensor.entity.ts`
  - [ ] Properties: id, name, type, unit, thresholds, location
  - [ ] Relationships: station, sensorData, alerts
  - [ ] Enums: SensorType
  
- [ ] Create `backend/src/database/entities/SensorData.entity.ts`
  - [ ] Properties: id, value, timestamp, quality_flags
  - [ ] Relationships: sensor
  
- [ ] Create `backend/src/database/entities/Alert.entity.ts`
  - [ ] Properties: id, type, severity, message, status
  - [ ] Relationships: sensor, station
  - [ ] Timestamps: createdAt, acknowledgedAt
  
- [ ] Create `backend/src/database/entities/Maintenance.entity.ts`
  - [ ] Properties: id, type, status, description
  - [ ] Relationships: station, assignedTo
  - [ ] Timestamps: startedAt, completedAt
  
- [ ] Create `backend/src/database/entities/Workflow.entity.ts`
  - [ ] Properties: id, name, description, graph (JSON), isActive
  - [ ] Relationships: createdBy
  - [ ] Timestamps: createdAt, updatedAt
  
- [ ] Create `backend/src/database/entities/Notification.entity.ts`
  - [ ] Properties: id, type, status, recipient, content
  - [ ] Relationships: alert
  - [ ] Timestamps: sentAt, deliveredAt

### TypeORM Configuration
- [ ] Test connection with migrations command
- [ ] Create migration infrastructure
- [ ] Generate initial migration
- [ ] Run migration successfully
- [ ] Verify tables created in PostgreSQL
- [ ] Create seed script skeleton
- [ ] Test entity relationships

**Status**: _____  
**Completed by**: _____  
**Date**: _____  
**Notes**: _____

---

## Task Group 1.2: JWT Authentication (Days 3-4)

### Dependencies
- [ ] Install @nestjs/jwt: `npm install @nestjs/jwt`
- [ ] Install @nestjs/passport: `npm install @nestjs/passport`
- [ ] Install passport-jwt: `npm install passport-jwt`
- [ ] Install bcrypt: `npm install bcrypt`
- [ ] Install @types/bcrypt: `npm install -D @types/bcrypt`

### Authentication Module Structure
- [ ] Create `backend/src/auth/auth.module.ts`
- [ ] Create `backend/src/auth/auth.service.ts`
- [ ] Create `backend/src/auth/auth.controller.ts`

### Auth Service Implementation
- [ ] Implement register() method
  - [ ] Hash password with bcrypt
  - [ ] Create user entity
  - [ ] Save to database
  - [ ] Return success message
  
- [ ] Implement login() method
  - [ ] Find user by email
  - [ ] Verify password
  - [ ] Generate JWT token
  - [ ] Return token and user data
  
- [ ] Implement refreshToken() method
  - [ ] Verify refresh token validity
  - [ ] Generate new access token
  - [ ] Return new token

### JWT Strategy
- [ ] Create `backend/src/auth/strategies/jwt.strategy.ts`
  - [ ] Extract token from Authorization header
  - [ ] Validate token signature
  - [ ] Load user from database
  - [ ] Return user object

- [ ] Create `backend/src/auth/strategies/local.strategy.ts`
  - [ ] For future local authentication

- [ ] Create `backend/src/auth/strategies/refresh-token.strategy.ts`
  - [ ] Validate refresh tokens

### DTOs
- [ ] Create `backend/src/auth/dto/login.dto.ts`
  - [ ] email, password fields
  - [ ] Validation rules
  
- [ ] Create `backend/src/auth/dto/register.dto.ts`
  - [ ] email, password, firstname, lastname
  - [ ] Validation rules
  
- [ ] Create `backend/src/auth/dto/refresh-token.dto.ts`
  - [ ] refresh_token field

### Controller Endpoints
- [ ] POST /api/auth/register
  - [ ] Accepts RegisterDto
  - [ ] Returns success message
  - [ ] Handles duplicate email error
  
- [ ] POST /api/auth/login
  - [ ] Accepts LoginDto
  - [ ] Returns access_token and user data
  - [ ] Handles invalid credentials
  
- [ ] POST /api/auth/refresh
  - [ ] Accepts RefreshTokenDto
  - [ ] Returns new access_token
  - [ ] Validates refresh token
  
- [ ] POST /api/auth/logout
  - [ ] Clears token (frontend responsibility)
  - [ ] Returns success message

### Testing
- [ ] Test register endpoint with valid data
- [ ] Test register endpoint with invalid email
- [ ] Test login with correct credentials
- [ ] Test login with wrong password
- [ ] Test token refresh
- [ ] Verify JWT contains user info
- [ ] Test token expiration

**Status**: _____  
**Completed by**: _____  
**Date**: _____  
**Notes**: _____

---

## Task Group 1.3: JWT Guard & RBAC (Days 5-6)

### Guards Creation
- [ ] Create `backend/src/common/guards/JwtGuard.ts`
  - [ ] Extends AuthGuard('jwt')
  - [ ] Validates JWT on protected routes
  
- [ ] Create `backend/src/common/guards/RolesGuard.ts`
  - [ ] Check user role against required roles
  - [ ] Extract roles from decorators
  - [ ] Allow/deny access
  
- [ ] Create `backend/src/common/guards/OptionalJwtGuard.ts`
  - [ ] Allow requests with or without JWT

### Decorators
- [ ] Create `backend/src/common/decorators/Roles.decorator.ts`
  - [ ] Accepts role array parameter
  - [ ] Sets metadata on handler
  
- [ ] Create `backend/src/common/decorators/CurrentUser.decorator.ts`
  - [ ] Extracts user from request
  - [ ] Returns user object

### User Roles Enum
- [ ] Create `backend/src/database/entities/User.entity.ts` updates
  - [ ] Add UserRole enum: ADMIN, OPERATOR, TECHNICIAN, ANALYST
  - [ ] Set default role on creation

### Route Protection
- [ ] Apply JwtGuard to protected routes
- [ ] Test protected route access with token
- [ ] Test protected route rejection without token
- [ ] Test role-based access control

### Update Auth Service
- [ ] Set default role to OPERATOR on registration
- [ ] Include role in JWT payload

**Status**: _____  
**Completed by**: _____  
**Date**: _____  
**Notes**: _____

---

## Task Group 1.4: WebSocket Real-Time Infrastructure (Days 7-8)

### Dependencies
- [ ] Install @nestjs/websockets: `npm install @nestjs/websockets`
- [ ] Install @nestjs/platform-socket.io: `npm install @nestjs/platform-socket.io`
- [ ] Install socket.io: included with above

### Gateway Creation
- [ ] Create `backend/src/realtime/realtime.gateway.ts`
  - [ ] @WebSocketGateway decorator with CORS
  - [ ] OnGatewayConnection implementation
  - [ ] OnGatewayDisconnect implementation
  - [ ] @SubscribeMessage decorators for events

### Realtime Service
- [ ] Create `backend/src/realtime/realtime.service.ts`
  - [ ] addConnection(clientId, socket) method
  - [ ] removeConnection(clientId) method
  - [ ] broadcastToRoom(room, event, data) method
  - [ ] broadcastToAll(event, data) method

### Realtime Module
- [ ] Create `backend/src/realtime/realtime.module.ts`
  - [ ] Import WebSocketGateway
  - [ ] Register RealtimeService
  - [ ] Export for other modules

### Event Types
- [ ] Create `backend/src/realtime/events/sensor-update.event.ts`
  - [ ] Fields: sensorId, value, timestamp
  
- [ ] Create `backend/src/realtime/events/alert.event.ts`
  - [ ] Fields: alertId, severity, message, timestamp
  
- [ ] Create `backend/src/realtime/events/station-status.event.ts`
  - [ ] Fields: stationId, status, timestamp
  
- [ ] Create `backend/src/realtime/events/workflow-event.ts`
  - [ ] Fields: workflowId, status, executionId

### Frontend Socket Integration
- [ ] Create `frontend/src/hooks/useSocket.js`
  - [ ] Initialize socket connection
  - [ ] Handle connection/disconnection
  - [ ] Emit events
  - [ ] Subscribe to events
  - [ ] Auto-reconnect logic
  
- [ ] Create Socket.io client setup in environment
  - [ ] WS_URL from .env
  - [ ] Auth token handling

### Redux Integration
- [ ] Create `frontend/src/store/slices/realtimeSlice.js`
  - [ ] socket state
  - [ ] subscriptions state
  - [ ] updates state
  - [ ] Reducers for updates
  
- [ ] Update Redux store to include realtime slice

### Testing
- [ ] Test socket connection with JWT token
- [ ] Test sensor-update event broadcasting
- [ ] Test alert-created event broadcasting
- [ ] Test multiple client connections
- [ ] Test client disconnection handling
- [ ] Test room-based subscriptions

**Status**: _____  
**Completed by**: _____  
**Date**: _____  
**Notes**: _____

---

## Task Group 1.5: MQTT Integration (Days 9-10)

### Dependencies
- [ ] Install mqtt: `npm install mqtt`

### MQTT Client Wrapper
- [ ] Create `backend/src/iot/mqtt/mqtt.client.ts`
  - [ ] Connect to MQTT broker
  - [ ] Handle connection events
  - [ ] Subscribe to sensor topics
  - [ ] Publish messages
  - [ ] Disconnect gracefully

### MQTT Configuration
- [ ] Create `backend/src/iot/mqtt/mqtt.config.ts`
  - [ ] Load from environment variables
  - [ ] Broker URL, username, password
  - [ ] Subscribe patterns

### Message Handlers
- [ ] Create `backend/src/iot/handlers/sensor-message.handler.ts`
  - [ ] Parse MQTT payload
  - [ ] Extract sensor ID and value
  - [ ] Validate data
  - [ ] Save to database
  - [ ] Trigger alert if threshold exceeded
  
- [ ] Create `backend/src/iot/handlers/device-event.handler.ts`
  - [ ] Handle device status messages
  - [ ] Update device status in database

### IoT Module
- [ ] Create `backend/src/iot/iot.module.ts`
  - [ ] Register MQTT client
  - [ ] Register handlers
  
- [ ] Create `backend/src/iot/iot.service.ts`
  - [ ] Initialize MQTT client on module startup
  - [ ] Handle incoming messages
  - [ ] Coordinate with other services

### Integration with Realtime
- [ ] Update RealtimeGateway to broadcast sensor updates
- [ ] Emit sensor-update event on MQTT message
- [ ] Emit alert event if threshold violation

### Configuration
- [ ] Add MQTT settings to docker-compose.yml
- [ ] Start Mosquitto container: `docker-compose up -d mosquitto`
- [ ] Verify MQTT connectivity
- [ ] Configure MQTT credentials

### Testing
- [ ] Publish test message to MQTT topic
- [ ] Verify sensor data saved to database
- [ ] Verify WebSocket broadcast on update
- [ ] Test threshold alert triggering
- [ ] Test multiple sensor messages

**Status**: _____  
**Completed by**: _____  
**Date**: _____  
**Notes**: _____

---

## Task Group 1.6: Redux State Management (Days 11-12)

### Redux Store Setup
- [ ] Create `frontend/src/store/store.js`
  - [ ] configureStore configuration
  - [ ] Reducer registration
  - [ ] Middleware setup
  
- [ ] Create `frontend/src/store/hooks.js`
  - [ ] useAppDispatch hook
  - [ ] useAppSelector hook

### Redux Slices

#### Auth Slice
- [ ] Create `frontend/src/store/slices/authSlice.js`
  - [ ] State: user, token, isAuthenticated, loading, error
  - [ ] Actions: login, logout, register, refreshToken
  - [ ] Async thunks for API calls
  - [ ] Selectors for common queries

#### Stations Slice
- [ ] Create `frontend/src/store/slices/stationsSlice.js`
  - [ ] State: stations, selectedStation, loading, error
  - [ ] Async thunks: fetchStations, createStation, updateStation
  - [ ] Reducers for state updates
  - [ ] Selectors

#### Sensors Slice
- [ ] Create `frontend/src/store/slices/sensorsSlice.js`
  - [ ] State: sensors, sensorData, loading
  - [ ] Async thunks: fetchSensors, fetchSensorData
  - [ ] Reducers for updates
  
#### Alerts Slice
- [ ] Create `frontend/src/store/slices/alertsSlice.js`
  - [ ] State: alerts, acknowledged, filters
  - [ ] Actions: acknowledgeAlert, filterAlerts
  - [ ] Selectors
  
#### Maintenance Slice
- [ ] Create `frontend/src/store/slices/maintenanceSlice.js`
  - [ ] State: interventions, technicians, loading
  - [ ] Async thunks
  - [ ] Actions
  
#### UI Slice
- [ ] Create `frontend/src/store/slices/uiSlice.js`
  - [ ] State: theme, sidebarOpen, notifications
  - [ ] Actions: toggleTheme, toggleSidebar, showNotification

#### Realtime Slice
- [ ] Create `frontend/src/store/slices/realtimeSlice.js`
  - [ ] State: socket, subscriptions, updates
  - [ ] Actions: setSocket, subscribeSensor, updateSensor

### Integration
- [ ] Configure Redux store in `frontend/src/main.jsx`
- [ ] Wrap App with Provider
- [ ] Test store with Redux DevTools

### Testing
- [ ] Verify store initialization
- [ ] Test slice reducers
- [ ] Test async thunks
- [ ] Test selectors
- [ ] Verify DevTools integration

**Status**: _____  
**Completed by**: _____  
**Date**: _____  
**Notes**: _____

---

## Task Group 1.7: App Module Integration (Days 13)

### Update Backend app.module.ts
- [ ] Import DatabaseModule
- [ ] Import AuthModule
- [ ] Import RealtimeModule
- [ ] Import IoTModule
- [ ] Register all modules

### Update Backend main.ts
- [ ] Initialize NestJS application
- [ ] Start server on PORT from .env
- [ ] Enable CORS from FRONTEND_URL
- [ ] Initialize WebSocket adapter
- [ ] Initialize MQTT client
- [ ] Log startup information

### Create Root Controller (Optional)
- [ ] Create `backend/src/app.controller.ts`
  - [ ] GET / endpoint for health check
  - [ ] Returns API version and status

### Testing
- [ ] Start backend server: `npm run start:dev`
- [ ] Verify all modules load
- [ ] Test health check endpoint
- [ ] Verify no compilation errors
- [ ] Check logs for startup messages

**Status**: _____  
**Completed by**: _____  
**Date**: _____  
**Notes**: _____

---

## Task Group 1.8: Final Verification (Day 14)

### Database
- [ ] [ ] All entities created
- [ ] [ ] All relationships configured
- [ ] [ ] Migrations run successfully
- [ ] [ ] Sample data seeded (optional)
- [ ] [ ] PostgreSQL container running

### Authentication
- [ ] [ ] Register endpoint works
- [ ] [ ] Login endpoint works
- [ ] [ ] JWT tokens generated correctly
- [ ] [ ] Token validation works
- [ ] [ ] Role-based guards protect routes

### Real-Time
- [ ] [ ] WebSocket gateway running
- [ ] [ ] Socket connections established
- [ ] [ ] Events broadcast correctly
- [ ] [ ] Redux updates on events
- [ ] [ ] Multiple clients connect simultaneously

### IoT/MQTT
- [ ] [ ] MQTT broker running
- [ ] [ ] Sensor messages received
- [ ] [ ] Data saved to database
- [ ] [ ] WebSocket broadcasts on data
- [ ] [ ] Alert triggers on threshold

### Redux (Frontend)
- [ ] [ ] Store configured correctly
- [ ] [ ] All slices registered
- [ ] [ ] DevTools working
- [ ] [ ] Async thunks callable
- [ ] [ ] State updates propagate

### Documentation
- [ ] [ ] Code comments added
- [ ] [ ] README.md updated
- [ ] [ ] Architecture documented
- [ ] [ ] API documented

### Git
- [ ] [ ] Code committed with messages
- [ ] [ ] .gitignore configured
- [ ] [ ] No sensitive data in repo
- [ ] [ ] Clean git history

**Status**: _____  
**Completed by**: _____  
**Date**: _____  
**Final Notes**: _____

---

## Phase 1 Deliverables Checklist

### Backend
- [ ] Database module with TypeORM and PostgreSQL
- [ ] 8 core entities with relationships
- [ ] Authentication service with JWT
- [ ] JWT strategy and guards
- [ ] RBAC decorators and guards
- [ ] WebSocket gateway for real-time events
- [ ] MQTT client integration
- [ ] IoT message handlers
- [ ] Realtime service for broadcasting
- [ ] Updated app.module.ts
- [ ] Environment configuration
- [ ] Working development server

### Frontend
- [ ] Redux store with 7 slices
- [ ] Socket.io integration with hooks
- [ ] Custom hooks (useAuth, useSocket, etc.)
- [ ] API client wrapper
- [ ] Environment configuration
- [ ] Component structure ready for Phase 2
- [ ] Working development server

### Infrastructure
- [ ] Docker Compose with PostgreSQL, Redis, Mosquitto
- [ ] All containers running
- [ ] Environment files (.env)
- [ ] Health checks passing

### Documentation
- [ ] Code well-commented
- [ ] API endpoints documented
- [ ] Database schema explained
- [ ] Setup guide completed
- [ ] Troubleshooting guide created

### Testing
- [ ] All endpoints tested
- [ ] Database operations verified
- [ ] Real-time events working
- [ ] MQTT integration confirmed
- [ ] Redux state management functional

---

## Success Criteria

Upon Phase 1 completion, verify:

- ✅ Backend API runs without errors
- ✅ Frontend connects to backend
- ✅ WebSocket connection established
- ✅ MQTT messages processed
- ✅ Database persists data
- ✅ Authentication flow works end-to-end
- ✅ Real-time updates <500ms latency
- ✅ All services containerized and running
- ✅ Git history clean and organized
- ✅ Ready for Phase 2 implementation

---

## Sign-Off

**Project Manager**: _________________ Date: _______

**Technical Lead**: _________________ Date: _______

**QA Lead**: _________________ Date: _______

---

## Notes for Phase 2

After Phase 1 completion, the following are ready for Phase 2:
- Stable backend with authentication and persistence
- Real-time infrastructure operational
- IoT data pipeline functional
- Redux state management in place
- Docker infrastructure established

**Next Phase Start Date**: _____________

**Expected Phase 2 Duration**: 5 weeks (Weeks 4-8)

