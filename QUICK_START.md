# AquaFlow: Quick Start Guide

## Overview

This guide provides quick reference code snippets to jumpstart implementation of AquaFlow's core features. Use this alongside the detailed architecture and roadmap documents.

---

## 1. Database Setup (TypeORM)

### Installation

```bash
cd backend
npm install typeorm pg @nestjs/typeorm
```

### Database Module Configuration

```typescript
// backend/src/database/database.module.ts

import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { User } from './entities/User.entity';
import { Station } from './entities/Station.entity';
import { Sensor } from './entities/Sensor.entity';
import { SensorData } from './entities/SensorData.entity';
import { Alert } from './entities/Alert.entity';
import { Maintenance } from './entities/Maintenance.entity';
import { Workflow } from './entities/Workflow.entity';
import { Notification } from './entities/Notification.entity';

@Module({
  imports: [
    TypeOrmModule.forRoot({
      type: 'postgres',
      host: process.env.DATABASE_HOST || 'localhost',
      port: parseInt(process.env.DATABASE_PORT) || 5432,
      username: process.env.DATABASE_USER || 'postgres',
      password: process.env.DATABASE_PASSWORD || 'postgres',
      database: process.env.DATABASE_NAME || 'aquaflow',
      entities: [
        User,
        Station,
        Sensor,
        SensorData,
        Alert,
        Maintenance,
        Workflow,
        Notification,
      ],
      synchronize: process.env.NODE_ENV !== 'production',
      logging: process.env.NODE_ENV === 'development',
    }),
    TypeOrmModule.forFeature([
      User,
      Station,
      Sensor,
      SensorData,
      Alert,
      Maintenance,
      Workflow,
      Notification,
    ]),
  ],
  exports: [TypeOrmModule],
})
export class DatabaseModule {}
```

### Example Entity

```typescript
// backend/src/database/entities/Station.entity.ts

import {
  Entity,
  PrimaryGeneratedColumn,
  Column,
  CreateDateColumn,
  UpdateDateColumn,
  OneToMany,
  ManyToOne,
  JoinColumn,
} from 'typeorm';
import { User } from './User.entity';
import { Sensor } from './Sensor.entity';
import { Alert } from './Alert.entity';

export enum StationStatus {
  NORMAL = 'normal',
  WARNING = 'warning',
  CRITICAL = 'critical',
  OFFLINE = 'offline',
}

@Entity('stations')
export class Station {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column({ type: 'varchar', length: 255 })
  name: string;

  @Column({ type: 'varchar', length: 255 })
  location: string;

  @Column({ type: 'decimal', precision: 10, scale: 6 })
  latitude: number;

  @Column({ type: 'decimal', precision: 10, scale: 6 })
  longitude: number;

  @Column({ type: 'decimal', precision: 10, scale: 2 })
  capacity: number;

  @Column({ type: 'enum', enum: StationStatus, default: StationStatus.NORMAL })
  status: StationStatus;

  @Column({ type: 'text', nullable: true })
  description: string;

  @ManyToOne(() => User, { nullable: true })
  @JoinColumn({ name: 'created_by' })
  createdBy: User;

  @OneToMany(() => Sensor, (sensor) => sensor.station)
  sensors: Sensor[];

  @OneToMany(() => Alert, (alert) => alert.station)
  alerts: Alert[];

  @CreateDateColumn()
  createdAt: Date;

  @UpdateDateColumn()
  updatedAt: Date;
}
```

---

## 2. Authentication (JWT)

### Installation

```bash
npm install @nestjs/passport passport passport-jwt @nestjs/jwt bcrypt
npm install -D @types/bcrypt
```

### JWT Strategy

```typescript
// backend/src/auth/strategies/jwt.strategy.ts

import { Injectable } from '@nestjs/common';
import { PassportStrategy } from '@nestjs/passport';
import { ExtractJwt, Strategy } from 'passport-jwt';
import { User } from '../../database/entities/User.entity';
import { Repository } from 'typeorm';
import { InjectRepository } from '@nestjs/typeorm';

@Injectable()
export class JwtStrategy extends PassportStrategy(Strategy) {
  constructor(
    @InjectRepository(User)
    private readonly userRepository: Repository<User>,
  ) {
    super({
      jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
      ignoreExpiration: false,
      secretOrKey: process.env.JWT_SECRET || 'your-secret-key',
    });
  }

  async validate(payload: any) {
    const user = await this.userRepository.findOne({
      where: { id: payload.sub },
    });
    return user;
  }
}
```

### Auth Service

```typescript
// backend/src/auth/auth.service.ts

import { Injectable, UnauthorizedException } from '@nestjs/common';
import { JwtService } from '@nestjs/jwt';
import { Repository } from 'typeorm';
import { InjectRepository } from '@nestjs/typeorm';
import { User, UserRole } from '../../database/entities/User.entity';
import * as bcrypt from 'bcrypt';
import { LoginDto } from './dto/login.dto';
import { RegisterDto } from './dto/register.dto';

@Injectable()
export class AuthService {
  constructor(
    @InjectRepository(User)
    private readonly userRepository: Repository<User>,
    private readonly jwtService: JwtService,
  ) {}

  async register(dto: RegisterDto) {
    const hashedPassword = await bcrypt.hash(dto.password, 10);
    const user = this.userRepository.create({
      email: dto.email,
      password: hashedPassword,
      firstname: dto.firstname,
      lastname: dto.lastname,
      role: UserRole.OPERATOR,
    });

    await this.userRepository.save(user);
    return { message: 'User registered successfully' };
  }

  async login(dto: LoginDto) {
    const user = await this.userRepository.findOne({
      where: { email: dto.email },
    });

    if (!user || !(await bcrypt.compare(dto.password, user.password))) {
      throw new UnauthorizedException('Invalid credentials');
    }

    const payload = { sub: user.id, email: user.email, role: user.role };
    return {
      access_token: this.jwtService.sign(payload),
      user: {
        id: user.id,
        email: user.email,
        firstname: user.firstname,
        lastname: user.lastname,
        role: user.role,
      },
    };
  }

  async refreshToken(userId: string) {
    const user = await this.userRepository.findOne({ where: { id: userId } });
    if (!user) throw new UnauthorizedException('User not found');

    const payload = { sub: user.id, email: user.email, role: user.role };
    return { access_token: this.jwtService.sign(payload) };
  }
}
```

---

## 3. Real-Time WebSocket (Socket.io)

### Installation

```bash
npm install @nestjs/websockets @nestjs/platform-socket.io socket.io
npm install socket.io-client # frontend
```

### WebSocket Gateway

```typescript
// backend/src/realtime/realtime.gateway.ts

import {
  WebSocketGateway,
  WebSocketServer,
  OnGatewayConnection,
  OnGatewayDisconnect,
  SubscribeMessage,
} from '@nestjs/websockets';
import { Server, Socket } from 'socket.io';
import { Injectable } from '@nestjs/common';
import { RealtimeService } from './realtime.service';

@WebSocketGateway({
  cors: {
    origin: process.env.FRONTEND_URL || 'http://localhost:3000',
    credentials: true,
  },
})
@Injectable()
export class RealtimeGateway
  implements OnGatewayConnection, OnGatewayDisconnect
{
  @WebSocketServer()
  server: Server;

  constructor(private readonly realtimeService: RealtimeService) {}

  handleConnection(client: Socket) {
    console.log(`Client connected: ${client.id}`);
    this.realtimeService.addConnection(client.id, client);
  }

  handleDisconnect(client: Socket) {
    console.log(`Client disconnected: ${client.id}`);
    this.realtimeService.removeConnection(client.id);
  }

  @SubscribeMessage('subscribe-sensor')
  subscribeSensor(client: Socket, data: { sensorId: string }) {
    const room = `sensor:${data.sensorId}`;
    client.join(room);
    return { success: true, message: `Subscribed to sensor ${data.sensorId}` };
  }

  broadcastSensorUpdate(sensorId: string, data: any) {
    this.server.to(`sensor:${sensorId}`).emit('sensor-update', data);
  }

  broadcastAlert(data: any) {
    this.server.emit('alert-created', data);
  }
}
```

### Frontend Socket Hook

```javascript
// frontend/src/hooks/useSocket.js

import { useEffect, useRef } from 'react';
import { useDispatch } from 'react-redux';
import io from 'socket.io-client';

export function useSocket() {
  const socketRef = useRef(null);
  const dispatch = useDispatch();

  useEffect(() => {
    const socket = io(process.env.REACT_APP_WS_URL || 'http://localhost:3001', {
      auth: {
        token: localStorage.getItem('access_token'),
      },
    });

    socketRef.current = socket;

    socket.on('sensor-update', (data) => {
      dispatch({ type: 'realtime/sensorUpdate', payload: data });
    });

    socket.on('alert-created', (data) => {
      dispatch({ type: 'realtime/alertCreated', payload: data });
    });

    return () => {
      socket.disconnect();
    };
  }, [dispatch]);

  const emit = (event, data) => {
    if (socketRef.current) {
      socketRef.current.emit(event, data);
    }
  };

  const subscribe = (event, callback) => {
    if (socketRef.current) {
      socketRef.current.on(event, callback);
    }
  };

  return { socket: socketRef.current, emit, subscribe };
}
```

---

## 4. Redux State Management

### Installation

```bash
cd frontend
npm install @reduxjs/toolkit react-redux
```

### Redux Store Setup

```javascript
// frontend/src/store/store.js

import { configureStore } from '@reduxjs/toolkit';
import authReducer from './slices/authSlice';
import stationsReducer from './slices/stationsSlice';
import sensorsReducer from './slices/sensorsSlice';
import alertsReducer from './slices/alertsSlice';
import realtimeReducer from './slices/realtimeSlice';

export const store = configureStore({
  reducer: {
    auth: authReducer,
    stations: stationsReducer,
    sensors: sensorsReducer,
    alerts: alertsReducer,
    realtime: realtimeReducer,
  },
});

export const RootState = store.getState;
export const AppDispatch = store.dispatch;
```

### Example Slice (Auth)

```javascript
// frontend/src/store/slices/authSlice.js

import { createSlice, createAsyncThunk } from '@reduxjs/toolkit';
import * as authService from '../../services/authService';

export const loginUser = createAsyncThunk(
  'auth/loginUser',
  async (credentials) => {
    const response = await authService.login(credentials);
    localStorage.setItem('access_token', response.access_token);
    return response.user;
  },
);

export const registerUser = createAsyncThunk(
  'auth/registerUser',
  async (data) => {
    await authService.register(data);
    return { message: 'Registration successful' };
  },
);

const authSlice = createSlice({
  name: 'auth',
  initialState: {
    user: null,
    token: localStorage.getItem('access_token'),
    isAuthenticated: !!localStorage.getItem('access_token'),
    loading: false,
    error: null,
  },
  reducers: {
    logout: (state) => {
      state.user = null;
      state.token = null;
      state.isAuthenticated = false;
      localStorage.removeItem('access_token');
    },
  },
  extraReducers: (builder) => {
    builder
      .addCase(loginUser.pending, (state) => {
        state.loading = true;
      })
      .addCase(loginUser.fulfilled, (state, action) => {
        state.loading = false;
        state.user = action.payload;
        state.isAuthenticated = true;
      })
      .addCase(loginUser.rejected, (state, action) => {
        state.loading = false;
        state.error = action.error.message;
      });
  },
});

export const { logout } = authSlice.actions;
export default authSlice.reducer;
```

---

## 5. MQTT Integration

### Installation

```bash
npm install mqtt
```

### MQTT Client

```typescript
// backend/src/iot/mqtt/mqtt.client.ts

import { Injectable, Logger } from '@nestjs/common';
import * as mqtt from 'mqtt';

@Injectable()
export class MqttClient {
  private readonly logger = new Logger(MqttClient.name);
  private client: mqtt.MqttClient;

  connect() {
    const brokerUrl = process.env.MQTT_BROKER_URL || 'mqtt://localhost:1883';

    this.client = mqtt.connect(brokerUrl, {
      username: process.env.MQTT_USERNAME,
      password: process.env.MQTT_PASSWORD,
    });

    this.client.on('connect', () => {
      this.logger.log('Connected to MQTT broker');
      // Subscribe to sensor topics
      this.client.subscribe('sensors/+/data', (err) => {
        if (err) this.logger.error('Subscription error:', err);
      });
    });

    this.client.on('message', (topic, message) => {
      this.handleSensorMessage(topic, message);
    });
  }

  private handleSensorMessage(topic: string, message: Buffer) {
    try {
      const data = JSON.parse(message.toString());
      const sensorId = topic.split('/')[1];
      // Emit to realtime gateway or process sensor data
      this.logger.debug(
        `Received sensor data from ${sensorId}:`,
        data,
      );
    } catch (error) {
      this.logger.error('Error parsing MQTT message:', error);
    }
  }

  publish(topic: string, payload: any) {
    this.client.publish(topic, JSON.stringify(payload));
  }

  disconnect() {
    this.client.end();
  }
}
```

---

## 6. CRUD Example (Stations)

### Station Service

```typescript
// backend/src/stations/stations.service.ts

import { Injectable, NotFoundException } from '@nestjs/common';
import { Repository } from 'typeorm';
import { InjectRepository } from '@nestjs/typeorm';
import { Station } from '../database/entities/Station.entity';
import { CreateStationDto } from './dto/create-station.dto';
import { UpdateStationDto } from './dto/update-station.dto';

@Injectable()
export class StationsService {
  constructor(
    @InjectRepository(Station)
    private readonly stationRepository: Repository<Station>,
  ) {}

  async create(dto: CreateStationDto, userId: string) {
    const station = this.stationRepository.create({
      ...dto,
      createdBy: { id: userId },
    });
    return this.stationRepository.save(station);
  }

  async findAll(page = 1, limit = 10) {
    const [data, total] = await this.stationRepository.findAndCount({
      skip: (page - 1) * limit,
      take: limit,
    });
    return { data, total, page, limit };
  }

  async findById(id: string) {
    const station = await this.stationRepository.findOne({
      where: { id },
      relations: ['sensors', 'createdBy'],
    });
    if (!station) throw new NotFoundException('Station not found');
    return station;
  }

  async update(id: string, dto: UpdateStationDto) {
    const station = await this.findById(id);
    Object.assign(station, dto);
    return this.stationRepository.save(station);
  }

  async delete(id: string) {
    const station = await this.findById(id);
    return this.stationRepository.remove(station);
  }
}
```

### Station Controller

```typescript
// backend/src/stations/stations.controller.ts

import {
  Controller,
  Get,
  Post,
  Body,
  Param,
  Put,
  Delete,
  UseGuards,
  Query,
} from '@nestjs/common';
import { JwtGuard } from '../common/guards/JwtGuard';
import { StationsService } from './stations.service';
import { CreateStationDto } from './dto/create-station.dto';
import { UpdateStationDto } from './dto/update-station.dto';
import { CurrentUser } from '../common/decorators/CurrentUser.decorator';

@Controller('api/stations')
export class StationsController {
  constructor(private readonly stationsService: StationsService) {}

  @Post()
  @UseGuards(JwtGuard)
  create(@Body() dto: CreateStationDto, @CurrentUser() user: any) {
    return this.stationsService.create(dto, user.id);
  }

  @Get()
  @UseGuards(JwtGuard)
  findAll(@Query('page') page = 1, @Query('limit') limit = 10) {
    return this.stationsService.findAll(page, limit);
  }

  @Get(':id')
  @UseGuards(JwtGuard)
  findById(@Param('id') id: string) {
    return this.stationsService.findById(id);
  }

  @Put(':id')
  @UseGuards(JwtGuard)
  update(@Param('id') id: string, @Body() dto: UpdateStationDto) {
    return this.stationsService.update(id, dto);
  }

  @Delete(':id')
  @UseGuards(JwtGuard)
  delete(@Param('id') id: string) {
    return this.stationsService.delete(id);
  }
}
```

---

## 7. Frontend Module Structure

### Stations Module

```javascript
// frontend/src/modules/stations/pages/StationsListPage.jsx

import { useEffect } from 'react';
import { useDispatch, useSelector } from 'react-redux';
import { fetchStations } from '../../../store/slices/stationsSlice';
import StationCard from '../components/StationCard';

export default function StationsListPage() {
  const dispatch = useDispatch();
  const { stations, loading } = useSelector((state) => state.stations);

  useEffect(() => {
    dispatch(fetchStations());
  }, [dispatch]);

  if (loading) return <div>Loading...</div>;

  return (
    <div className="p-8">
      <h1 className="text-3xl font-bold mb-8">Stations</h1>
      <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-6">
        {stations.map((station) => (
          <StationCard key={station.id} station={station} />
        ))}
      </div>
    </div>
  );
}
```

---

## 8. Extended Workflow Block

### New Block Definition

```javascript
// Add to frontend/src/data/blocks.js

{
  type: "sensor-trigger",
  title: "Sensor Trigger",
  icon: "fa-microchip",
  category: "Industrial",
  description: "Triggers when sensor value changes",
  color: "#3b82f6",
  inputs: [],
  outputs: [{ id: "out", label: "Triggered" }],
  properties: [
    { name: "label", label: "Label", type: "text", defaultValue: "Sensor Trigger" },
    { name: "sensorId", label: "Sensor ID", type: "text", defaultValue: "" },
    { name: "operator", label: "Operator", type: "select", defaultValue: ">", options: [">", "<", "==", "!="] },
    { name: "threshold", label: "Threshold", type: "number", defaultValue: 0 },
  ],
},

{
  type: "alert-sender",
  title: "Alert Sender",
  icon: "fa-exclamation-triangle",
  category: "Industrial",
  description: "Creates an alert in the system",
  color: "#ef4444",
  inputs: [{ id: "in", label: "Trigger" }],
  outputs: [{ id: "out", label: "Alert Sent" }],
  properties: [
    { name: "label", label: "Label", type: "text", defaultValue: "Alert" },
    { name: "severity", label: "Severity", type: "select", defaultValue: "medium", options: ["low", "medium", "high", "critical"] },
    { name: "message", label: "Message", type: "textarea", defaultValue: "Alert message" },
  ],
},
```

### Handler Implementation

```typescript
// backend/src/execution/handlers/alert-sender.handler.ts

import { Injectable } from '@nestjs/common';
import { AlertsService } from '../../alerts/alerts.service';
import { RealtimeGateway } from '../../realtime/realtime.gateway';
import { WorkflowNode } from '../../common/types/workflow.types';
import { ExecutionContext } from '../engine/execution-context';

@Injectable()
export class AlertSenderHandler {
  constructor(
    private readonly alertsService: AlertsService,
    private readonly realtimeGateway: RealtimeGateway,
  ) {}

  async handle(
    node: WorkflowNode,
    input: any,
    context: ExecutionContext,
  ) {
    const { severity, message } = node.properties;

    const alert = await this.alertsService.create({
      severity,
      message,
      type: 'workflow-triggered',
    });

    // Broadcast alert via WebSocket
    this.realtimeGateway.broadcastAlert({
      alertId: alert.id,
      severity,
      message,
      timestamp: new Date(),
    });

    return { alertId: alert.id, sent: true };
  }
}
```

---

## 9. Environment Configuration

### .env Example

```env
# Database
DATABASE_HOST=localhost
DATABASE_PORT=5432
DATABASE_NAME=aquaflow
DATABASE_USER=postgres
DATABASE_PASSWORD=postgres
DATABASE_URL=postgresql://postgres:postgres@localhost:5432/aquaflow

# JWT
JWT_SECRET=your-very-secure-secret-key-change-this
JWT_EXPIRATION=3600

# MQTT
MQTT_BROKER_URL=mqtt://localhost:1883
MQTT_USERNAME=admin
MQTT_PASSWORD=password

# Server
PORT=3000
NODE_ENV=development

# Frontend
FRONTEND_URL=http://localhost:3000
REACT_APP_API_URL=http://localhost:3000/api
REACT_APP_WS_URL=ws://localhost:3001

# Mail (optional)
MAIL_HOST=smtp.gmail.com
MAIL_PORT=587
MAIL_USER=your-email@gmail.com
MAIL_PASS=your-app-password
```

---

## 10. Docker Compose Setup

### docker-compose.yml

```yaml
version: '3.8'

services:
  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: aquaflow
    ports:
      - '5432:5432'
    volumes:
      - postgres_data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    ports:
      - '6379:6379'
    volumes:
      - redis_data:/data

  mosquitto:
    image: eclipse-mosquitto:latest
    ports:
      - '1883:1883'
      - '9001:9001'
    volumes:
      - mosquitto_data:/mosquitto/data
      - ./mosquitto.conf:/mosquitto/config/mosquitto.conf
    environment:
      - MOSQUITTO_USERNAME=admin
      - MOSQUITTO_PASSWORD=password

volumes:
  postgres_data:
  redis_data:
  mosquitto_data:
```

---

## Getting Started Checklist

- [ ] Clone repository
- [ ] Install dependencies (backend & frontend)
- [ ] Set up `.env` files
- [ ] Start Docker services: `docker-compose up -d`
- [ ] Run database migrations
- [ ] Start backend: `npm run start:dev` (from backend/)
- [ ] Start frontend: `npm start` (from frontend/)
- [ ] Login with test credentials
- [ ] Create a test station
- [ ] Add test sensors
- [ ] View real-time data on dashboard

---

## Useful Commands

### Backend
```bash
# Start development server
npm run start:dev

# Build for production
npm build

# Run migrations
npm run typeorm migration:run

# Generate migration
npm run typeorm migration:generate -- -n MigrationName
```

### Frontend
```bash
# Start development server
npm start

# Build for production
npm run build

# Test
npm test
```

### Docker
```bash
# Start all services
docker-compose up -d

# View logs
docker-compose logs -f [service-name]

# Stop services
docker-compose down
```

---

## Next Steps

1. Review AQUAFLOW_ARCHITECTURE.md for detailed design
2. Review IMPLEMENTATION_ROADMAP.md for phased approach
3. Start with Phase 1 (Core Infrastructure)
4. Use this Quick Start guide for code templates
5. Refer to existing code patterns in the project

