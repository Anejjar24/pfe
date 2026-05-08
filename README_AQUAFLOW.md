# AquaFlow: Industrial Water Station Supervision Platform

## Executive Summary

**AquaFlow** is a professional industrial SCADA-like platform designed to transform drinking water station supervision through real-time monitoring, intelligent automation, and comprehensive analytics. Built by extending an existing workflow-builder project, AquaFlow preserves the core architecture while adding enterprise-grade features for water facility management.

---

## What is AquaFlow?

AquaFlow is an intelligent supervision and analytics platform inspired by SCADA systems and industrial IoT dashboards. It provides:

### Core Capabilities
- **Real-Time Monitoring**: Live sensor data from water treatment facilities
- **IoT Integration**: MQTT-based sensor ingestion from pumps, pressure gauges, flow meters
- **Intelligent Alerts**: Threshold-based and anomaly-detection alerts
- **Maintenance Management**: Intervention tracking, technician assignment, history
- **Visual Workflows**: Drag-and-drop automation builder for industrial logic
- **GIS Visualization**: Station mapping with live status indicators
- **Advanced Analytics**: Trend analysis, anomaly detection, KPI calculation
- **Reporting**: PDF/Excel reports with customizable templates
- **User Management**: Role-based access control (Admin, Operator, Technician, Analyst)

### Design Philosophy
- **Preserve Existing**: Keep the current workflow builder and extend it
- **Enterprise-Ready**: Production-grade security, performance, scalability
- **Modular Architecture**: Feature-based modules for easy maintenance
- **Real-Time First**: WebSocket-powered live updates throughout
- **Data-Driven**: PostgreSQL persistence, comprehensive audit trails

---

## Architecture Overview

### Three-Tier Architecture

```
┌─────────────────────────────────────────────────────┐
│                    Frontend (React)                  │
│  - Dashboard, Monitoring, Alerts, Maps, Analytics   │
│  - Redux State, WebSocket Integration, TailwindCSS  │
│  - Modular Feature Structure (Stations, IoT, etc.)  │
└─────────────────┬───────────────────────────────────┘
                  │
                  │ REST API + WebSocket
                  │
┌─────────────────┴───────────────────────────────────┐
│                 Backend (NestJS)                     │
│  - Authentication, CRUD, Business Logic             │
│  - WebSocket Gateway, Event Broadcasting            │
│  - Workflow Execution Engine                        │
│  - MQTT Client Integration                          │
└─────────────────┬───────────────────────────────────┘
                  │
        ┌─────────┼─────────┐
        │         │         │
   ┌────▼────┐  ┌─▼──┐  ┌─▼─────┐
   │ Database │  │ MQTT│  │ Socket │
   │ PostgreSQL│ │ Broker   │ Server  │
   └──────────┘  └─────┘  └───────┘
```

### Key Components

**Frontend**
- React 18 with Redux Toolkit state management
- 11 feature modules (dashboard, stations, monitoring, alerts, etc.)
- Real-time updates via Socket.io
- TailwindCSS styling with Framer Motion animations
- React Leaflet for GIS visualization

**Backend**
- NestJS framework with TypeScript
- 10+ modular services (auth, stations, sensors, alerts, etc.)
- PostgreSQL database via TypeORM
- WebSocket gateway for real-time events
- MQTT client for IoT device communication
- JWT authentication with role-based guards

**Infrastructure**
- PostgreSQL for persistence
- Redis for caching
- Mosquitto MQTT broker for sensor data
- Docker containerization
- Environment-based configuration

---

## Project Structure

```
pfe-project/
├── AQUAFLOW_ARCHITECTURE.md          # Detailed design (this!)
├── IMPLEMENTATION_ROADMAP.md         # 4-phase implementation plan
├── QUICK_START.md                     # Code examples and templates
│
├── frontend/src/
│   ├── modules/                       # Feature-based modules
│   │   ├── auth/                      # Login, registration
│   │   ├── dashboard/                 # KPI, charts, alerts feed
│   │   ├── stations/                  # Station management CRUD
│   │   ├── monitoring/                # Real-time sensor data
│   │   ├── alerts/                    # Alert management
│   │   ├── maintenance/               # Maintenance workflows
│   │   ├── map/                       # GIS visualization
│   │   ├── analytics/                 # Trends, anomalies, KPIs
│   │   ├── reports/                   # Report generation
│   │   ├── iot/                       # Device management
│   │   ├── automation/                # Workflow builder (extended)
│   │   └── notifications/             # Alerts and channels
│   ├── store/                         # Redux state management
│   ├── services/                      # API clients
│   ├── hooks/                         # Custom React hooks
│   ├── components/                    # Reusable UI components
│   └── utils/                         # Utilities and helpers
│
└── backend/src/
    ├── database/                      # TypeORM entities & config
    │   └── entities/                  # User, Station, Sensor, etc.
    ├── auth/                          # JWT, strategies, guards
    ├── stations/                      # Station CRUD
    ├── sensors/                       # Sensor management
    ├── alerts/                        # Alert rules & processing
    ├── maintenance/                   # Maintenance workflows
    ├── iot/                           # MQTT client & handlers
    ├── realtime/                      # WebSocket gateway
    ├── analytics/                     # Trend & anomaly processing
    ├── reports/                       # PDF/Excel generation
    ├── notifications/                 # Email, SMS, push
    ├── execution/                     # Workflow runner (extended)
    └── common/                        # Guards, decorators, pipes
```

---

## Technology Stack

### Frontend
- **Framework**: React 18.2
- **State**: Redux Toolkit
- **Styling**: TailwindCSS
- **Charts**: Recharts
- **Animations**: Framer Motion
- **Maps**: React Leaflet
- **Real-time**: Socket.io-client
- **Routing**: React Router v6
- **HTTP**: Axios
- **Forms**: React Hook Form + Zod

### Backend
- **Framework**: NestJS 10
- **Language**: TypeScript
- **ORM**: TypeORM
- **Database**: PostgreSQL
- **Real-time**: Socket.io
- **IoT**: MQTT.js
- **Auth**: JWT, Passport.js
- **Validation**: Class-validator
- **Testing**: Jest

### Infrastructure
- **Containerization**: Docker & Docker Compose
- **Database**: PostgreSQL 15
- **Cache**: Redis 7
- **Message Broker**: Mosquitto MQTT
- **Version Control**: Git

---

## Key Features by Module

### 1. Authentication & RBAC
- JWT-based authentication
- Refresh token mechanism
- Four roles: Admin, Operator, Technician, Analyst
- Protected routes and endpoint guards
- Secure password hashing (bcrypt)

### 2. Dashboard
- Real-time KPI cards (pressure, flow, quality)
- Live charts with Recharts
- Active alerts feed
- Station status overview
- Energy consumption metrics
- Animated transitions with Framer Motion

### 3. Station Management
- Full CRUD operations
- Geographic coordinates with GIS integration
- Equipment tracking
- Capacity and status management
- Historical data and analytics

### 4. Real-Time Monitoring
- Live sensor data visualization
- Animated gauge indicators
- Threshold violation alerts
- Multi-sensor comparison
- WebSocket-powered updates (<500ms latency)

### 5. Alerts Module
- Threshold-based alert generation
- Severity levels (low, medium, high, critical)
- Alert acknowledgment workflow
- Comprehensive filtering and search
- Timeline view of alert history

### 6. Maintenance Management
- Intervention creation and tracking
- Technician assignment
- Status workflow (pending → in-progress → completed)
- Maintenance history
- Timeline visualization

### 7. GIS Map
- Interactive station mapping
- Color-coded status indicators
- Station popups with key metrics
- Filtering by region and status
- Real-time status updates

### 8. Analytics
- Trend analysis over time periods
- Anomaly detection algorithms
- KPI calculations
- Predictive metrics (optional)
- Period comparison views

### 9. Reports
- Customizable report builder
- PDF export functionality
- Excel export with formatting
- Report scheduling
- Predefined templates

### 10. IoT Integration
- MQTT broker connectivity
- Sensor data ingestion
- Device status monitoring
- Topic-based subscriptions
- Payload validation

### 11. Automation & Workflows
- Extended workflow builder (existing + new blocks)
- Industrial action blocks (sensor triggers, alerts, maintenance, MQTT, etc.)
- Workflow execution with logging
- Real-time event triggers
- Integration with all other modules

---

## Implementation Phases

### Phase 1: Core Infrastructure (Weeks 1-3)
- Database setup with TypeORM and PostgreSQL
- JWT authentication implementation
- WebSocket/Socket.io real-time infrastructure
- MQTT client integration
- Redux state management setup

**Deliverables**: Working auth, database, real-time pipeline, MQTT connection

### Phase 2: Feature Modules (Weeks 4-8)
- Frontend modular architecture
- Dashboard with live data
- Station CRUD operations
- Real-time monitoring interface
- Alerts and maintenance management
- GIS map visualization
- Backend APIs for all modules

**Deliverables**: Full-featured platform UI, complete REST APIs, WebSocket integration

### Phase 3: Workflow Extension (Weeks 9-11)
- Industrial workflow blocks (sensor-trigger, alert-sender, etc.)
- Execution handlers for each block type
- Workflow automation integration
- Real-time event triggering

**Deliverables**: Extended workflow builder, industrial automation capability

### Phase 4: Advanced Features (Weeks 12-16)
- Analytics and trend processing
- Report generation (PDF/Excel)
- IoT device management
- Notifications system
- Performance optimization
- UI/UX refinement

**Deliverables**: Complete platform, advanced features, production-ready code

---

## Getting Started

### Prerequisites
- Node.js 18+
- npm or yarn
- Docker & Docker Compose
- PostgreSQL 15
- Git

### Quick Start
```bash
# 1. Clone the repository
git clone <repo-url>
cd pfe-project

# 2. Install dependencies
cd frontend && npm install
cd ../backend && npm install

# 3. Set up environment variables
cp backend/.env.example backend/.env
cp frontend/.env.example frontend/.env

# 4. Start infrastructure
docker-compose up -d

# 5. Run migrations
cd backend && npm run typeorm migration:run

# 6. Start backend (from backend directory)
npm run start:dev

# 7. Start frontend (from frontend directory)
npm start

# 8. Access application
# Frontend: http://localhost:3000
# Backend API: http://localhost:3000/api
# WebSocket: ws://localhost:3001
```

For detailed code examples, see **QUICK_START.md**.

---

## API Routes Summary

### Authentication
- `POST /api/auth/register` - User registration
- `POST /api/auth/login` - Login with credentials
- `POST /api/auth/refresh` - Refresh access token
- `POST /api/auth/logout` - Logout

### Stations
- `GET /api/stations` - List all stations (paginated)
- `POST /api/stations` - Create new station
- `GET /api/stations/:id` - Get station details
- `PUT /api/stations/:id` - Update station
- `DELETE /api/stations/:id` - Delete station

### Sensors
- `GET /api/sensors` - List sensors
- `POST /api/sensors` - Create sensor
- `GET /api/sensors/:id` - Get sensor details
- `GET /api/sensors/:id/data` - Get sensor readings

### Alerts
- `GET /api/alerts` - List alerts
- `POST /api/alerts` - Create alert
- `PATCH /api/alerts/:id/acknowledge` - Acknowledge alert
- `DELETE /api/alerts/:id` - Clear alert

### Maintenance
- `GET /api/maintenance` - List interventions
- `POST /api/maintenance` - Create intervention
- `PATCH /api/maintenance/:id` - Update intervention
- `PATCH /api/maintenance/:id/assign` - Assign technician

### Workflows
- `GET /api/workflows` - List workflows
- `POST /api/workflows` - Create workflow
- `POST /api/workflows/:id/execute` - Execute workflow

### Analytics
- `GET /api/analytics/trends` - Get trend data
- `GET /api/analytics/anomalies` - Get anomalies
- `GET /api/analytics/kpis` - Get KPIs

Full API documentation available in architecture guide.

---

## Database Schema Highlights

### Core Entities
- **User**: Credentials, roles, profile
- **Station**: Facility info, location, capacity, status
- **Sensor**: Equipment with thresholds, types, location
- **SensorData**: Time-series readings with timestamps
- **Alert**: Threshold violations with severity and status
- **Maintenance**: Intervention records with assignments
- **Workflow**: Automation rules, graph JSON, execution logs
- **Notification**: Delivery logs for alerts and messages

### Relationships
- Users create and manage Stations
- Stations contain Sensors
- Sensors generate SensorData and Alerts
- Alerts trigger Notifications
- Workflows automate all of the above

---

## Security Considerations

- **Authentication**: JWT tokens with refresh mechanism
- **Authorization**: Role-based access control on all endpoints
- **Password Security**: Bcrypt hashing with salt rounds
- **Input Validation**: Class-validator on all DTOs
- **Error Handling**: Safe error messages without data leakage
- **CORS**: Configured for frontend domain only
- **HTTPS**: Recommended for production deployment

---

## Performance Targets

- **Dashboard Load**: <2 seconds
- **API Response**: <100ms (p95)
- **WebSocket Latency**: <500ms (real-time updates)
- **Database Query**: <50ms (with proper indexing)
- **Uptime SLA**: 95%+
- **Concurrent Users**: 100+ simultaneous connections

---

## Monitoring & Observability

Recommended additions (Phase 4+):
- Centralized logging (Winston, ELK Stack)
- Distributed tracing (Jaeger)
- Metrics collection (Prometheus)
- Application monitoring (New Relic, DataDog)
- Error tracking (Sentry)
- Health checks and liveness probes

---

## Deployment

### Development
- Local environment with Docker Compose
- Hot reload for code changes
- SQLite optional for quick testing

### Production
- Docker containers for all services
- PostgreSQL with replication
- Redis cluster for caching
- Load balancing (Nginx)
- SSL/TLS certificates
- Environment-based secrets
- Database migrations with rollback
- Automated backups

---

## Testing Strategy

### Unit Tests
- Service business logic
- Utility functions
- Component logic

### Integration Tests
- API endpoints
- Database operations
- WebSocket events
- MQTT integration

### E2E Tests
- Critical user workflows
- Authentication flows
- Data persistence

---

## Documentation

This project includes comprehensive documentation:
- **AQUAFLOW_ARCHITECTURE.md** - Detailed design and rationale
- **IMPLEMENTATION_ROADMAP.md** - Phase-by-phase implementation guide
- **QUICK_START.md** - Code templates and examples
- **API Documentation** - OpenAPI/Swagger (to be generated)
- **Database Schema** - SQL script and entity diagrams

---

## Contributing

### Code Standards
- Use existing code patterns and conventions
- Follow NestJS best practices
- Follow React hooks best practices
- Add unit tests for new functionality
- Document complex business logic
- Keep commits atomic and descriptive

### Commit Messages
Include co-author in commit messages:
```
feat: Add sensor threshold validation

Co-Authored-By: Oz <oz-agent@warp.dev>
```

---

## Support & Troubleshooting

### Common Issues

**Database Connection Error**
- Verify PostgreSQL is running: `docker-compose ps`
- Check DATABASE_URL in .env
- Ensure database exists

**WebSocket Not Connecting**
- Check REACT_APP_WS_URL in frontend .env
- Verify backend is running: `npm run start:dev`
- Check CORS configuration

**MQTT Not Receiving Data**
- Verify Mosquitto is running: `docker-compose ps`
- Check MQTT_BROKER_URL in backend .env
- Verify sensor is publishing to correct topics

### Performance Debugging
- Use Redux DevTools for state inspection
- Check browser Network tab for API delays
- Monitor database queries with TypeORM logging
- Use Node.js profiler for backend bottlenecks

---

## Roadmap & Future Enhancements

### Short Term (Next Quarter)
- [ ] Advanced analytics with ML models
- [ ] Real-time anomaly detection
- [ ] Multi-language support
- [ ] Mobile app (React Native)

### Medium Term (Next Year)
- [ ] Third-party integrations (SAP, Salesforce)
- [ ] Advanced reporting with BI tools
- [ ] Machine learning predictions
- [ ] Distributed system support

### Long Term (2+ Years)
- [ ] Edge computing integration
- [ ] Blockchain for audit trails
- [ ] GraphQL API
- [ ] Cloud-native deployment

---

## License

[Add your license here - MIT, Apache 2.0, etc.]

---

## Contact & Support

For questions or issues:
- Create GitHub issues for bug reports
- Submit pull requests with improvements
- Contact development team: [your-email]

---

## Acknowledgments

Built upon the existing workflow-builder project by extending it with enterprise industrial features while preserving the core architecture and user experience.

---

**Last Updated**: January 2024  
**Version**: 1.0.0 - Initial Architecture & Planning

