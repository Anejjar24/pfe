# AquaFlow Feature Implementation Matrix

# AquaFlow — Feature Implementation Matrix

**Audit date:** 2026-05-10
**Legend:** ✅ COMPLETE | ⚠️ PARTIAL | ❌ MISSING | 🔴 BROKEN

---

## 1. Authentication & Authorization


| Feature                     | Backend                              | Frontend                     | API                       | UI                  | Realtime | Status       |
| --------------------------- | ------------------------------------ | ---------------------------- | ------------------------- | ------------------- | -------- | ------------ |
| User registration           | ✅                                   | ✅                           | ✅ POST /auth/register    | ✅ Register form    | ❌ N/A   | ✅ COMPLETE  |
| User login (JWT)            | ✅                                   | ✅                           | ✅ POST /auth/login       | ✅ Login form       | ❌ N/A   | ✅ COMPLETE  |
| JWT access token            | ✅ 1h expiry                         | ✅ stored in memory          | ✅                        | ✅                  | ❌ N/A   | ✅ COMPLETE  |
| JWT refresh token           | ✅ 7d expiry                         | ✅ persisted via authSession | ✅ POST /auth/refresh     | ✅ auto-interceptor | ❌ N/A   | ✅ COMPLETE  |
| Logout                      | ✅                                   | ✅ useLogout hook            | ✅ POST /auth/logout      | ✅                  | ❌ N/A   | ✅ COMPLETE  |
| Current user (me)           | ✅                                   | ✅                           | ✅ GET /auth/me           | ✅                  | ❌ N/A   | ✅ COMPLETE  |
| JwtGuard (route protection) | ✅                                   | ✅ ProtectedRoute            | ✅ Applied on most routes | ✅                  | ❌ N/A   | ✅ COMPLETE  |
| RolesGuard (RBAC)           | ✅ admin/operator/technician/analyst | ✅ role checks in UI         | ✅                        | ✅                  | ❌ N/A   | ✅ COMPLETE  |
| Roles decorator             | ✅                                   | ❌ N/A                       | ✅                        | ❌ N/A              | ❌ N/A   | ✅ COMPLETE  |
| CurrentUser decorator       | ✅                                   | ❌ N/A                       | ✅                        | ❌ N/A              | ❌ N/A   | ✅ COMPLETE  |
| User profile page           | ❌ no API                            | ⚠️ Argon stub              | ❌                        | ⚠️ template only  | ❌ N/A   | ⚠️ PARTIAL |
| User management (admin)     | ❌ no UsersModule                    | ❌ no page                   | ❌                        | ❌                  | ❌ N/A   | ❌ MISSING   |
| Password reset              | ❌                                   | ❌                           | ❌                        | ❌                  | ❌ N/A   | ❌ MISSING   |
| Email verification          | ❌                                   | ❌                           | ❌                        | ❌                  | ❌ N/A   | ❌ MISSING   |

---

## 2. Station Management


| Feature                    | Backend              | Frontend                            | API                     | UI                  | Realtime                        | Status       |
| -------------------------- | -------------------- | ----------------------------------- | ----------------------- | ------------------- | ------------------------------- | ------------ |
| List stations (paginated)  | ✅                   | ✅                                  | ✅ GET /stations        | ✅ table view       | ❌                              | ✅ COMPLETE  |
| Create station             | ✅                   | ✅                                  | ✅ POST /stations       | ✅ modal form       | ❌                              | ✅ COMPLETE  |
| Update station             | ✅                   | ✅                                  | ✅ PATCH /stations/:id  | ✅ edit modal       | ❌                              | ✅ COMPLETE  |
| Delete station             | ✅                   | ❌                                  | ✅ DELETE /stations/:id | ❌ no delete UI     | ❌                              | ⚠️ PARTIAL |
| Filter by status/type      | ✅                   | ✅                                  | ✅ query params         | ✅ filter dropdowns | ❌                              | ✅ COMPLETE  |
| Station details page       | ✅ GET /stations/:id | ❌                                  | ✅                      | ❌ no route         | ❌                              | ⚠️ PARTIAL |
| Station analytics          | ❌                   | ❌                                  | ❌                      | ❌                  | ❌                              | ❌ MISSING   |
| GIS map visualization      | ❌                   | ❌                                  | ❌                      | ❌                  | ❌                              | ❌ MISSING   |
| Station timeline/history   | ❌                   | ❌                                  | ❌                      | ❌                  | ❌                              | ❌ MISSING   |
| Equipment list per station | ❌                   | ❌                                  | ❌                      | ❌                  | ❌                              | ❌ MISSING   |
| Station status realtime    | ✅ entity has status | ❌ no`station-status` event emitted | ❌                      | ❌                  | 🔴 listener exists, never fires | 🔴 BROKEN    |

---

## 3. Sensor Monitoring


| Feature                         | Backend                                   | Frontend             | API                    | UI              | Realtime                      | Status       |
| ------------------------------- | ----------------------------------------- | -------------------- | ---------------------- | --------------- | ----------------------------- | ------------ |
| List sensors                    | ✅                                        | ✅                   | ✅ GET /sensors        | ✅ table        | ❌                            | ✅ COMPLETE  |
| Create sensor                   | ✅                                        | ✅                   | ✅ POST /sensors       | ✅ modal form   | ❌                            | ✅ COMPLETE  |
| Update sensor                   | ✅                                        | ❌                   | ✅ PATCH /sensors/:id  | ❌ no edit UI   | ❌                            | ⚠️ PARTIAL |
| Delete sensor                   | ✅                                        | ❌                   | ✅ DELETE /sensors/:id | ❌ no delete UI | ❌                            | ⚠️ PARTIAL |
| Sensor historical data          | ✅ GET /sensors/:id/data                  | ❌                   | ✅                     | ❌ no chart UI  | ❌                            | ⚠️ PARTIAL |
| Sensor details page             | ✅                                        | ❌                   | ✅                     | ❌ no route     | ❌                            | ⚠️ PARTIAL |
| Live sensor reading display     | ✅ lastReading field                      | ✅ table shows value | ✅                     | ✅ static       | ⚠️ updates via WS           | ⚠️ PARTIAL |
| Live chart / time series        | ❌                                        | ❌                   | ❌                     | ❌              | ❌                            | ❌ MISSING   |
| Realtime gauge widget           | ❌                                        | ❌                   | ❌                     | ❌              | ❌                            | ❌ MISSING   |
| Threshold configuration UI      | ✅ in create modal                        | ✅                   | ✅                     | ✅              | ❌                            | ✅ COMPLETE  |
| Threshold violation detection   | ✅ IotService checks`isThresholdViolated` | ❌                   | ✅                     | ❌              | ✅ via alert-created WS event | ✅ COMPLETE  |
| MQTT data ingestion             | ✅ MqttClient → IotService               | ❌ N/A               | ❌ N/A                 | ❌ N/A          | ✅ auto-persists + broadcasts | ✅ COMPLETE  |
| Sensor statistics (avg/min/max) | ❌                                        | ❌                   | ❌                     | ❌              | ❌                            | ❌ MISSING   |
| Sensor calibration              | ❌                                        | ❌                   | ❌                     | ❌              | ❌                            | ❌ MISSING   |
| Filter sensors by type/station  | ✅ backend                                | ❌ no filter UI      | ✅                     | ❌              | ❌                            | ⚠️ PARTIAL |

---

## 4. Alerts System


| Feature                            | Backend                          | Frontend                | API             | UI                        | Realtime                 | Status       |
| ---------------------------------- | -------------------------------- | ----------------------- | --------------- | ------------------------- | ------------------------ | ------------ |
| Create alert (manual)              | ✅                               | ❌                      | ✅ POST /alerts | ❌                        | ❌                       | ⚠️ PARTIAL |
| Auto-create on threshold violation | ✅ IotService                    | ❌ N/A                  | ❌ N/A          | ❌ N/A                    | ✅ alert-created WS      | ✅ COMPLETE  |
| List alerts (paginated)            | ✅                               | ❌                      | ✅ GET /alerts  | 🔴**AlertsPage is empty** | ❌                       | 🔴 BROKEN    |
| Filter alerts by severity/status   | ✅                               | ❌                      | ✅              | ❌                        | ❌                       | ⚠️ PARTIAL |
| Acknowledge alert                  | ✅ PATCH /alerts/:id/acknowledge | ❌                      | ✅              | ❌                        | ❌                       | ⚠️ PARTIAL |
| Resolve alert                      | ✅ PATCH /alerts/:id/resolve     | ❌                      | ✅              | ❌                        | ❌                       | ⚠️ PARTIAL |
| Alert detail view                  | ✅ GET /alerts/:id               | ❌                      | ✅              | ❌                        | ❌                       | ⚠️ PARTIAL |
| Alert feed on dashboard            | ✅                               | ✅ AlertsFeed component | ✅ fetchAlerts  | ✅                        | ✅ alertRealtimeReceived | ✅ COMPLETE  |
| Email notification on alert        | ❌                               | ❌                      | ❌              | ❌                        | ❌                       | ❌ MISSING   |
| SMS notification on alert          | ❌                               | ❌                      | ❌              | ❌                        | ❌                       | ❌ MISSING   |
| Alert rules engine                 | ❌                               | ❌                      | ❌              | ❌                        | ❌                       | ❌ MISSING   |
| Alert history export               | ❌                               | ❌                      | ❌              | ❌                        | ❌                       | ❌ MISSING   |

---

## 5. Maintenance Management


| Feature                    | Backend                  | Frontend | API                        | UI                 | Realtime | Status       |
| -------------------------- | ------------------------ | -------- | -------------------------- | ------------------ | -------- | ------------ |
| List maintenance orders    | ✅                       | ✅       | ✅ GET /maintenance        | ✅ read-only table | ❌       | ⚠️ PARTIAL |
| Create maintenance order   | ✅                       | ❌       | ✅ POST /maintenance       | ❌ no create UI    | ❌       | ⚠️ PARTIAL |
| Update maintenance order   | ✅                       | ❌       | ✅ PATCH /maintenance/:id  | ❌ no edit UI      | ❌       | ⚠️ PARTIAL |
| Delete maintenance order   | ✅                       | ❌       | ✅ DELETE /maintenance/:id | ❌                 | ❌       | ⚠️ PARTIAL |
| Assign to technician       | ✅ entity has assignedTo | ❌       | ✅                         | ❌                 | ❌       | ⚠️ PARTIAL |
| Filter by priority/status  | ✅                       | ❌       | ✅                         | ❌ no filter UI    | ❌       | ⚠️ PARTIAL |
| Maintenance history        | ✅ entity tracks dates   | ❌       | ✅                         | ❌                 | ❌       | ⚠️ PARTIAL |
| Predictive maintenance     | ❌                       | ❌       | ❌                         | ❌                 | ❌       | ❌ MISSING   |
| Maintenance calendar view  | ❌                       | ❌       | ❌                         | ❌                 | ❌       | ❌ MISSING   |
| Export maintenance reports | ❌                       | ❌       | ❌                         | ❌                 | ❌       | ❌ MISSING   |

---

## 6. Dashboard


| Feature                                             | Backend                             | Frontend           | API | UI | Realtime              | Status      |
| --------------------------------------------------- | ----------------------------------- | ------------------ | --- | -- | --------------------- | ----------- |
| KPI cards (active stations, pressure, flow, alerts) | ✅ derived from station/sensor data | ✅ KPISection      | ✅  | ✅ | ⚠️ not live-updated | ✅ COMPLETE |
| Station overview table                              | ✅                                  | ✅ StationOverview | ✅  | ✅ | ❌                    | ✅ COMPLETE |
| Active alerts feed                                  | ✅                                  | ✅ AlertsFeed      | ✅  | ✅ | ✅ via WS             | ✅ COMPLETE |
| Realtime stats panel                                | ✅ realtime slice                   | ✅ RealtimeStats   | ✅  | ✅ | ✅                    | ✅ COMPLETE |
| Charts / time series                                | ❌                                  | ❌                 | ❌  | ❌ | ❌                    | ❌ MISSING  |
| Energy metrics section                              | ❌                                  | ❌                 | ❌  | ❌ | ❌                    | ❌ MISSING  |
| Water quality metrics                               | ❌                                  | ❌                 | ❌  | ❌ | ❌                    | ❌ MISSING  |
| Customizable widgets                                | ❌                                  | ❌                 | ❌  | ❌ | ❌                    | ❌ MISSING  |

---

## 7. Workflow Builder (Original + Extensions)


| Feature                                       | Backend                                   | Frontend                            | API                        | UI | Realtime | Status       |
| --------------------------------------------- | ----------------------------------------- | ----------------------------------- | -------------------------- | -- | -------- | ------------ |
| Drag-and-drop canvas (JointJS)                | ❌ N/A                                    | ✅                                  | ❌ N/A                     | ✅ | ❌       | ✅ COMPLETE  |
| Block sidebar with categories                 | ❌ N/A                                    | ✅ BlockSidebar                     | ❌ N/A                     | ✅ | ❌       | ✅ COMPLETE  |
| Node property editor                          | ❌ N/A                                    | ✅ PropertiesPanel, NodeEditorModal | ❌ N/A                     | ✅ | ❌       | ✅ COMPLETE  |
| Generic blocks (input/action/decision/output) | ✅ handlers                               | ✅ data/blocks.js                   | ✅                         | ✅ | ❌       | ✅ COMPLETE  |
| Workflow execution engine                     | ✅ WorkflowRunner, NodeExecutor, handlers | ✅ workflowExecutorClient           | ✅ POST /flows/:id/execute | ✅ | ❌       | ✅ COMPLETE  |
| Workflow save/load                            | ⚠️ in-memory Map only                   | ✅                                  | ✅ GET/POST /flows         | ✅ | ❌       | ⚠️ PARTIAL |
| Workflow persistence to DB                    | ❌ Workflow entity unused                 | ❌                                  | ❌                         | ❌ | ❌       | ❌ MISSING   |
| Execution history in DB                       | ❌ WorkflowExecution entity unused        | ❌                                  | ❌                         | ❌ | ❌       | ❌ MISSING   |
| Auth guard on /api/flows                      | ❌**all routes public**                   | ❌                                  | 🔴 SECURITY ISSUE          | ❌ | ❌       | 🔴 BROKEN    |
| Autosave                                      | ❌ N/A                                    | ✅ autosaveManager                  | ✅                         | ✅ | ❌       | ✅ COMPLETE  |
| Graph serialization                           | ❌ N/A                                    | ✅ graphSerializer/Deserializer     | ❌ N/A                     | ✅ | ❌       | ✅ COMPLETE  |
| Industrial block: Sensor Read                 | ❌                                        | ❌                                  | ❌                         | ❌ | ❌       | ❌ MISSING   |
| Industrial block: Threshold Check             | ❌                                        | ❌                                  | ❌                         | ❌ | ❌       | ❌ MISSING   |
| Industrial block: Pump Control                | ❌                                        | ❌                                  | ❌                         | ❌ | ❌       | ❌ MISSING   |
| Industrial block: Alert Trigger               | ❌                                        | ❌                                  | ❌                         | ❌ | ❌       | ❌ MISSING   |
| Industrial block: MQTT Publish                | ❌                                        | ❌                                  | ❌                         | ❌ | ❌       | ❌ MISSING   |
| Industrial block: Station Control             | ❌                                        | ❌                                  | ❌                         | ❌ | ❌       | ❌ MISSING   |
| Workflow scheduling/triggers                  | ❌                                        | ❌                                  | ❌                         | ❌ | ❌       | ❌ MISSING   |
| Workflow triggered by MQTT event              | ❌                                        | ❌                                  | ❌                         | ❌ | ❌       | ❌ MISSING   |

---

## 8. Real-time Infrastructure


| Feature                       | Backend                     | Frontend                    | API    | UI                | Realtime       | Status       |
| ----------------------------- | --------------------------- | --------------------------- | ------ | ----------------- | -------------- | ------------ |
| Socket.IO server              | ✅ RealtimeGateway          | ✅ useSocket hook           | ❌ N/A | ❌ N/A            | ✅             | ✅ COMPLETE  |
| JWT auth on WS connect        | ✅                          | ✅ sends token in auth      | ❌ N/A | ❌ N/A            | ✅             | ✅ COMPLETE  |
| Channel subscribe/unsubscribe | ✅                          | ✅                          | ❌ N/A | ❌ N/A            | ✅             | ✅ COMPLETE  |
| sensor-update event           | ✅ IotService broadcasts    | ✅ dispatches to Redux      | ❌ N/A | ✅ table updates  | ✅             | ✅ COMPLETE  |
| alert-created event           | ✅ AlertsService broadcasts | ✅ dispatches to Redux      | ❌ N/A | ✅ dashboard feed | ✅             | ✅ COMPLETE  |
| station-status event          | ❌ never emitted            | ✅ listener in useSocket    | ❌ N/A | ❌                | ❌ signal lost | 🔴 BROKEN    |
| workflow-event broadcast      | ❌                          | ❌                          | ❌     | ❌                | ❌             | ❌ MISSING   |
| Ping/heartbeat                | ✅ ping handler             | ❌ not called               | ❌ N/A | ❌                | ✅ server-side | ⚠️ PARTIAL |
| Room-based broadcasting       | ✅ channel rooms            | ✅ subscribes to 4 channels | ❌ N/A | ❌                | ✅             | ✅ COMPLETE  |

---

## 9. MQTT Integration


| Feature                          | Backend                | Frontend                   | API          | UI       | Realtime | Status       |
| -------------------------------- | ---------------------- | -------------------------- | ------------ | -------- | -------- | ------------ |
| MQTT client connect/reconnect    | ✅                     | ❌ N/A                     | ❌ N/A       | ❌ N/A   | ✅       | ✅ COMPLETE  |
| Subscribe sensor/+/data topic    | ✅                     | ❌ N/A                     | ❌ N/A       | ❌ N/A   | ✅       | ✅ COMPLETE  |
| Parse & persist sensor readings  | ✅                     | ❌ N/A                     | ❌ N/A       | ❌ N/A   | ✅       | ✅ COMPLETE  |
| Threshold alert on MQTT data     | ✅                     | ❌ N/A                     | ❌ N/A       | ❌ N/A   | ✅       | ✅ COMPLETE  |
| Publish commands to devices      | ✅ publish() method    | ❌ N/A                     | ❌ no caller | ❌ no UI | ❌       | ⚠️ PARTIAL |
| Device status/heartbeat          | ✅ subscribes to topic | ⚠️ parsed but no handler | ❌ N/A       | ❌       | ❌       | ⚠️ PARTIAL |
| MQTT connection status display   | ❌                     | ❌                         | ❌           | ❌       | ❌       | ❌ MISSING   |
| MQTT topic configuration UI      | ❌                     | ❌                         | ❌           | ❌       | ❌       | ❌ MISSING   |
| MQTT publish from workflow block | ❌                     | ❌                         | ❌           | ❌       | ❌       | ❌ MISSING   |

---

## 10. Analytics & Reporting


| Feature                   | Backend | Frontend | API | UI | Realtime | Status     |
| ------------------------- | ------- | -------- | --- | -- | -------- | ---------- |
| Analytics API             | ❌      | ❌       | ❌  | ❌ | ❌       | ❌ MISSING |
| Consumption trends chart  | ❌      | ❌       | ❌  | ❌ | ❌       | ❌ MISSING |
| Pressure history graph    | ❌      | ❌       | ❌  | ❌ | ❌       | ❌ MISSING |
| Water quality analytics   | ❌      | ❌       | ❌  | ❌ | ❌       | ❌ MISSING |
| Energy efficiency metrics | ❌      | ❌       | ❌  | ❌ | ❌       | ❌ MISSING |
| Reports module            | ❌      | ❌       | ❌  | ❌ | ❌       | ❌ MISSING |
| CSV export                | ❌      | ❌       | ❌  | ❌ | ❌       | ❌ MISSING |
| PDF export                | ❌      | ❌       | ❌  | ❌ | ❌       | ❌ MISSING |
| Scheduled reports         | ❌      | ❌       | ❌  | ❌ | ❌       | ❌ MISSING |

---

## 11. Notifications


| Feature                  | Backend      | Frontend | API    | UI     | Realtime | Status       |
| ------------------------ | ------------ | -------- | ------ | ------ | -------- | ------------ |
| Notification entity      | ✅ (defined) | ❌ N/A   | ❌ N/A | ❌ N/A | ❌ N/A   | ⚠️ PARTIAL |
| NotificationsModule      | ❌           | ❌       | ❌     | ❌     | ❌       | ❌ MISSING   |
| Email notifications      | ❌           | ❌       | ❌     | ❌     | ❌       | ❌ MISSING   |
| SMS notifications        | ❌           | ❌       | ❌     | ❌     | ❌       | ❌ MISSING   |
| In-app notification bell | ❌           | ❌       | ❌     | ❌     | ❌       | ❌ MISSING   |
| Notification preferences | ❌           | ❌       | ❌     | ❌     | ❌       | ❌ MISSING   |

---

## 12. Infrastructure & DevOps


| Feature                                   | Backend             | Frontend                      | Config        | Status       |
| ----------------------------------------- | ------------------- | ----------------------------- | ------------- | ------------ |
| Docker Compose (Postgres/Redis/Mosquitto) | ✅                  | ✅                            | ✅            | ✅ COMPLETE  |
| Backend Dockerfile                        | ❌                  | ❌ N/A                        | ❌            | ❌ MISSING   |
| Frontend Dockerfile                       | ❌ N/A              | ❌                            | ❌            | ❌ MISSING   |
| Backend in docker-compose                 | ❌                  | ❌ N/A                        | ❌            | ❌ MISSING   |
| Frontend in docker-compose                | ❌ N/A              | ❌                            | ❌            | ❌ MISSING   |
| Environment variables (.env)              | ✅                  | ✅                            | ✅            | ✅ COMPLETE  |
| TypeORM migrations                        | ❌                  | ❌ N/A                        | ❌            | ❌ MISSING   |
| Database seeding                          | ✅ seed.ts exists   | ❌ N/A                        | ⚠️          | ⚠️ PARTIAL |
| Health check endpoint                     | ❌ no AppController | ❌ N/A                        | ❌            | ❌ MISSING   |
| CI/CD pipeline                            | ❌                  | ✅ .github/workflows/main.yml | ⚠️          | ⚠️ PARTIAL |
| Unit tests                                | ❌                  | ❌                            | ❌            | ❌ MISSING   |
| Integration tests                         | ❌                  | ❌                            | ❌            | ❌ MISSING   |
| Redis integration                         | ❌ unused           | ❌ N/A                        | ✅ in compose | 🔴 BROKEN    |
| API documentation (Swagger)               | ❌                  | ❌ N/A                        | ❌            | ❌ MISSING   |

---

## Summary Counts


| Status                     | Count   |
| -------------------------- | ------- |
| ✅ COMPLETE                | 38      |
| ⚠️ PARTIAL               | 29      |
| ❌ MISSING                 | 54      |
| 🔴 BROKEN                  | 5       |
| **Total features tracked** | **126** |

**Overall implementation: ~30% complete (38/126 fully done)**

Generated from the real codebase in `C:\Users\Grous info\Downloads\pfe` on 2026-05-10.

Status values:

- COMPLETE: implemented end-to-end with backend, frontend, API/UI, and integration appropriate for current scope.
- PARTIAL: meaningful implementation exists, but important roadmap behavior is missing.
- MISSING: no meaningful implementation found.
- BROKEN: implemented in a way that is known to be inconsistent or likely nonfunctional.


| Feature                       | Backend                                                                                          | Frontend                                                                    | API                                      | UI                                      | Realtime                           | Status   |
| ----------------------------- | ------------------------------------------------------------------------------------------------ | --------------------------------------------------------------------------- | ---------------------------------------- | --------------------------------------- | ---------------------------------- | -------- |
| Project documentation         | Architecture, roadmap, setup, checklist, deliverables exist                                      | N/A                                                                         | N/A                                      | N/A                                     | N/A                                | COMPLETE |
| NestJS backend shell          | `AppModule`, `main.ts`, ConfigModule, global API prefix                                          | N/A                                                                         | `/api` prefix                            | N/A                                     | N/A                                | COMPLETE |
| React frontend shell          | N/A                                                                                              | CRA/Argon Dashboard app with layouts/routes                                 | N/A                                      | Admin/Auth layouts                      | N/A                                | COMPLETE |
| Database configuration        | TypeORM Postgres configured                                                                      | N/A                                                                         | Repository injection works               | N/A                                     | N/A                                | PARTIAL  |
| Database migrations           | Migration path configured and migration status documented                                        | N/A                                                                         | N/A                                      | N/A                                     | N/A                                | PARTIAL  |
| Core entities                 | User, Station, Sensor, SensorData, Alert, Maintenance, Workflow, WorkflowExecution, Notification | N/A                                                                         | N/A                                      | N/A                                     | N/A                                | COMPLETE |
| User authentication           | Auth service/controller, JWT strategy, bcrypt                                                    | Auth pages, auth slice, ProtectedRoute                                      | login/register/me/logout                 | Login/Register                          | Token passed to socket client      | PARTIAL  |
| Refresh tokens                | JWT refresh endpoint/service exists                                                              | Axios interceptor retries 401 through refresh endpoint                      | `POST /auth/refresh`                     | N/A                                     | N/A                                | PARTIAL  |
| RBAC                          | JwtGuard, RolesGuard, Roles decorator                                                            | ProtectedRoute checks auth; selected actions hidden by role                 | Applied to CRUD modules                  | Station/sensor/alert actions role-gated | N/A                                | PARTIAL  |
| API client                    | N/A                                                                                              | Axios client with bearer interceptor                                        | Calls backend`/api`                      | N/A                                     | N/A                                | PARTIAL  |
| Redux store                   | N/A                                                                                              | auth, dashboard, realtime, stations, sensors, alerts, maintenance slices    | N/A                                      | Used by current pages                   | Realtime slice exists              | PARTIAL  |
| UI slice/theme                | N/A                                                                                              | No`uiSlice` found                                                           | N/A                                      | Existing Argon styles                   | N/A                                | MISSING  |
| Dashboard                     | Uses existing stations/sensors/alerts APIs                                                       | Dashboard derives KPIs, station overview, and alert feed from real API data | Reuses CRUD APIs                         | Present                                 | Socket hook subscribed             | PARTIAL  |
| Station CRUD                  | Stations module/service/controller/DTOs                                                          | Stations page with list/filter/create/edit                                  | GET/POST/PATCH/DELETE                    | List and modal form                     | Station status event listener only | PARTIAL  |
| Station details               | Backend detail endpoint exists                                                                   | No detail page found                                                        | GET`/stations/:id`                       | Missing                                 | N/A                                | PARTIAL  |
| Station analytics             | No backend endpoint                                                                              | No station analytics page                                                   | Missing                                  | Missing                                 | Missing                            | MISSING  |
| Sensor CRUD                   | Sensors module/service/controller/DTOs                                                           | Monitoring page can list/create sensors                                     | GET/POST/PATCH/DELETE                    | Basic list/create modal                 | Sensor update listener             | PARTIAL  |
| Sensor data history           | SensorData entity and`GET /sensors/:id/data`                                                     | Service exists; no detail chart page found                                  | Present                                  | Minimal/no dedicated view               | Updates via socket                 | PARTIAL  |
| Live monitoring dashboard     | `IotService.processSensorData` can save/broadcast if called                                      | Monitoring page lists sensors                                               | Sensor APIs                              | No live chart/gauge components found    | Partial socket handling            | PARTIAL  |
| Alerts                        | Alerts module/service/controller/DTOs                                                            | Alerts page list/ack/resolve                                                | GET/POST/PATCH acknowledge/resolve       | Present                                 | `alert-created` listener           | PARTIAL  |
| Alert delete/clear            | No delete endpoint                                                                               | No clear button                                                             | Missing                                  | Missing                                 | N/A                                | MISSING  |
| Maintenance                   | Maintenance module/service/controller/DTOs                                                       | Maintenance page list only                                                  | GET/POST/PATCH/DELETE                    | List                                    | No realtime maintenance listener   | PARTIAL  |
| Technician assignment         | No dedicated assign endpoint                                                                     | No assignment page                                                          | Missing`/maintenance/:id/assign`         | Missing                                 | Missing                            | MISSING  |
| WebSocket gateway             | Gateway/service exists                                                                           | `useSocket` exists                                                          | Socket.IO events                         | Used in dashboard/monitoring            | Present                            | PARTIAL  |
| Socket authentication         | Gateway validates JWT from handshake auth                                                        | Token sent in`auth`                                                         | N/A                                      | N/A                                     | Validated on connection            | PARTIAL  |
| MQTT client                   | Connects/subscribes/publishes                                                                    | N/A                                                                         | No HTTP API                              | N/A                                     | Not fully bridged                  | PARTIAL  |
| MQTT ingestion to DB          | MQTT client delegates valid sensor data to`IotService`                                           | N/A                                                                         | N/A                                      | N/A                                     | Broadcasts sensor updates          | PARTIAL  |
| Threshold alert creation      | Threshold violations create persistent alerts through`AlertsService`                             | Frontend listens to alerts                                                  | Alert records available through`/alerts` | Alert UI exists                         | Emits`alert-created`               | PARTIAL  |
| Workflow builder UI           | Existing builder components/registry/engine                                                      | Builder components exist                                                    | Save/execute service exists              | Route inconsistent                      | N/A                                | PARTIAL  |
| Workflow route integration    | `/flows` backend exists                                                                          | `/admin/builder` renders `Test`, not imported `BuilderPage`                 | `/flows` endpoints                       | Inconsistent                            | N/A                                | BROKEN   |
| Workflow persistence          | Workflow entities exist                                                                          | Save service exists                                                         | `/flows` stores in memory                | N/A                                     | N/A                                | BROKEN   |
| Workflow execution            | Generic handlers for input/action/decision/output                                                | Generic blocks exist                                                        | `/flows/execute`                         | Builder tooling exists                  | No event triggers                  | PARTIAL  |
| Industrial workflow blocks    | No industrial handlers                                                                           | No sensor/alert/maintenance/MQTT blocks                                     | Missing                                  | Missing                                 | Missing                            | MISSING  |
| Automation module             | No`automation` backend module                                                                    | No`modules/automation`                                                      | Missing planned`/workflows`              | Missing workflow list/logs              | Missing                            | MISSING  |
| GIS map                       | No module                                                                                        | No`modules/map`; old Argon map route commented                              | Missing                                  | Missing                                 | Missing                            | MISSING  |
| Analytics                     | No module                                                                                        | No`modules/analytics`                                                       | Missing`/analytics/*`                    | Missing                                 | Missing                            | MISSING  |
| Reports                       | No module                                                                                        | No`modules/reports`                                                         | Missing`/reports/*`                      | Missing                                 | Missing                            | MISSING  |
| Notifications                 | Entity only                                                                                      | No`modules/notifications`                                                   | Missing                                  | Missing                                 | Missing                            | MISSING  |
| IoT device management         | MQTT client only                                                                                 | No`modules/iot`                                                             | Missing                                  | Missing                                 | Partial MQTT only                  | MISSING  |
| Email/SMS channels            | No services/channels                                                                             | No preferences UI                                                           | Missing                                  | Missing                                 | Missing                            | MISSING  |
| Docker Compose infrastructure | Postgres, Redis, Mosquitto                                                                       | N/A                                                                         | N/A                                      | N/A                                     | MQTT broker                        | PARTIAL  |
| Backend Dockerfile            | Missing                                                                                          | N/A                                                                         | N/A                                      | N/A                                     | N/A                                | MISSING  |
| Frontend Dockerfile           | N/A                                                                                              | Missing                                                                     | N/A                                      | N/A                                     | N/A                                | MISSING  |
| Redis caching                 | Redis provisioned                                                                                | N/A                                                                         | No usage found                           | N/A                                     | No Socket adapter                  | MISSING  |
| Testing                       | No test files found in source scan                                                               | No component tests found                                                    | No API tests                             | No UI tests                             | No realtime tests                  | MISSING  |
| Backend build/type check      | `npx.cmd tsc --noEmit` succeeds                                                                  | N/A                                                                         | N/A                                      | N/A                                     | N/A                                | COMPLETE |
| Frontend production build     | N/A                                                                                              | `npm.cmd run build` succeeds with warnings                                  | N/A                                      | Build artifact generated                | N/A                                | PARTIAL  |
| Lint cleanliness              | N/A                                                                                              | Build reports unused imports                                                | N/A                                      | N/A                                     | N/A                                | PARTIAL  |
| Environment examples          | Docs include templates                                                                           | No`.env.example` observed in scanned file list                              | N/A                                      | N/A                                     | N/A                                | MISSING  |
| Production observability      | Logger usage only                                                                                | N/A                                                                         | No health/metrics                        | N/A                                     | No metrics                         | MISSING  |
