# AquaFlow: Complete Architecture Guide

## Table of Contents
1. [Project Overview](#project-overview)
2. [Folder Structure](#folder-structure)
3. [Frontend Architecture](#frontend-architecture)
4. [Backend Architecture](#backend-architecture)
5. [Database Schema](#database-schema)
6. [Workflow Extension Strategy](#workflow-extension-strategy)
7. [Real-Time Architecture](#real-time-architecture)
8. [API Routes](#api-routes)
9. [Authentication & RBAC](#authentication--rbac)
10. [Integration Patterns](#integration-patterns)

---

## Project Overview

**AquaFlow** is an industrial SCADA-like platform for drinking water station supervision, built by extending the existing workflow-builder project. It combines:

- Real-time IoT sensor monitoring
- Intelligent alerts and anomaly detection
- Maintenance and intervention management
- Visual workflow automation builder (existing, enhanced)
- Analytics and reporting
- GIS-based station visualization

**Key Principle**: Preserve the existing workflow builder architecture while extending it into an industrial automation platform.

---

## Folder Structure

### Updated Project Tree

```
pfe-project/
├── blocks.js/                              # Reference only
├── AQUAFLOW_ARCHITECTURE.md               # This document
├── AQUAFLOW_FOLDER_STRUCTURE.md           # Detailed structure
├── AQUAFLOW_DATABASE_SCHEMA.sql           # PostgreSQL schema
│
├── frontend/
│   ├── public/
│   ├── src/
│   │   ├── index.js
│   │   ├── main.jsx
│   │   ├── App.jsx
│   │   ├── routes.js                      # UPDATED: Add module routes
│   │   │
│   │   ├── assets/                        # Logos, fonts, images
│   │   ├── styles/                        # Global CSS (TailwindCSS)
│   │   │   ├── globals.css
│   │   │   ├── tailwind.config.js
│   │   │   └── animations.css
│   │   │
│   │   ├── components/                    # Reusable UI components
│   │   │   ├── Blocksidebar/             # Existing workflow builder
│   │   │   ├── canvas/                   # Existing workflow editor
│   │   │   ├── nodes/                    # Existing node types
│   │   │   ├── properties/               # Existing property editors
│   │   │   ├── builder/                  # Existing builder layout
│   │   │   ├── Navbars/                  # Existing navigation
│   │   │   ├── Footers/                  # Existing footers
│   │   │   ├── Headers/                  # Existing headers
│   │   │   ├── Sidebar/                  # Existing sidebar
│   │   │   │
│   │   │   ├── Common/                   # NEW: Shared components
│   │   │   │   ├── Button.jsx
│   │   │   │   ├── Card.jsx
│   │   │   │   ├── Loading.jsx
│   │   │   │   ├── Modal.jsx
│   │   │   │   ├── Alert.jsx
│   │   │   │   ├── Tabs.jsx
│   │   │   │   └── Badge.jsx
│   │   │   │
│   │   │   ├── DataDisplay/               # NEW: Data visualization
│   │   │   │   ├── KPICard.jsx
│   │   │   │   ├── Chart.jsx
│   │   │   │   ├── Table.jsx
│   │   │   │   ├── StatCard.jsx
│   │   │   │   └── StatusBadge.jsx
│   │   │   │
│   │   │   ├── Forms/                    # NEW: Form components
│   │   │   │   ├── FormInput.jsx
│   │   │   │   ├── FormSelect.jsx
│   │   │   │   ├── FormCheckbox.jsx
│   │   │   │   └── FormSubmit.jsx
│   │   │   │
│   │   │   └── Layout/                   # NEW: Layout wrappers
│   │   │       ├── MainLayout.jsx
│   │   │       ├── AuthLayout.jsx
│   │   │       └── PageHeader.jsx
│   │   │
│   │   ├── data/                         # Data and constants
│   │   │   ├── blocks.js                 # UPDATED: Add industrial blocks
│   │   │   ├── constants.js              # NEW: App constants
│   │   │   └── mockData.js               # NEW: For development
│   │   │
│   │   ├── engine/                       # Workflow execution engine (UNCHANGED)
│   │   │   ├── autosaveManager.js
│   │   │   ├── graphSerializer.js
│   │   │   ├── graphDeserializer.js
│   │   │   └── workflowExecutorClient.js
│   │   │
│   │   ├── registry/                     # Block registry (UNCHANGED)
│   │   │   ├── blockRegistry.js
│   │   │   └── blockFactory.js
│   │   │
│   │   ├── hooks/                        # Custom React hooks
│   │   │   ├── useWorkflowEditor.js      # Existing
│   │   │   ├── useJointGraph.js          # Existing
│   │   │   ├── useAutosave.js            # Existing
│   │   │   ├── useAuth.js                # NEW: Auth hook
│   │   │   ├── useSocket.js              # NEW: WebSocket hook
│   │   │   ├── useFetch.js               # NEW: Data fetching
│   │   │   ├── useLocalStorage.js        # NEW: Persistent state
│   │   │   └── useTheme.js               # NEW: Dark mode
│   │   │
│   │   ├── services/                     # API clients
│   │   │   ├── workflowApi.js            # Existing
│   │   │   ├── authService.js            # NEW
│   │   │   ├── stationService.js         # NEW
│   │   │   ├── sensorService.js          # NEW
│   │   │   ├── alertService.js           # NEW
│   │   │   ├── maintenanceService.js     # NEW
│   │   │   └── apiClient.js              # NEW: Axios wrapper
│   │   │
│   │   ├── store/                        # Redux state management (NEW)
│   │   │   ├── store.js                  # Redux configuration
│   │   │   ├── slices/
│   │   │   │   ├── authSlice.js
│   │   │   │   ├── stationsSlice.js
│   │   │   │   ├── sensorsSlice.js
│   │   │   │   ├── alertsSlice.js
│   │   │   │   ├── maintenanceSlice.js
│   │   │   │   ├── uiSlice.js
│   │   │   │   └── realtimeSlice.js
│   │   │   └── hooks.js                  # Custom selector hooks
│   │   │
│   │   ├── utils/                        # Utility functions
│   │   │   ├── graphHelpers.js           # Existing
│   │   │   ├── nodeHelpers.js            # Existing
│   │   │   ├── formatters.js             # NEW: Date, number formatting
│   │   │   ├── validators.js             # NEW: Form validation
│   │   │   ├── colors.js                 # NEW: Color utilities
│   │   │   └── constants.js              # NEW: App constants
│   │   │
│   │   ├── modules/                      # Feature-based modules (NEW)
│   │   │   │
│   │   │   ├── auth/                     # Authentication module
│   │   │   │   ├── pages/
│   │   │   │   │   ├── LoginPage.jsx
│   │   │   │   │   └── RegisterPage.jsx
│   │   │   │   ├── components/
│   │   │   │   │   ├── LoginForm.jsx
│   │   │   │   │   ├── RegisterForm.jsx
│   │   │   │   │   └── ProtectedRoute.jsx
│   │   │   │   └── index.js
│   │   │   │
│   │   │   ├── dashboard/                # Dashboard module
│   │   │   │   ├── pages/
│   │   │   │   │   └── DashboardPage.jsx
│   │   │   │   ├── components/
│   │   │   │   │   ├── KPISection.jsx
│   │   │   │   │   ├── AlertsFeed.jsx
│   │   │   │   │   ├── StationStatus.jsx
│   │   │   │   │   ├── ChartSection.jsx
│   │   │   │   │   ├── EnergyMetrics.jsx
│   │   │   │   │   └── RealtimeStats.jsx
│   │   │   │   ├── hooks/
│   │   │   │   │   └── useDashboardData.js
│   │   │   │   └── index.js
│   │   │   │
│   │   │   ├── stations/                 # Station management
│   │   │   │   ├── pages/
│   │   │   │   │   ├── StationsListPage.jsx
│   │   │   │   │   ├── StationDetailsPage.jsx
│   │   │   │   │   ├── CreateStationPage.jsx
│   │   │   │   │   └── StationAnalyticsPage.jsx
│   │   │   │   ├── components/
│   │   │   │   │   ├── StationCard.jsx
│   │   │   │   │   ├── StationForm.jsx
│   │   │   │   │   ├── StationEquipments.jsx
│   │   │   │   │   ├── StationMetrics.jsx
│   │   │   │   │   └── StationTimeline.jsx
│   │   │   │   ├── hooks/
│   │   │   │   │   └── useStations.js
│   │   │   │   └── index.js
│   │   │   │
│   │   │   ├── monitoring/               # Real-time monitoring
│   │   │   │   ├── pages/
│   │   │   │   │   ├── MonitoringPage.jsx
│   │   │   │   │   └── SensorDetailsPage.jsx
│   │   │   │   ├── components/
│   │   │   │   │   ├── SensorGrid.jsx
│   │   │   │   │   ├── LiveChart.jsx
│   │   │   │   │   ├── SensorCard.jsx
│   │   │   │   │   ├── ThresholdAlert.jsx
│   │   │   │   │   └── RealtimeGauge.jsx
│   │   │   │   ├── hooks/
│   │   │   │   │   └── useMonitoringData.js
│   │   │   │   └── index.js
│   │   │   │
│   │   │   ├── alerts/                   # Alerts management
│   │   │   │   ├── pages/
│   │   │   │   │   ├── AlertsPage.jsx
│   │   │   │   │   └── AlertDetailsPage.jsx
│   │   │   │   ├── components/
│   │   │   │   │   ├── AlertList.jsx
│   │   │   │   │   ├── AlertFilters.jsx
│   │   │   │   │   ├── AlertCard.jsx
│   │   │   │   │   ├── AlertTimeline.jsx
│   │   │   │   │   └── SeverityBadge.jsx
│   │   │   │   ├── hooks/
│   │   │   │   │   └── useAlerts.js
│   │   │   │   └── index.js
│   │   │   │
│   │   │   ├── maintenance/              # Maintenance module
│   │   │   │   ├── pages/
│   │   │   │   │   ├── MaintenancePage.jsx
│   │   │   │   │   ├── InterventionDetailsPage.jsx
│   │   │   │   │   └── TechnicianAssignmentPage.jsx
│   │   │   │   ├── components/
│   │   │   │   │   ├── MaintenanceList.jsx
│   │   │   │   │   ├── InterventionForm.jsx
│   │   │   │   │   ├── InterventionCard.jsx
│   │   │   │   │   ├── TechnicianSelect.jsx
│   │   │   │   │   ├── MaintenanceHistory.jsx
│   │   │   │   │   └── StatusTimeline.jsx
│   │   │   │   ├── hooks/
│   │   │   │   │   └── useMaintenance.js
│   │   │   │   └── index.js
│   │   │   │
│   │   │   ├── map/                      # GIS visualization
│   │   │   │   ├── pages/
│   │   │   │   │   └── MapPage.jsx
│   │   │   │   ├── components/
│   │   │   │   │   ├── StationMap.jsx
│   │   │   │   │   ├── MapMarker.jsx
│   │   │   │   │   ├── StationPopup.jsx
│   │   │   │   │   ├── MapLegend.jsx
│   │   │   │   │   └── MapFilters.jsx
│   │   │   │   ├── hooks/
│   │   │   │   │   └── useMapData.js
│   │   │   │   └── index.js
│   │   │   │
│   │   │   ├── analytics/                # Analytics module
│   │   │   │   ├── pages/
│   │   │   │   │   ├── AnalyticsPage.jsx
│   │   │   │   │   ├── TrendAnalysisPage.jsx
│   │   │   │   │   └── AnomalyDetectionPage.jsx
│   │   │   │   ├── components/
│   │   │   │   │   ├── TrendChart.jsx
│   │   │   │   │   ├── AnomalyAlert.jsx
│   │   │   │   │   ├── PeriodComparison.jsx
│   │   │   │   │   ├── PredictiveMetrics.jsx
│   │   │   │   │   └── OperationalKPI.jsx
│   │   │   │   ├── hooks/
│   │   │   │   │   └── useAnalytics.js
│   │   │   │   └── index.js
│   │   │   │
│   │   │   ├── reports/                  # Reports module
│   │   │   │   ├── pages/
│   │   │   │   │   ├── ReportsPage.jsx
│   │   │   │   │   └── ReportBuilderPage.jsx
│   │   │   │   ├── components/
│   │   │   │   │   ├── ReportList.jsx
│   │   │   │   │   ├── ReportBuilder.jsx
│   │   │   │   │   ├── ReportExport.jsx
│   │   │   │   │   └── ReportPreview.jsx
│   │   │   │   ├── hooks/
│   │   │   │   │   └── useReports.js
│   │   │   │   └── index.js
│   │   │   │
│   │   │   ├── iot/                      # IoT management
│   │   │   │   ├── pages/
│   │   │   │   │   ├── IoTDevicesPage.jsx
│   │   │   │   │   └── SensorConfigPage.jsx
│   │   │   │   ├── components/
│   │   │   │   │   ├── DeviceList.jsx
│   │   │   │   │   ├── SensorForm.jsx
│   │   │   │   │   ├── DeviceCard.jsx
│   │   │   │   │   └── ConnectionStatus.jsx
│   │   │   │   ├── hooks/
│   │   │   │   │   └── useIoT.js
│   │   │   │   └── index.js
│   │   │   │
│   │   │   ├── automation/               # Workflow automation (NEW)
│   │   │   │   ├── pages/
│   │   │   │   │   ├── AutomationPage.jsx
│   │   │   │   │   ├── WorkflowListPage.jsx
│   │   │   │   │   └── WorkflowDetailsPage.jsx
│   │   │   │   ├── components/
│   │   │   │   │   ├── WorkflowList.jsx
│   │   │   │   │   ├── WorkflowBuilder.jsx (reuses existing builder)
│   │   │   │   │   ├── WorkflowExecution.jsx
│   │   │   │   │   └── WorkflowLogs.jsx
│   │   │   │   ├── hooks/
│   │   │   │   │   └── useAutomation.js
│   │   │   │   └── index.js
│   │   │   │
│   │   │   ├── notifications/            # Notifications
│   │   │   │   ├── pages/
│   │   │   │   │   └── NotificationsPage.jsx
│   │   │   │   ├── components/
│   │   │   │   │   ├── NotificationCenter.jsx
│   │   │   │   │   ├── NotificationItem.jsx
│   │   │   │   │   └── PreferencesForm.jsx
│   │   │   │   ├── hooks/
│   │   │   │   │   └── useNotifications.js
│   │   │   │   └── index.js
│   │   │   │
│   │   │   └── shared/                   # Shared utilities for modules
│   │   │       ├── constants/
│   │   │       ├── hooks/
│   │   │       ├── utils/
│   │   │       └── types/
│   │   │
│   │   ├── pages/                        # Legacy: page components
│   │   │   └── BuilderPage.jsx           # Existing workflow builder
│   │   │
│   │   ├── layouts/                      # Layout templates
│   │   │   ├── Admin.js                  # Existing
│   │   │   └── Auth.js                   # Existing
│   │   │
│   │   ├── views/                        # Legacy: view components
│   │   │   ├── Index.js                  # Existing
│   │   │   ├── builder/                  # Existing
│   │   │   └── examples/                 # Existing
│   │   │
│   │   ├── variables/                    # Constants (UNCHANGED)
│   │   │   └── charts.js
│   │   │
│   │   └── routes.js                     # UPDATED: Router configuration
│   │
│   ├── package.json                      # UPDATED: Add new dependencies
│   └── .env.example                      # NEW: Environment template
│
├── backend/
│   ├── src/
│   │   ├── main.ts                       # NestJS entry point
│   │   ├── app.module.ts                 # UPDATED: Import all modules
│   │   │
│   │   ├── common/                       # Shared utilities
│   │   │   ├── types/
│   │   │   │   ├── workflow.types.ts    # Existing
│   │   │   │   ├── api.types.ts         # NEW
│   │   │   │   └── database.types.ts    # NEW
│   │   │   ├── decorators/
│   │   │   │   ├── Roles.decorator.ts   # NEW: Role-based access
│   │   │   │   └── ApiResponse.decorator.ts # NEW
│   │   │   ├── guards/
│   │   │   │   ├── JwtGuard.ts          # NEW
│   │   │   │   ├── RolesGuard.ts        # NEW
│   │   │   │   └── OptionalJwtGuard.ts  # NEW
│   │   │   ├── interceptors/
│   │   │   │   ├── ResponseInterceptor.ts # NEW
│   │   │   │   └── ErrorInterceptor.ts   # NEW
│   │   │   ├── pipes/
│   │   │   │   ├── ValidationPipe.ts    # NEW
│   │   │   │   └── ParseIdPipe.ts       # NEW
│   │   │   ├── filters/
│   │   │   │   └── HttpExceptionFilter.ts # NEW
│   │   │   └── exceptions/
│   │   │       ├── CustomException.ts    # NEW
│   │   │       └── NotFoundError.ts      # NEW
│   │   │
│   │   ├── database/                     # Database layer
│   │   │   ├── entities/
│   │   │   │   ├── User.entity.ts       # NEW
│   │   │   │   ├── Station.entity.ts    # NEW
│   │   │   │   ├── Sensor.entity.ts     # NEW
│   │   │   │   ├── SensorData.entity.ts # NEW
│   │   │   │   ├── Alert.entity.ts      # NEW
│   │   │   │   ├── Maintenance.entity.ts # NEW
│   │   │   │   ├── Workflow.entity.ts   # NEW
│   │   │   │   ├── Notification.entity.ts # NEW
│   │   │   │   └── flow.schema.ts       # Existing
│   │   │   ├── migrations/
│   │   │   │   └── [timestamps]/        # NEW: DB migrations
│   │   │   ├── seeds/
│   │   │   │   └── seed.ts              # NEW: Initial data
│   │   │   ├── database.module.ts       # NEW: TypeORM config
│   │   │   └── database.service.ts      # NEW: DB utilities
│   │   │
│   │   ├── auth/                         # Authentication module (NEW)
│   │   │   ├── auth.service.ts
│   │   │   ├── auth.controller.ts
│   │   │   ├── auth.module.ts
│   │   │   ├── dto/
│   │   │   │   ├── login.dto.ts
│   │   │   │   ├── register.dto.ts
│   │   │   │   └── refresh-token.dto.ts
│   │   │   ├── strategies/
│   │   │   │   ├── jwt.strategy.ts
│   │   │   │   ├── local.strategy.ts
│   │   │   │   └── refresh-token.strategy.ts
│   │   │   └── utils/
│   │   │       └── password.util.ts
│   │   │
│   │   ├── stations/                     # Station management (NEW)
│   │   │   ├── stations.service.ts
│   │   │   ├── stations.controller.ts
│   │   │   ├── stations.module.ts
│   │   │   ├── dto/
│   │   │   │   ├── create-station.dto.ts
│   │   │   │   ├── update-station.dto.ts
│   │   │   │   └── station-filter.dto.ts
│   │   │   └── helpers/
│   │   │       └── station.helpers.ts
│   │   │
│   │   ├── sensors/                      # Sensor management (NEW)
│   │   │   ├── sensors.service.ts
│   │   │   ├── sensors.controller.ts
│   │   │   ├── sensors.module.ts
│   │   │   ├── dto/
│   │   │   │   ├── create-sensor.dto.ts
│   │   │   │   └── update-sensor.dto.ts
│   │   │   └── sensor-data.service.ts
│   │   │
│   │   ├── iot/                          # IoT integration (NEW)
│   │   │   ├── iot.service.ts
│   │   │   ├── iot.module.ts
│   │   │   ├── mqtt/
│   │   │   │   ├── mqtt.client.ts
│   │   │   │   ├── mqtt.gateway.ts
│   │   │   │   └── mqtt.config.ts
│   │   │   ├── handlers/
│   │   │   │   ├── sensor-message.handler.ts
│   │   │   │   └── device-event.handler.ts
│   │   │   └── dto/
│   │   │       └── mqtt-payload.dto.ts
│   │   │
│   │   ├── realtime/                     # WebSocket real-time (NEW)
│   │   │   ├── realtime.gateway.ts
│   │   │   ├── realtime.service.ts
│   │   │   ├── realtime.module.ts
│   │   │   └── events/
│   │   │       ├── sensor-update.event.ts
│   │   │       ├── alert.event.ts
│   │   │       ├── station-status.event.ts
│   │   │       └── workflow-event.ts
│   │   │
│   │   ├── alerts/                       # Alerts management (NEW)
│   │   │   ├── alerts.service.ts
│   │   │   ├── alerts.controller.ts
│   │   │   ├── alerts.module.ts
│   │   │   ├── dto/
│   │   │   │   ├── create-alert.dto.ts
│   │   │   │   ├── acknowledge-alert.dto.ts
│   │   │   │   └── alert-filter.dto.ts
│   │   │   ├── rules/
│   │   │   │   ├── alert-rule.engine.ts
│   │   │   │   └── rule.evaluator.ts
│   │   │   └── processors/
│   │   │       └── alert.processor.ts
│   │   │
│   │   ├── maintenance/                  # Maintenance module (NEW)
│   │   │   ├── maintenance.service.ts
│   │   │   ├── maintenance.controller.ts
│   │   │   ├── maintenance.module.ts
│   │   │   ├── dto/
│   │   │   │   ├── create-intervention.dto.ts
│   │   │   │   ├── update-intervention.dto.ts
│   │   │   │   └── assign-technician.dto.ts
│   │   │   └── helpers/
│   │   │       └── intervention.helpers.ts
│   │   │
│   │   ├── reports/                      # Reports generation (NEW)
│   │   │   ├── reports.service.ts
│   │   │   ├── reports.controller.ts
│   │   │   ├── reports.module.ts
│   │   │   ├── generators/
│   │   │   │   ├── pdf.generator.ts
│   │   │   │   ├── excel.generator.ts
│   │   │   │   └── report.builder.ts
│   │   │   └── dto/
│   │   │       └── report-config.dto.ts
│   │   │
│   │   ├── analytics/                    # Analytics module (NEW)
│   │   │   ├── analytics.service.ts
│   │   │   ├── analytics.controller.ts
│   │   │   ├── analytics.module.ts
│   │   │   ├── processors/
│   │   │   │   ├── trend.processor.ts
│   │   │   │   ├── anomaly.detector.ts
│   │   │   │   └── kpi.calculator.ts
│   │   │   └── dto/
│   │   │       └── analytics-query.dto.ts
│   │   │
│   │   ├── execution/                    # Workflow execution (ENHANCED)
│   │   │   ├── engine/
│   │   │   │   ├── execution-context.ts      # Existing
│   │   │   │   ├── node-executor.ts          # Existing
│   │   │   │   ├── workflow-runner.ts        # Existing
│   │   │   │   ├── workflow-context.ts       # NEW: Extended context
│   │   │   │   └── block-executor.ts         # NEW: Extensible executor
│   │   │   ├── handlers/
│   │   │   │   ├── input.handler.ts          # Existing
│   │   │   │   ├── output.handler.ts         # Existing
│   │   │   │   ├── action.handler.ts         # Existing
│   │   │   │   ├── decision.handler.ts       # Existing
│   │   │   │   ├── sensor-trigger.handler.ts # NEW
│   │   │   │   ├── threshold-check.handler.ts # NEW
│   │   │   │   ├── alert-sender.handler.ts  # NEW
│   │   │   │   ├── maintenance-request.handler.ts # NEW
│   │   │   │   ├── mqtt-publish.handler.ts  # NEW
│   │   │   │   ├── email-notification.handler.ts # NEW
│   │   │   │   ├── sms-notification.handler.ts # NEW
│   │   │   │   ├── pump-control.handler.ts  # NEW
│   │   │   │   ├── scheduler.handler.ts      # NEW
│   │   │   │   ├── station-monitor.handler.ts # NEW
│   │   │   │   └── analytics-processor.handler.ts # NEW
│   │   │   ├── execution.module.ts
│   │   │   └── dto/
│   │   │       └── handler-execution.dto.ts
│   │   │
│   │   ├── flows/                        # Workflow flows (EXISTING)
│   │   │   ├── flows.module.ts
│   │   │   ├── flows.service.ts
│   │   │   ├── flows.controller.ts
│   │   │   ├── flow-executor.service.ts
│   │   │   ├── flow-validator.service.ts
│   │   │   └── dto/
│   │   │       ├── create-flow.dto.ts
│   │   │       ├── execute-flow.dto.ts
│   │   │       └── flow-metadata.dto.ts
│   │   │
│   │   └── notifications/                # Notification delivery (NEW)
│   │       ├── notifications.service.ts
│   │       ├── notifications.module.ts
│   │       ├── channels/
│   │       │   ├── email.channel.ts
│   │       │   ├── sms.channel.ts
│   │       │   └── push.channel.ts
│   │       └── dto/
│   │           └── send-notification.dto.ts
│   │
│   ├── config/                           # Configuration (NEW)
│   │   ├── database.config.ts
│   │   ├── jwt.config.ts
│   │   ├── mqtt.config.ts
│   │   ├── mail.config.ts
│   │   └── app.config.ts
│   │
│   ├── app.module.ts                     # UPDATED: All modules imported
│   ├── package.json                      # UPDATED: New dependencies
│   ├── tsconfig.json
│   ├── .env.example                      # NEW: Environment template
│   └── nest-cli.json
│
├── docker-compose.yml                    # NEW: PostgreSQL, Redis, MQTT
├── .gitignore
├── README.md                             # UPDATED: Project documentation
└── DEVELOPMENT.md                        # NEW: Development guide
```

---

## Frontend Architecture

### State Management (Redux Toolkit)

```
store/
├── store.js                 # Redux configuration with middleware
├── slices/
│   ├── authSlice.js        # { user, token, isAuthenticated, roles }
│   ├── stationsSlice.js    # { stations, selectedStation, loading }
│   ├── sensorsSlice.js     # { sensors, sensorData, realtime }
│   ├── alertsSlice.js      # { alerts, acknowledged, filters }
│   ├── maintenanceSlice.js # { interventions, technicians, history }
│   ├── uiSlice.js          # { theme, sidebarOpen, notifications }
│   └── realtimeSlice.js    # { socket, subscriptions, updates }
└── hooks.js                # useAppDispatch, useAppSelector, etc.
```

### Component Hierarchy

```
App
├── Router
│   ├── AuthLayout
│   │   ├── LoginPage
│   │   └── RegisterPage
│   └── MainLayout
│       ├── Sidebar (Navigation)
│       ├── TopBar (Header)
│       └── Main Content
│           ├── DashboardPage
│           ├── StationsPage
│           ├── MonitoringPage
│           ├── AlertsPage
│           ├── MaintenancePage
│           ├── MapPage
│           ├── AnalyticsPage
│           ├── ReportsPage
│           ├── AutomationPage (Workflow Builder)
│           ├── IoTPage
│           └── NotificationsPage
```

### Custom Hooks Pattern

```javascript
// Authentication
useAuth()           // { user, login, logout, hasRole }
useProtectedRoute() // Guard navigation

// Data Fetching
useFetch(url, options)  // { data, loading, error, refetch }
useStations()           // Redux selector shortcuts
useAlerts()
useSensors()

// Real-time
useSocket()             // { socket, emit, on, off }
useRealtimeUpdates()    // Subscribe to sensor updates

// UI
useTheme()              // { theme, toggleTheme }
useLocalStorage()       // { value, setValue }
```

---

## Backend Architecture

### Module Structure

Each module follows NestJS conventions:

```
module/
├── module.controller.ts     # HTTP endpoints
├── module.service.ts        # Business logic
├── module.module.ts         # Module definition
├── dto/
│   ├── create-module.dto.ts
│   ├── update-module.dto.ts
│   └── filter.dto.ts
└── helpers/
    └── module.helpers.ts
```

### Layered Architecture

```
HTTP Request
    ↓
Guard (JWT, Roles)
    ↓
Controller (Route handling)
    ↓
Service (Business logic)
    ↓
Repository/TypeORM (Data access)
    ↓
Database
```

### Dependency Injection

All modules use NestJS dependency injection with proper module imports.

---

## Database Schema

### Entity Relationships

```
User (1) ──────────→ (N) Station
User (1) ──────────→ (N) Maintenance
Station (1) ─────────→ (N) Sensor
Station (1) ─────────→ (N) Alert
Sensor (1) ──────────→ (N) SensorData
Sensor (1) ──────────→ (N) Alert
Alert (1) ──────────→ (N) Notification
Workflow (1) ────────→ (N) WorkflowExecution
```

### Key Entities

**User**
- id, email, password_hash, firstname, lastname
- role (enum: admin, operator, technician, analyst)
- created_at, updated_at

**Station**
- id, name, location, coordinates (lat/long), capacity, type
- status (enum: normal, warning, critical, offline)
- description, created_at, updated_at

**Sensor**
- id, station_id, name, type (pressure, flow, temperature, quality)
- unit, min_threshold, max_threshold, location
- last_reading, last_reading_at, created_at

**SensorData**
- id, sensor_id, value, timestamp, quality_flags

**Alert**
- id, sensor_id, station_id, type, severity, message
- status (active, acknowledged, resolved)
- created_at, acknowledged_at

**Maintenance**
- id, station_id, type, status, description
- assigned_to (user_id), created_by, started_at, completed_at

**Workflow**
- id, name, description, graph (JSON), is_active
- created_by, created_at, updated_at

---

## Workflow Extension Strategy

### Industrial Blocks

New blocks added to `blocks.js`:

1. **sensor-trigger**: Listen to sensor value changes
2. **threshold-checker**: Validate sensor against min/max
3. **alert-sender**: Create alerts with message
4. **maintenance-request**: Trigger maintenance workflow
5. **mqtt-publisher**: Publish data to MQTT topic
6. **email-notification**: Send email alerts
7. **sms-notification**: Send SMS messages
8. **pump-control**: Send control commands
9. **analytics-processor**: Calculate metrics
10. **timer/scheduler**: Delay or schedule actions
11. **station-monitor**: Monitor station status
12. **decision**: Route based on conditions (existing, enhanced)

### Execution Handler Pattern

```typescript
// backend/src/execution/handlers/sensor-trigger.handler.ts

@Injectable()
export class SensorTriggerHandler implements INodeHandler {
  handle(node: WorkflowNode, input: any, context: ExecutionContext) {
    const { sensorId } = node.properties;
    // Listen to sensor data and trigger workflow
    return { triggered: true, sensor_id: sensorId };
  }
}
```

---

## Real-Time Architecture

### WebSocket Flow

```
Frontend (Socket.io Client)
    ↓
NestJS Gateway (Socket.io Adapter)
    ↓
RealtimeService (Event broadcasting)
    ↓
IoT Module (Sensor data) / Alerts Service / Maintenance Service
    ↓
Event Emission
    ↓
Frontend (Real-time update)
```

### Events

**sensor-update**
```json
{ "sensorId": "123", "value": 50.5, "timestamp": "2024-01-01T00:00:00Z" }
```

**alert-created**
```json
{ "alertId": "456", "severity": "high", "message": "Pressure critical" }
```

**maintenance-status-changed**
```json
{ "maintenanceId": "789", "status": "in-progress", "assignedTo": "user123" }
```

---

## API Routes

### Authentication
- `POST /api/auth/register` - Register user
- `POST /api/auth/login` - Login
- `POST /api/auth/refresh` - Refresh token
- `POST /api/auth/logout` - Logout

### Stations
- `GET /api/stations` - List stations
- `POST /api/stations` - Create station
- `GET /api/stations/:id` - Get station details
- `PUT /api/stations/:id` - Update station
- `DELETE /api/stations/:id` - Delete station
- `GET /api/stations/:id/analytics` - Station analytics

### Sensors
- `GET /api/sensors` - List sensors
- `POST /api/sensors` - Create sensor
- `GET /api/sensors/:id` - Get sensor details
- `PUT /api/sensors/:id` - Update sensor
- `GET /api/sensors/:id/data` - Get sensor readings

### Alerts
- `GET /api/alerts` - List alerts
- `POST /api/alerts` - Create alert (trigger)
- `GET /api/alerts/:id` - Get alert details
- `PATCH /api/alerts/:id/acknowledge` - Acknowledge alert
- `DELETE /api/alerts/:id` - Clear alert

### Maintenance
- `GET /api/maintenance` - List interventions
- `POST /api/maintenance` - Create intervention
- `GET /api/maintenance/:id` - Get intervention details
- `PATCH /api/maintenance/:id` - Update status
- `PATCH /api/maintenance/:id/assign` - Assign technician

### Workflows
- `GET /api/workflows` - List workflows
- `POST /api/workflows` - Create workflow
- `GET /api/workflows/:id` - Get workflow
- `PUT /api/workflows/:id` - Update workflow
- `POST /api/workflows/:id/execute` - Execute workflow
- `GET /api/workflows/:id/executions` - Get execution history

### Analytics
- `GET /api/analytics/trends` - Trend analysis
- `GET /api/analytics/anomalies` - Anomaly detection
- `GET /api/analytics/kpis` - Operational KPIs

### Reports
- `GET /api/reports` - List reports
- `POST /api/reports/generate` - Generate new report
- `GET /api/reports/:id/pdf` - Download PDF
- `GET /api/reports/:id/excel` - Download Excel

---

## Authentication & RBAC

### JWT Flow

```
Login Endpoint
    ↓
Verify Credentials
    ↓
Generate JWT + Refresh Token
    ↓
Client Stores Tokens
    ↓
Request with JWT in Authorization Header
    ↓
JwtGuard Validates
    ↓
RolesGuard Checks Permissions
    ↓
Access Granted
```

### Decorator Usage

```typescript
@Controller('protected')
export class ProtectedController {
  
  @Get()
  @UseGuards(JwtGuard, RolesGuard)
  @Roles('admin', 'operator')
  getSomething() {
    // Only admin and operator can access
  }
}
```

### Roles

- **Admin**: Full access, user management, system configuration
- **Operator**: Dashboard, monitoring, acknowledge alerts
- **Technician**: Maintenance, intervention records, equipment status
- **Analyst**: Analytics, reports, read-only access

---

## Integration Patterns

### How Modules Work Together

**Workflow + Alerts + Maintenance**
```
1. Workflow triggers on sensor-threshold event
2. Decision node evaluates: Is pressure > max?
3. If true → alert-sender block creates alert
4. Same workflow → maintenance-request creates intervention
5. Alerts service broadcasts to frontend via WebSocket
6. Frontend updates alerts feed in real-time
```

**IoT + Realtime + Analytics**
```
1. MQTT broker sends sensor data to backend
2. IoT module receives and validates
3. Sensor service saves to database
4. Realtime gateway broadcasts via WebSocket
5. Analytics module aggregates for trends
6. Dashboard receives and displays live charts
```

**Automation + Notifications**
```
1. Workflow execution completes
2. Notification handler formats message
3. Notification service enqueues delivery
4. Email/SMS channel sends via external service
5. Delivery log recorded in database
6. User receives notification
```

---

## Configuration Files

### Environment Variables (.env)

```env
# Database
DATABASE_URL=postgresql://user:password@localhost:5432/aquaflow
DATABASE_HOST=localhost
DATABASE_PORT=5432
DATABASE_NAME=aquaflow
DATABASE_USER=postgres
DATABASE_PASSWORD=postgres

# JWT
JWT_SECRET=your-secret-key-here
JWT_EXPIRATION=3600

# MQTT
MQTT_BROKER_URL=mqtt://localhost:1883
MQTT_USERNAME=user
MQTT_PASSWORD=pass

# Socket.io
SOCKET_PORT=3001

# Mail (optional)
MAIL_HOST=smtp.gmail.com
MAIL_PORT=587
MAIL_USER=your-email@gmail.com
MAIL_PASS=app-password

# Frontend API
REACT_APP_API_URL=http://localhost:3000/api
REACT_APP_WS_URL=ws://localhost:3001
```

### Docker Compose (postgres, redis, mqtt)

Includes services for local development with PostgreSQL, Redis (caching), and Mosquitto MQTT.

---

## Development Workflow

### Adding a New Feature

1. **Define Entity** in `backend/src/database/entities/`
2. **Create Migration** with TypeORM
3. **Create Module** with Service + Controller
4. **Implement DTOs** for validation
5. **Create Redux Slice** in frontend
6. **Build UI Components** in appropriate module
7. **Implement API Client** in services
8. **Add Custom Hooks** for data fetching
9. **Update Routes** if needed
10. **Test** with API and Frontend

### Extending Workflows

1. Add block definition to `frontend/src/data/blocks.js`
2. Create handler in `backend/src/execution/handlers/`
3. Register handler in execution module
4. Test with workflow builder
5. Document in workflow examples

---

## Next Steps

This architecture document provides the complete structure. The implementation will follow in phases with actual code files, database schemas, and example implementations for each major module.

