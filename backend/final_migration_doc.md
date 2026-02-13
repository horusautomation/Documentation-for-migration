# Plan de Migración HSE Backend - Versión Final

## Tabla de Contenidos

1. [Introducción y Contexto](#1-introducción-y-contexto)
2. [Arquitectura General](#2-arquitectura-general)
3. [Stack Tecnológico](#3-stack-tecnológico)
4. [Módulos del Sistema](#4-módulos-del-sistema)
5. [Modelo de Datos](#5-modelo-de-datos)
6. [Reglas de Negocio y Flujos](#6-reglas-de-negocio-y-flujos)
7. [Arquitectura CQRS en Devices Service](#7-arquitectura-cqrs-en-devices-service)
8. [Patrones Críticos de Implementación](#8-patrones-críticos-de-implementación)
9. [Estructura de Carpetas](#9-estructura-de-carpetas)
10. [Infraestructura Cloud](#10-infraestructura-cloud)
11. [Estrategia de Migración](#11-estrategia-de-migración)
12. [Monitoreo y Observabilidad](#12-monitoreo-y-observabilidad)
13. [Plan de Ejecución](#13-plan-de-ejecución)

---

## 1. Introducción y Contexto

### 1.1 Objetivos de la Migración

- Migrar el core a tecnologías más robustas y escalables
- Implementar una arquitectura bien definida y documentada
- Facilitar la escalabilidad sin requerir cambios estructurales mayores
- Mejorar la mantenibilidad y separación de responsabilidades
- **Garantizar alta disponibilidad y cero pérdida de datos durante deploys**

### 1.2 Contexto del Sistema

El sistema HSE maneja:
- **100+ eventos por segundo** de controladores IoT
- Datos de series temporales para análisis y gráficas
- Múltiples módulos de negocio (IoT, Booking, Smart Access, etc.)
- Comunicación en tiempo real con dispositivos
- Análisis de eficiencia energética y automatización

### 1.3 Decisiones Arquitectónicas Principales

#### 1.3.1 Separación de Servicios

**Decisión**: Dividir en dos servicios principales:
1. **Monolito Modular** (NestJS): Lógica de negocio
2. **Devices Service** (Go): Manejo de conexiones y eventos IoT

**Justificación**:
- Evita pérdida de datos durante deploys del monolito
- Permite actualizar lógica de negocio sin interrumpir recolección de datos
- Si un servicio cae, el otro mantiene operación
- Escalabilidad independiente de cada servicio

#### 1.3.2 Golang para Devices Service

**Justificación**:
- **Alta concurrencia**: Manejo eficiente de 100+ eventos/seg
- **Bajo consumo de memoria**: Para conexiones TCP persistentes
- **Performance**: Goroutines nativas para procesamiento paralelo
- **Binarios compilados**: Deploy simple y rápido

#### 1.3.3 CQRS en Devices Service

**Justificación**:
- **Write side**: Optimizado para alta frecuencia de escritura (100+ eventos/seg)
- **Read side**: Optimizado para consultas y generación de gráficas
- **Escalabilidad independiente**: Escalar writes y reads por separado
- **Consistencia eventual**: Aceptable para análisis y reportes

#### 1.3.4 gRPC Bidireccional

**Justificación**:
- **Performance**: 7-10x más rápido que REST para alta frecuencia
- **Streaming bidireccional**: Notificaciones en tiempo real de cambios
- **Tipado fuerte**: Contratos bien definidos (Protocol Buffers)
- **Comunicación eficiente**: Binario en vez de JSON

---

## 2. Arquitectura General

### 2.1 Vista de Alto Nivel

```
┌─────────────────────────────────────────────────────────────┐
│                         CLIENTES                             │
│  (Web App, Mobile App, APIs de Terceros)                    │
└────────────────────┬────────────────────────────────────────┘
                     │
                     │ HTTP/HTTPS (GraphQL/REST)
                     │
┌────────────────────▼────────────────────────────────────────┐
│                  MONOLITO MODULAR (NestJS)                   │
│                                                              │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │   Auth   │  │   IoT    │  │ Booking  │  │  Smart   │   │
│  │  Module  │  │   Core   │  │  Module  │  │  Access  │   │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘   │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │         Shared Infrastructure Layer                   │  │
│  │  (Database, Cache, gRPC Client, Monitoring)          │  │
│  └──────────────────────────────────────────────────────┘  │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       │ gRPC (Bidireccional)
                       │
┌──────────────────────▼──────────────────────────────────────┐
│              DEVICES SERVICE (Golang)                        │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              WRITE SIDE (Commands)                   │   │
│  │  TCP Server → WebSocket Server → Event Queue        │   │
│  └─────────────────────┬───────────────────────────────┘   │
│                        │                                     │
│                        ▼                                     │
│  ┌─────────────────────────────────────────────────────┐   │
│  │         Redis Queue + Batch Processor               │   │
│  └─────────────────────┬───────────────────────────────┘   │
│                        │                                     │
│                        ▼                                     │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              READ SIDE (Queries)                     │   │
│  │  TimescaleDB + Redis Cache + Continuous Aggregates  │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
                       │
                       │ TCP/WebSocket
                       │
┌──────────────────────▼──────────────────────────────────────┐
│              DISPOSITIVOS IoT                                │
│  (Controladores, Sensores, Actuadores)                      │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 Servicio Monolito Modular

**Arquitectura**: Hexagonal (Puertos y Adaptadores)

**Características**:
- Separación clara de responsabilidades por capas
- Módulos independientes con bajo acoplamiento
- Facilidad para agregar nuevos módulos
- Posibilidad futura de extraer a microservicios

**Patrones de Diseño**:
- **Adapter Pattern**: Para adaptar interfaces externas
- **Repository Pattern**: Para abstracción de persistencia
- **Use Case Pattern**: Para lógica de aplicación
- **Singleton Pattern**: Para servicios compartidos específicos

### 2.3 Servicio de Devices/Connection

**Arquitectura**: CQRS (Command Query Responsibility Segregation)

**Características**:
- Write side optimizado para alta concurrencia
- Read side optimizado para consultas y análisis
- Batch processing para eficiencia
- Event sourcing ligero para auditoría

**Patrones de Diseño**:
- **Adapter Pattern**: Para diferentes protocolos (TCP, WebSocket, Modbus)
- **Abstract Factory**: Para crear handlers de conexión
- **Command Pattern**: Para comandos a dispositivos
- **Observer Pattern**: Para notificaciones en tiempo real

---

## 3. Stack Tecnológico

### 3.1 Monolito Modular (NestJS)

| Categoría | Tecnología | Propósito | Versión |
|-----------|-----------|-----------|---------|
| **Framework** | NestJS | Framework principal | ^10.0.0 |
| **Lenguaje** | TypeScript | Lenguaje de programación | ^5.0.0 |
| **Servidor** | Fastify | Servidor HTTP de alto rendimiento | ^4.0.0 |
| **Base de Datos** | PostgreSQL | Base de datos relacional | 15+ |
| **ORM** | Prisma | Mapeo objeto-relacional | ^5.0.0 |
| **Caché** | Redis | Base de datos de caché | 7.0+ |
| **Autenticación** | JWT | Tokens de autenticación | - |
| **Hashing** | Argon2 | Hashing de contraseñas | - |
| **API** | Apollo Server | Servidor GraphQL | ^4.0.0 |
| **API** | @nestjs/graphql | Integración GraphQL | ^12.0.0 |
| **Comunicación** | @grpc/grpc-js | Cliente gRPC | ^1.9.0 |
| **Seguridad** | helmet | Headers de seguridad | ^7.0.0 |
| **Seguridad** | @nestjs/throttler | Rate limiting | ^5.0.0 |
| **Validación** | class-validator | Validación de DTOs | ^0.14.0 |
| **Monitoreo** | newrelic | APM y observabilidad | ^11.0.0 |
| **Testing** | Jest | Testing unitario e integración | ^29.0.0 |
| **Testing** | @nestjs/testing | Utilidades de testing | ^10.0.0 |

### 3.2 Devices Service (Golang)

| Categoría | Tecnología | Propósito | Versión |
|-----------|-----------|-----------|---------|
| **Lenguaje** | Go | Lenguaje de programación | 1.21+ |
| **Base de Datos** | PostgreSQL + TimescaleDB | Series temporales | 15+ / 2.13+ |
| **ORM** | GORM | ORM para Go | v1.25.0+ |
| **Framework Web** | Echo | HTTP y WebSocket | v4.11.0+ |
| **Caché/Queue** | go-redis | Cliente Redis | v9.0.0+ |
| **gRPC** | google.golang.org/grpc | Servidor gRPC | v1.58.0+ |
| **Protocol Buffers** | google.golang.org/protobuf | Serialización | v1.31.0+ |
| **Circuit Breaker** | github.com/sony/gobreaker | Tolerancia a fallos | v2.0.0+ |
| **Logging** | zerolog | Logging estructurado | v1.31.0+ |
| **Validación** | go-playground/validator | Validación de structs | v10.15.0+ |
| **Testing** | testify | Assertions y mocks | v1.8.0+ |
| **Monitoreo** | newrelic/go-agent | APM (opcional) | v3.25.0+ |

### 3.3 Infraestructura

| Categoría | Tecnología | Propósito |
|-----------|-----------|-----------|
| **Container** | Docker | Contenedorización |
| **Orquestación** | ECS Fargate (AWS) | Gestión de containers |
| **Base de Datos** | RDS PostgreSQL + TimescaleDB | Base de datos gestionada |
| **Caché/Queue** | ElastiCache Redis | Redis gestionado |
| **Load Balancer** | ALB (Application Load Balancer) | Balanceo de carga |
| **CDN** | CloudFront | Distribución de contenido |
| **Secrets** | AWS Secrets Manager | Gestión de credenciales |
| **Logs** | CloudWatch + New Relic | Logs centralizados |
| **CI/CD** | GitHub Actions + AWS CodeDeploy | Pipeline de despliegue |

---

## 4. Módulos del Sistema

### 4.1 Módulo de Autenticación (Auth)

**Responsabilidad**: Autenticar y autorizar usuarios y terceros.

**Funcionalidades**:
- Autenticación JWT para usuarios finales
- Autenticación API Keys para terceros (modo desarrollador)
- Gestión de sesiones con cookies seguras (HttpOnly, SameSite)
- Refresh tokens
- Gestión de permisos granulares

**Tipos de Usuarios**:
1. **Terceros (API)**:
   - Autenticación mediante API Keys
   - Permisos específicos por API Key
   - Rate limiting por cliente
   - Auditoría de uso

2. **Clientes**:
   - Autenticación JWT + cookies seguras
   - Roles y permisos dinámicos
   - Multi-factor authentication (futuro)

### 4.2 Módulo IoT/Core

**Responsabilidad**: Gestión de eficiencia energética, automatización y funcionalidades core.

**Funcionalidades**:
- Gestión de usuarios y permisos
- Administración de organizaciones y establecimientos
- Control de dispositivos IoT
- Monitoreo energético en tiempo real
- Análisis y reportes de consumo
- Automatizaciones y escenarios
- Alertas y notificaciones

**Comunicación con Devices Service**:
- Envío de comandos via gRPC
- Recepción de eventos en tiempo real via gRPC streaming
- Consulta de estados de dispositivos

### 4.3 Módulo de Reservas (Booking)

**Responsabilidad**: Gestionar reservaciones de espacios con dispositivos IoT.

**Funcionalidades**:
- Creación de reservas con validación de disponibilidad
- Gestión de huéspedes y acompañantes
- Generación automática de códigos de acceso
- Notificaciones automáticas con instrucciones
- Consulta de estados de dispositivos en áreas reservadas
- Historial de reservas

**Estados de Reservas**:
- `pending`: Creada, pendiente de confirmación
- `confirmed`: Confirmada por el huésped
- `in-progress`: Activa (dentro del rango de fechas)
- `completed`: Finalizada
- `cancelled`: Cancelada

### 4.4 Módulo de Smart Access

**Responsabilidad**: Control de acceso vehicular y peatonal inteligente.

**Funcionalidades** (a detallar):
- Control de acceso vehicular autorizado
- Gestión de permisos por propietarios
- Gestión de permisos por administración
- Registro de entradas/salidas
- Integración con reconocimiento de placas

### 4.5 Módulo de Universidades/Escuelas

**Responsabilidad**: Gestión de accesos e interacciones para entidades educativas.

**Funcionalidades** (a detallar):
- Control de acceso de estudiantes
- Gestión de horarios
- Reportes de asistencia

---

## 5. Modelo de Datos

### 5.1 Módulo IoT/Core - Eficiencia Energética

#### 5.1.1 Entidades Principales

**Users**
```typescript
{
  id: UUID
  email: string
  passwordHash: string
  firstName: string
  lastName: string
  roleId: UUID
  partnerCompanyId?: UUID
  organizationId?: UUID
  isActive: boolean
  emailVerified: boolean
  createdAt: DateTime
  updatedAt: DateTime
}
```

**Roles**
```typescript
{
  id: UUID
  name: string
  description: string
  type: 'system' | 'custom'  // system = root/partner/admin/user
  hierarchy: number           // 1=root, 2=partner, 3=admin, 4=user
  organizationId?: UUID       // null para roles de sistema
  establishmentId?: UUID      // para roles específicos de establecimiento
  createdBy: UUID
  createdAt: DateTime
  updatedAt: DateTime
}
```

**Actions** (Permisos atómicos)
```typescript
{
  id: UUID
  name: string                // 'create_user', 'delete_establishment', etc.
  resource: string            // 'user', 'establishment', 'device', etc.
  action: string              // 'create', 'read', 'update', 'delete', etc.
  description: string
  category: string            // 'user_management', 'device_control', etc.
}
```

**Permissions** (Muchos a Muchos)
```typescript
{
  roleId: UUID
  actionId: UUID
  grantedAt: DateTime
  grantedBy: UUID
}
```

**PartnerCompanies**
```typescript
{
  id: UUID
  name: string
  contactEmail: string
  contactPhone: string
  address: string
  isActive: boolean
  createdAt: DateTime
  updatedAt: DateTime
}
```

**Organizations**
```typescript
{
  id: UUID
  name: string
  partnerCompanyId?: UUID     // null si es administrado por Horus
  industry: string
  contactEmail: string
  contactPhone: string
  address: string
  isActive: boolean
  createdAt: DateTime
  updatedAt: DateTime
}
```

**Establishments**
```typescript
{
  id: UUID
  organizationId: UUID
  name: string
  address: string
  location: Point             // PostGIS
  timezone: string
  isActive: boolean
  metadata: JSONB             // configuraciones específicas
  createdAt: DateTime
  updatedAt: DateTime
}
```

**UserEstablishments** (Muchos a Muchos)
```typescript
{
  userId: UUID
  establishmentId: UUID
  roleId?: UUID               // rol específico para este establecimiento
  assignedAt: DateTime
  assignedBy: UUID
}
```

**Areas**
```typescript
{
  id: UUID
  establishmentId: UUID
  name: string
  type: string                // 'office', 'production', 'storage', etc.
  parentAreaId?: UUID         // para jerarquías
  floor?: number
  metadata: JSONB
  createdAt: DateTime
  updatedAt: DateTime
}
```

**Devices**
```typescript
{
  id: UUID
  areaId: UUID
  deviceType: string          // 'sensor', 'actuator', 'doorlock', etc.
  manufacturer: string
  model: string
  serialNumber: string
  macAddress: string
  ipAddress?: string
  protocol: string            // 'tcp', 'modbus', 'mqtt', etc.
  status: string              // 'online', 'offline', 'error', etc.
  metadata: JSONB
  createdAt: DateTime
  updatedAt: DateTime
}
```

#### 5.1.2 Diagrama de Relaciones

```
PartnerCompanies
    │
    ├──> Organizations
    │       │
    │       ├──> Establishments
    │       │       │
    │       │       ├──> UserEstablishments ←──┐
    │       │       │                          │
    │       │       └──> Areas                 │
    │       │               │                  │
    │       │               └──> Devices       │
    │       │                                  │
    │       └──> Users ─────────────────────────┤
    │                                           │
    └──> Users (partners) ──────────────────────┘
                │
                ├──> Roles ──> Permissions ──> Actions
                │       │
                │       └──> (custom roles)
                │
                └──> (email, password, etc.)
```

### 5.2 Módulo de Booking

#### 5.2.1 Entidades Principales

**Guests**
```typescript
{
  id: UUID
  firstName: string
  lastName: string
  email: string
  phone: string
  documentType: string
  documentNumber: string
  country: string
  metadata: JSONB
  createdAt: DateTime
  updatedAt: DateTime
}
```

**GuestCompanions** (Relación recíproca)
```typescript
{
  guestId: UUID
  companionId: UUID
  relationship: string        // 'spouse', 'child', 'friend', etc.
  addedAt: DateTime
}
```

**Bookings**
```typescript
{
  id: UUID
  guestId: UUID
  areaId: UUID                // habitación/espacio
  checkInDate: DateTime
  checkOutDate: DateTime
  status: 'pending' | 'confirmed' | 'in-progress' | 'completed' | 'cancelled'
  confirmationCode: string
  specialRequests?: string
  totalGuests: number
  metadata: JSONB
  createdBy: UUID
  createdAt: DateTime
  updatedAt: DateTime
  cancelledAt?: DateTime
  cancelledBy?: UUID
  cancellationReason?: string
}
```

**DoorlockCodes**
```typescript
{
  id: UUID
  bookingId: UUID
  deviceId: UUID              // doorlock específico
  code: string
  validFrom: DateTime
  validUntil: DateTime
  isActive: boolean
  usageCount: number
  maxUsages?: number
  createdAt: DateTime
}
```

**Notifications**
```typescript
{
  id: UUID
  bookingId: UUID
  guestId: UUID
  type: string                // 'booking_confirmed', 'check_in_instructions', etc.
  channel: string             // 'email', 'sms', 'push', etc.
  recipient: string
  subject?: string
  message: string
  status: 'pending' | 'sent' | 'failed'
  sentAt?: DateTime
  metadata: JSONB
  createdAt: DateTime
}
```

#### 5.2.2 Diagrama de Relaciones

```
Guests ──┬──> GuestCompanions (self-referential)
         │
         └──> Bookings ──┬──> Areas ──> Devices (Doorlocks)
                         │                  │
                         ├──> DoorlockCodes ┘
                         │
                         └──> Notifications
```

### 5.3 Devices Service - Series Temporales

**DeviceEvents** (TimescaleDB Hypertable)
```sql
CREATE TABLE device_events (
  time          TIMESTAMPTZ NOT NULL,
  device_id     UUID NOT NULL,
  event_type    VARCHAR(50) NOT NULL,
  value         NUMERIC,
  unit          VARCHAR(20),
  metadata      JSONB,
  source        VARCHAR(50),
  INDEX idx_device_time (device_id, time DESC)
);

-- Convertir a hypertable (TimescaleDB)
SELECT create_hypertable('device_events', 'time');

-- Compresión automática (datos > 7 días)
ALTER TABLE device_events SET (
  timescaledb.compress,
  timescaledb.compress_segmentby = 'device_id'
);

SELECT add_compression_policy('device_events', INTERVAL '7 days');

-- Retención automática (eliminar datos > 2 años)
SELECT add_retention_policy('device_events', INTERVAL '2 years');
```

**DeviceStates** (Estados actuales en Redis)
```typescript
// Redis Key: device:state:{device_id}
{
  deviceId: string
  status: 'online' | 'offline' | 'error'
  lastEvent: {
    type: string
    value: number
    unit: string
    timestamp: number
  }
  connectionState: {
    connected: boolean
    lastSeen: number
    protocol: string
    remoteAddress: string
  }
  updatedAt: number
}
```

---

## 6. Reglas de Negocio y Flujos

### 6.1 Sistema de Roles y Permisos

#### 6.1.1 Jerarquía de Roles

```
root (hierarchy: 1)
  └──> partner (hierarchy: 2)
        └──> admin (hierarchy: 3)
              ├──> user (hierarchy: 4)
              └──> custom roles (hierarchy: 5+)
```

#### 6.1.2 Reglas de Creación y Asignación

**Creación de Roles**:
1. `root`, `partner` y `admin` pueden crear roles
2. Solo pueden asignar permisos que tengan y que NO sean exclusivos de su nivel
3. Los roles creados tienen `hierarchy` mayor que el creador
4. `user` NO puede crear roles

**Creación de Usuarios**:
1. `root`, `partner` y `admin` pueden crear usuarios
2. Solo pueden asignar roles iguales o inferiores al suyo
3. Un `admin` NO puede crear `root` o `partner`

**Permisos Exclusivos** (a definir):
```typescript
const EXCLUSIVE_PERMISSIONS = {
  root: [
    'create_partner_company',
    'delete_partner_company',
    'manage_system_roles',
    'view_all_organizations'
  ],
  partner: [
    'create_organization',
    'delete_organization',
    'view_partner_analytics'
  ],
  admin: [
    'create_establishment',
    'delete_establishment',
    'manage_organization_users'
  ]
}
```

#### 6.1.3 Visibilidad por Rol

**Root**:
- Acceso sin restricciones
- Ve todas las entidades del sistema

**Partner**:
- Ve su `PartnerCompany`
- Ve todas las `Organizations` de su partner
- Ve todos los `Establishments` de sus organizaciones
- **NO ve**:
  - Organizations de otros partners
  - Organizations sin partner (Horus directo)

**Admin**:
- Ve su `Organization`
- Ve solo `Establishments` asignados a él en `UserEstablishments`
- El "admin en jefe" tiene todos los establishments asignados

**User**:
- Acceso según permisos específicos asignados
- Visibilidad limitada a recursos autorizados

### 6.2 Flujo de Onboarding

```
1. Root crea entidad base
   └──> PartnerCompany (para partner) o Organization (para admin)

2. Root crea usuario
   └──> Asigna rol correspondiente (partner/admin)
   └──> Envía invitación por email

3. Usuario invitado
   └──> Recibe email con link único
   └──> Acepta o rechaza invitación
   └──> Si acepta: configura credenciales (contraseña, 2FA)
   └──> Inicia sesión en la plataforma

4. Primer login
   └──> Wizard de configuración inicial
   └──> Configura establecimientos (si es admin)
   └──> Invita a su equipo (si aplica)
```

**Regla de Consistencia**: Una invitación solo puede existir si la entidad asociada (`PartnerCompany` o `Organization`) ya fue creada por `root`.

### 6.3 Módulo de Booking - Reglas de Reservas

#### 6.3.1 Estados y Transiciones

```
pending ──> confirmed ──> in-progress ──> completed
   │                                          ↑
   │                                          │
   └──────────> cancelled ────────────────────┘
                   ↑
                   │
           (solo si pending o confirmed)
```

**Transiciones Válidas**:
- `pending` → `confirmed`: Usuario confirma reserva
- `confirmed` → `in-progress`: Check-in automático al llegar la fecha
- `in-progress` → `completed`: Check-out automático al pasar la fecha
- `pending` → `cancelled`: ✅ Permitido
- `confirmed` → `cancelled`: ✅ Permitido
- `in-progress` → `cancelled`: ❌ NO PERMITIDO
- `completed` → `cancelled`: ❌ NO PERMITIDO

#### 6.3.2 Reglas de Solapamiento

**Regla Principal**: No se pueden crear reservas que se solapen en fechas para la misma área.

**Excepciones**:
```typescript
// Se permite crear reserva si la existente está:
const ALLOWED_OVERLAP_STATES = ['cancelled'];

// O si fue eliminada físicamente
const canCreateBooking = (
  newBooking: BookingDTO,
  existingBookings: Booking[]
): boolean => {
  const overlapping = existingBookings.filter(b => 
    // Verifica solapamiento de fechas
    (newBooking.checkIn < b.checkOut && newBooking.checkOut > b.checkIn) &&
    // Solo cuenta reservas activas
    !ALLOWED_OVERLAP_STATES.includes(b.status)
  );
  
  return overlapping.length === 0;
}
```

**Margen de Tiempo**:
```typescript
const CHECK_IN_MARGIN = 60; // minutos después del checkout anterior
const CHECK_OUT_MARGIN = 30; // minutos antes del checkin siguiente

// Ejemplo: Si checkout es 11:00, próximo checkin puede ser 12:00
```

#### 6.3.3 Operaciones Permitidas

**Cancelar**:
- ✅ Estados: `pending`, `confirmed`
- ❌ Estados: `in-progress`, `completed`
- Acción: Cambia estado a `cancelled` (registro se mantiene)

**Eliminar**:
- ✅ Estados: `pending`, `confirmed`
- ❌ Estados: `in-progress`, `completed`
- Acción: Elimina físicamente el registro

**Modificar**:
- ✅ Estados: `pending`, `confirmed`
- ❌ Estados: `in-progress`, `completed`
- Validación: Debe cumplir reglas de solapamiento

#### 6.3.4 Generación de Códigos de Acceso

```typescript
const generateDoorlockCode = async (booking: Booking): Promise<string> => {
  // 1. Obtener doorlock del área reservada
  const doorlock = await getAreaDoorlock(booking.areaId);
  
  // 2. Generar código único (6 dígitos)
  const code = generateSecureCode(6);
  
  // 3. Verificar unicidad
  const exists = await checkCodeExists(doorlock.id, code);
  if (exists) return generateDoorlockCode(booking);
  
  // 4. Guardar en DB con validez temporal
  await saveDoorlockCode({
    bookingId: booking.id,
    deviceId: doorlock.id,
    code: code,
    validFrom: booking.checkInDate,
    validUntil: booking.checkOutDate,
    isActive: true
  });
  
  // 5. Enviar comando al dispositivo via gRPC
  await devicesService.programDoorlockCode(doorlock.id, code, {
    validFrom: booking.checkInDate,
    validUntil: booking.checkOutDate
  });
  
  return code;
}
```

---

## 7. Arquitectura CQRS en Devices Service

### 7.1 Vista Completa

```
┌─────────────────────────────────────────────────────────────┐
│                  WRITE SIDE (Commands)                       │
│                                                              │
│  ┌────────────┐   ┌────────────┐   ┌────────────┐         │
│  │TCP Server  │   │ WS Server  │   │gRPC Server │         │
│  │(Devices)   │   │(Real-time) │   │(Monolith)  │         │
│  └──────┬─────┘   └──────┬─────┘   └──────┬─────┘         │
│         │                │                │                 │
│         └────────────────┼────────────────┘                 │
│                          ▼                                   │
│                 ┌─────────────────┐                         │
│                 │ Command Handler │                         │
│                 └────────┬────────┘                         │
│                          │                                   │
│                 ┌────────▼────────┐                         │
│                 │ Validation      │                         │
│                 └────────┬────────┘                         │
│                          │                                   │
│            ┌─────────────┼──────────────┐                  │
│            ▼             ▼              ▼                   │
│      ┌─────────┐   ┌─────────┐   ┌──────────┐            │
│      │  Redis  │   │  Redis  │   │   gRPC   │            │
│      │  Queue  │   │  Cache  │   │(Notify   │            │
│      │         │   │(States) │   │Monolith) │            │
│      └────┬────┘   └─────────┘   └──────────┘            │
│           │                                                 │
│           │ Async Processing                               │
│           ▼                                                 │
│   ┌───────────────┐                                        │
│   │Event Processor│ ← Procesa en batches                  │
│   │ (Background)  │   cada 1-5 seg                        │
│   └───────┬───────┘                                        │
│           │                                                 │
│           ▼                                                 │
│   ┌───────────────┐                                        │
│   │  TimescaleDB  │ ← Batch inserts                       │
│   │(Bulk writes)  │   (100-1000 eventos)                  │
│   └───────────────┘                                        │
│                                                             │
│  Latencia: < 10ms (solo Redis)                            │
│  Throughput: 1000+ eventos/seg                            │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                   READ SIDE (Queries)                        │
│                                                              │
│  ┌────────────┐                                             │
│  │   gRPC     │                                             │
│  │ (Monolith  │                                             │
│  │  queries)  │                                             │
│  └──────┬─────┘                                             │
│         │                                                    │
│         ▼                                                    │
│  ┌─────────────┐                                           │
│  │Query Handler│                                           │
│  └──────┬──────┘                                           │
│         │                                                    │
│    ┌────┴──────┐                                           │
│    ▼           ▼                                            │
│ ┌─────────┐ ┌──────────────────┐                          │
│ │  Redis  │ │   TimescaleDB    │                          │
│ │ (Cache) │ │                  │                          │
│ │         │ │ - Raw events     │                          │
│ │Hot data │ │ - Continuous     │                          │
│ │Estados  │ │   aggregates     │                          │
│ │actuales │ │   (pre-calc)     │                          │
│ └─────────┘ └──────────────────┘                          │
│                                                             │
│  Cache Hit Rate: ~85%                                      │
│  Query Latency: < 50ms (cached), < 200ms (DB)             │
└─────────────────────────────────────────────────────────────┘

CONSISTENCIA EVENTUAL:
- Writes: Confirmación inmediata (< 10ms)
- Reads: Pueden tener delay de 1-5 segundos
- Estados críticos: Siempre en Redis (tiempo real)
```

### 7.2 Flujo de Datos Detallado

#### 7.2.1 Write Path (Hot Path)

```
1. Evento llega al servidor
   ↓
2. Validación básica (schema, device exists)
   ↓
3. Escritura a Redis Queue (< 1ms)
   ↓
4. ACK inmediato al dispositivo
   ↓
5. [Background] Worker lee de queue en batches
   ↓
6. [Background] Batch insert a TimescaleDB
   ↓
7. [Background] Actualiza cache (estados actuales)
   ↓
8. [Background] Notifica a monolito si es evento relevante
```

**Ventajas**:
- Latencia ultra baja (< 10ms)
- No bloquea por escrituras a disco
- Tolerante a picos de tráfico
- Sin pérdida de datos (Redis persistente)

#### 7.2.2 Read Path

```
1. Query llega via gRPC
   ↓
2. Intenta leer de Redis (estados actuales)
   ├─> HIT: Retorna inmediatamente (< 5ms)
   └─> MISS:
       ↓
       3. Query a TimescaleDB
          ├─> Datos recientes: Tabla raw
          └─> Análisis/gráficas: Continuous aggregates
       ↓
       4. Guarda en cache para próximas queries
       ↓
       5. Retorna resultado (< 200ms)
```

### 7.3 TimescaleDB - Continuous Aggregates

Los continuous aggregates son vistas materializadas que se actualizan automáticamente:

```sql
-- Agregación por hora
CREATE MATERIALIZED VIEW device_events_hourly
WITH (timescaledb.continuous) AS
SELECT 
  time_bucket('1 hour', time) AS hour,
  device_id,
  event_type,
  AVG(value) AS avg_value,
  MAX(value) AS max_value,
  MIN(value) AS min_value,
  COUNT(*) AS event_count,
  STDDEV(value) AS stddev_value
FROM device_events
GROUP BY hour, device_id, event_type;

-- Agregación por día
CREATE MATERIALIZED VIEW device_events_daily
WITH (timescaledb.continuous) AS
SELECT 
  time_bucket('1 day', time) AS day,
  device_id,
  event_type,
  AVG(value) AS avg_value,
  MAX(value) AS max_value,
  MIN(value) AS min_value,
  SUM(value) AS total_value,
  COUNT(*) AS event_count
FROM device_events
GROUP BY day, device_id, event_type;

-- Política de refresh (cada 10 minutos)
SELECT add_continuous_aggregate_policy('device_events_hourly',
  start_offset => INTERVAL '3 hours',
  end_offset => INTERVAL '1 hour',
  schedule_interval => INTERVAL '10 minutes'
);

SELECT add_continuous_aggregate_policy('device_events_daily',
  start_offset => INTERVAL '3 days',
  end_offset => INTERVAL '1 day',
  schedule_interval => INTERVAL '1 hour'
);
```

**Beneficios**:
- Queries de gráficas son 100-1000x más rápidas
- No hay carga adicional en queries
- Se actualizan automáticamente
- Ahorro de almacenamiento (datos antiguos comprimidos)

---

## 8. Patrones Críticos de Implementación

### 8.1 Batch Processing con Redis Queue

#### 8.1.1 Implementación en Go

```go
// internal/application/services/event_processor.go
package services

import (
    "context"
    "time"
    "github.com/go-redis/redis/v9"
)

type EventProcessor struct {
    redisClient   *redis.Client
    eventWriter   *timescale.EventWriter
    cacheManager  *CacheManager
    grpcNotifier  *grpc.MonolithClient
    batchSize     int
    flushInterval time.Duration
}

func NewEventProcessor(deps *Dependencies) *EventProcessor {
    return &EventProcessor{
        redisClient:   deps.Redis,
        eventWriter:   deps.EventWriter,
        cacheManager:  deps.CacheManager,
        grpcNotifier:  deps.GRPCNotifier,
        batchSize:     1000,           // eventos por batch
        flushInterval: 5 * time.Second, // flush cada 5 segundos
    }
}

func (ep *EventProcessor) Start(ctx context.Context) error {
    ticker := time.NewTicker(ep.flushInterval)
    defer ticker.Stop()
    
    batch := make([]*domain.DeviceEvent, 0, ep.batchSize)
    
    for {
        select {
        case <-ctx.Done():
            // Flush final antes de cerrar
            if len(batch) > 0 {
                ep.flushBatch(ctx, batch)
            }
            return ctx.Err()
            
        case <-ticker.C:
            // Flush periódico
            if len(batch) > 0 {
                if err := ep.flushBatch(ctx, batch); err != nil {
                    // Log error pero continuar
                    ep.handleFlushError(batch, err)
                }
                batch = batch[:0] // clear batch
            }
            
        default:
            // Intentar leer evento de la queue
            result, err := ep.redisClient.BRPop(
                ctx,
                100*time.Millisecond,
                "events:queue",
            ).Result()
            
            if err == redis.Nil {
                // No hay eventos, continuar
                continue
            }
            
            if err != nil {
                // Error leyendo, log y continuar
                log.Error().Err(err).Msg("Error reading from queue")
                continue
            }
            
            // Parsear evento
            event, err := ep.parseEvent(result[1])
            if err != nil {
                log.Error().Err(err).Msg("Error parsing event")
                continue
            }
            
            batch = append(batch, event)
            
            // Flush si batch está lleno
            if len(batch) >= ep.batchSize {
                if err := ep.flushBatch(ctx, batch); err != nil {
                    ep.handleFlushError(batch, err)
                }
                batch = batch[:0]
            }
        }
    }
}

func (ep *EventProcessor) flushBatch(
    ctx context.Context,
    events []*domain.DeviceEvent,
) error {
    // 1. Batch insert a TimescaleDB
    if err := ep.eventWriter.InsertBatch(ctx, events); err != nil {
        return fmt.Errorf("failed to insert batch: %w", err)
    }
    
    // 2. Actualizar cache de estados actuales
    if err := ep.cacheManager.UpdateDeviceStates(ctx, events); err != nil {
        // Log pero no fallar por esto
        log.Error().Err(err).Msg("Failed to update cache")
    }
    
    // 3. Notificar eventos relevantes al monolito
    relevantEvents := ep.filterRelevantEvents(events)
    if len(relevantEvents) > 0 {
        if err := ep.grpcNotifier.NotifyEvents(ctx, relevantEvents); err != nil {
            // Log pero no fallar
            log.Error().Err(err).Msg("Failed to notify monolith")
        }
    }
    
    log.Info().
        Int("count", len(events)).
        Msg("Batch processed successfully")
    
    return nil
}

func (ep *EventProcessor) handleFlushError(
    events []*domain.DeviceEvent,
    err error,
) {
    log.Error().
        Err(err).
        Int("count", len(events)).
        Msg("Failed to flush batch, moving to DLQ")
    
    // Mover a Dead Letter Queue
    for _, event := range events {
        eventJSON, _ := json.Marshal(event)
        ep.redisClient.LPush(
            context.Background(),
            "events:dlq",
            eventJSON,
        )
    }
}

func (ep *EventProcessor) filterRelevantEvents(
    events []*domain.DeviceEvent,
) []*domain.DeviceEvent {
    relevant := make([]*domain.DeviceEvent, 0)
    
    for _, event := range events {
        // Solo notificar eventos importantes
        if event.Type == "alarm" ||
           event.Type == "state_change" ||
           event.Type == "error" {
            relevant = append(relevant, event)
        }
    }
    
    return relevant
}
```

#### 8.1.2 Batch Insert en TimescaleDB

```go
// internal/infrastructure/persistence/postgres/timescale/event_writer.go
package timescale

func (ew *EventWriter) InsertBatch(
    ctx context.Context,
    events []*domain.DeviceEvent,
) error {
    if len(events) == 0 {
        return nil
    }
    
    // Construir query de bulk insert
    query := `
        INSERT INTO device_events (
            time, device_id, event_type, value, unit, metadata, source
        ) VALUES `
    
    values := make([]interface{}, 0, len(events)*7)
    placeholders := make([]string, 0, len(events))
    
    for i, event := range events {
        offset := i * 7
        placeholders = append(placeholders, fmt.Sprintf(
            "($%d, $%d, $%d, $%d, $%d, $%d, $%d)",
            offset+1, offset+2, offset+3, offset+4,
            offset+5, offset+6, offset+7,
        ))
        
        metadataJSON, _ := json.Marshal(event.Metadata)
        
        values = append(values,
            event.Time,
            event.DeviceID,
            event.Type,
            event.Value,
            event.Unit,
            metadataJSON,
            event.Source,
        )
    }
    
    query += strings.Join(placeholders, ",")
    
    // Ejecutar con timeout
    ctx, cancel := context.WithTimeout(ctx, 10*time.Second)
    defer cancel()
    
    start := time.Now()
    _, err := ew.db.ExecContext(ctx, query, values...)
    duration := time.Since(start)
    
    if err != nil {
        log.Error().
            Err(err).
            Int("count", len(events)).
            Dur("duration", duration).
            Msg("Batch insert failed")
        return err
    }
    
    log.Info().
        Int("count", len(events)).
        Dur("duration", duration).
        Float64("events_per_sec", float64(len(events))/duration.Seconds()).
        Msg("Batch insert successful")
    
    return nil
}
```

### 8.2 Circuit Breaker para gRPC

```go
// internal/infrastructure/grpc/monolith_client.go
package grpc

import (
    "github.com/sony/gobreaker"
)

type MonolithClient struct {
    client  pb.DeviceStreamClient
    breaker *gobreaker.CircuitBreaker
}

func NewMonolithClient(conn *grpc.ClientConn) *MonolithClient {
    settings := gobreaker.Settings{
        Name:        "monolith-grpc",
        MaxRequests: 3,  // requests permitidos en half-open
        Interval:    10 * time.Second,
        Timeout:     30 * time.Second,
        ReadyToTrip: func(counts gobreaker.Counts) bool {
            failureRatio := float64(counts.TotalFailures) / float64(counts.Requests)
            return counts.Requests >= 3 && failureRatio >= 0.6
        },
        OnStateChange: func(name string, from, to gobreaker.State) {
            log.Warn().
                Str("circuit", name).
                Str("from", from.String()).
                Str("to", to.String()).
                Msg("Circuit breaker state changed")
        },
    }
    
    return &MonolithClient{
        client:  pb.NewDeviceStreamClient(conn),
        breaker: gobreaker.NewCircuitBreaker(settings),
    }
}

func (mc *MonolithClient) NotifyEvents(
    ctx context.Context,
    events []*domain.DeviceEvent,
) error {
    _, err := mc.breaker.Execute(func() (interface{}, error) {
        return nil, mc.sendEventsToMonolith(ctx, events)
    })
    
    if err == gobreaker.ErrOpenState {
        // Circuit abierto, guardar en DLQ para retry posterior
        log.Warn().
            Int("events", len(events)).
            Msg("Circuit breaker open, saving to DLQ")
        
        return mc.saveToDeadLetterQueue(events)
    }
    
    return err
}

func (mc *MonolithClient) sendEventsToMonolith(
    ctx context.Context,
    events []*domain.DeviceEvent,
) error {
    ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
    defer cancel()
    
    pbEvents := make([]*pb.DeviceEvent, len(events))
    for i, event := range events {
        pbEvents[i] = &pb.DeviceEvent{
            DeviceId:  event.DeviceID.String(),
            EventType: event.Type,
            Timestamp: event.Time.Unix(),
            Value:     event.Value,
            Unit:      event.Unit,
        }
    }
    
    _, err := mc.client.NotifyDeviceEvents(ctx, &pb.NotifyEventsRequest{
        Events: pbEvents,
    })
    
    return err
}
```

### 8.3 gRPC Bidireccional para Streaming

#### 8.3.1 Definición Protocol Buffers

```protobuf
// internal/infrastructure/grpc/proto/devices.proto
syntax = "proto3";

package devices;

option go_package = "github.com/hse/devices-service/internal/infrastructure/grpc/pb";

// Servicio principal
service DeviceStream {
  // Monolito se suscribe a cambios de dispositivos
  rpc SubscribeToDeviceChanges(SubscriptionRequest) 
    returns (stream DeviceChangeEvent);
  
  // Monolito envía comandos a dispositivos
  rpc SendDeviceCommand(DeviceCommand) 
    returns (CommandResponse);
  
  // Monolito notifica al devices service (callback)
  rpc NotifyDeviceEvents(NotifyEventsRequest)
    returns (NotifyEventsResponse);
  
  // Queries de datos
  rpc GetDeviceStatus(DeviceStatusRequest)
    returns (DeviceStatusResponse);
  
  rpc GetDeviceEvents(DeviceEventsRequest)
    returns (DeviceEventsResponse);
}

message SubscriptionRequest {
  repeated string device_ids = 1;
  repeated string event_types = 2;
  string client_id = 3;
}

message DeviceChangeEvent {
  string device_id = 1;
  string event_type = 2;
  int64 timestamp = 3;
  double value = 4;
  string unit = 5;
  map<string, string> metadata = 6;
}

message DeviceCommand {
  string device_id = 1;
  string command_type = 2;
  map<string, string> params = 3;
  string request_id = 4;  // para tracking
}

message CommandResponse {
  bool success = 1;
  string message = 2;
  string request_id = 3;
}

message NotifyEventsRequest {
  repeated DeviceEvent events = 1;
}

message NotifyEventsResponse {
  bool success = 1;
  int32 events_received = 2;
}

message DeviceEvent {
  string device_id = 1;
  string event_type = 2;
  int64 timestamp = 3;
  double value = 4;
  string unit = 5;
}

message DeviceStatusRequest {
  string device_id = 1;
}

message DeviceStatusResponse {
  string device_id = 1;
  string status = 2;  // online, offline, error
  int64 last_seen = 3;
  DeviceEvent last_event = 4;
}

message DeviceEventsRequest {
  string device_id = 1;
  int64 start_time = 2;
  int64 end_time = 3;
  repeated string event_types = 4;
  int32 limit = 5;
}

message DeviceEventsResponse {
  repeated DeviceEvent events = 1;
  int32 total_count = 2;
}
```

#### 8.3.2 Implementación del Stream en Go

```go
// internal/infrastructure/grpc/server/devices_server.go
package server

type DeviceServer struct {
    pb.UnimplementedDeviceStreamServer
    connectionManager *services.ConnectionManager
    eventService      *services.EventService
    commandService    *services.CommandService
}

func (s *DeviceServer) SubscribeToDeviceChanges(
    req *pb.SubscriptionRequest,
    stream pb.DeviceStream_SubscribeToDeviceChangesServer,
) error {
    ctx := stream.Context()
    
    // Crear canal para este cliente
    clientChan := make(chan *domain.DeviceEvent, 100)
    clientID := req.ClientId
    if clientID == "" {
        clientID = uuid.New().String()
    }
    
    // Registrar cliente en el connection manager
    s.connectionManager.RegisterSubscriber(
        clientID,
        clientChan,
        req.DeviceIds,
        req.EventTypes,
    )
    defer s.connectionManager.UnregisterSubscriber(clientID)
    
    log.Info().
        Str("client_id", clientID).
        Strs("device_ids", req.DeviceIds).
        Msg("Client subscribed to device changes")
    
    // Stream eventos al cliente
    for {
        select {
        case event := <-clientChan:
            pbEvent := &pb.DeviceChangeEvent{
                DeviceId:  event.DeviceID.String(),
                EventType: event.Type,
                Timestamp: event.Time.Unix(),
                Value:     event.Value,
                Unit:      event.Unit,
                Metadata:  event.Metadata,
            }
            
            if err := stream.Send(pbEvent); err != nil {
                log.Error().
                    Err(err).
                    Str("client_id", clientID).
                    Msg("Error sending event to client")
                return err
            }
            
        case <-ctx.Done():
            log.Info().
                Str("client_id", clientID).
                Msg("Client disconnected")
            return nil
        }
    }
}

func (s *DeviceServer) SendDeviceCommand(
    ctx context.Context,
    req *pb.DeviceCommand,
) (*pb.CommandResponse, error) {
    deviceID, err := uuid.Parse(req.DeviceId)
    if err != nil {
        return &pb.CommandResponse{
            Success:   false,
            Message:   "Invalid device ID",
            RequestId: req.RequestId,
        }, nil
    }
    
    command := &domain.DeviceCommand{
        DeviceID:    deviceID,
        CommandType: req.CommandType,
        Params:      req.Params,
        RequestID:   req.RequestId,
        Timestamp:   time.Now(),
    }
    
    // Enviar comando al dispositivo
    err = s.commandService.SendCommand(ctx, command)
    if err != nil {
        return &pb.CommandResponse{
            Success:   false,
            Message:   err.Error(),
            RequestId: req.RequestId,
        }, nil
    }
    
    return &pb.CommandResponse{
        Success:   true,
        Message:   "Command sent successfully",
        RequestId: req.RequestId,
    }, nil
}
```

#### 8.3.3 Cliente gRPC en NestJS (Monolito)

```typescript
// src/shared/infrastructure/grpc/devices-grpc.client.ts
import { Injectable, OnModuleInit, OnModuleDestroy } from '@nestjs/common';
import * as grpc from '@grpc/grpc-js';
import * as protoLoader from '@grpc/proto-loader';
import { Observable, Subject } from 'rxjs';

@Injectable()
export class DevicesGrpcClient implements OnModuleInit, OnModuleDestroy {
  private client: any;
  private deviceChanges$ = new Subject<DeviceChangeEvent>();
  private stream: any;

  async onModuleInit() {
    // Cargar proto file
    const packageDefinition = protoLoader.loadSync(
      './proto/devices.proto',
      {
        keepCase: true,
        longs: String,
        enums: String,
        defaults: true,
        oneofs: true,
      },
    );

    const protoDescriptor = grpc.loadPackageDefinition(packageDefinition);
    const DeviceStream = (protoDescriptor.devices as any).DeviceStream;

    // Crear cliente
    this.client = new DeviceStream(
      process.env.DEVICES_GRPC_URL,
      grpc.credentials.createInsecure(),
    );

    // Suscribirse a cambios de dispositivos
    await this.subscribeToDeviceChanges();
  }

  async onModuleDestroy() {
    if (this.stream) {
      this.stream.cancel();
    }
    this.deviceChanges$.complete();
  }

  private async subscribeToDeviceChanges() {
    const request = {
      device_ids: [], // vacío = todos los dispositivos
      event_types: ['alarm', 'state_change', 'error'],
      client_id: 'monolith-' + process.env.INSTANCE_ID,
    };

    this.stream = this.client.SubscribeToDeviceChanges(request);

    this.stream.on('data', (event: any) => {
      this.deviceChanges$.next({
        deviceId: event.device_id,
        eventType: event.event_type,
        timestamp: new Date(event.timestamp * 1000),
        value: event.value,
        unit: event.unit,
        metadata: event.metadata,
      });
    });

    this.stream.on('error', (error: any) => {
      console.error('gRPC stream error:', error);
      // Reconectar después de 5 segundos
      setTimeout(() => this.subscribeToDeviceChanges(), 5000);
    });

    this.stream.on('end', () => {
      console.log('gRPC stream ended, reconnecting...');
      setTimeout(() => this.subscribeToDeviceChanges(), 1000);
    });
  }

  // Observable para suscribirse a cambios
  onDeviceChange(): Observable<DeviceChangeEvent> {
    return this.deviceChanges$.asObservable();
  }

  // Enviar comando a dispositivo
  async sendCommand(
    deviceId: string,
    commandType: string,
    params: Record<string, string>,
  ): Promise<CommandResponse> {
    return new Promise((resolve, reject) => {
      this.client.SendDeviceCommand(
        {
          device_id: deviceId,
          command_type: commandType,
          params: params,
          request_id: uuid(),
        },
        (error: any, response: any) => {
          if (error) {
            reject(error);
          } else {
            resolve({
              success: response.success,
              message: response.message,
              requestId: response.request_id,
            });
          }
        },
      );
    });
  }

  // Query de estado de dispositivo
  async getDeviceStatus(deviceId: string): Promise<DeviceStatus> {
    return new Promise((resolve, reject) => {
      this.client.GetDeviceStatus(
        { device_id: deviceId },
        (error: any, response: any) => {
          if (error) {
            reject(error);
          } else {
            resolve({
              deviceId: response.device_id,
              status: response.status,
              lastSeen: new Date(response.last_seen * 1000),
              lastEvent: response.last_event,
            });
          }
        },
      );
    });
  }
}
```

### 8.4 Caché en Capas con Redis

```go
// internal/infrastructure/persistence/redis/cache_manager.go
package redis

type CacheManager struct {
    client *redis.Client
}

// L1: Estados actuales (TTL: 1 min)
func (cm *CacheManager) GetDeviceState(
    ctx context.Context,
    deviceID uuid.UUID,
) (*domain.DeviceState, error) {
    key := fmt.Sprintf("device:state:%s", deviceID)
    
    val, err := cm.client.Get(ctx, key).Result()
    if err == redis.Nil {
        return nil, ErrNotFound
    }
    if err != nil {
        return nil, err
    }
    
    var state domain.DeviceState
    if err := json.Unmarshal([]byte(val), &state); err != nil {
        return nil, err
    }
    
    return &state, nil
}

func (cm *CacheManager) SetDeviceState(
    ctx context.Context,
    state *domain.DeviceState,
) error {
    key := fmt.Sprintf("device:state:%s", state.DeviceID)
    
    val, err := json.Marshal(state)
    if err != nil {
        return err
    }
    
    return cm.client.Set(ctx, key, val, 1*time.Minute).Err()
}

// L2: Agregaciones (TTL: 5 min)
func (cm *CacheManager) GetDeviceAggregates(
    ctx context.Context,
    deviceID uuid.UUID,
    startTime, endTime time.Time,
    aggregationType string,
) (*domain.DeviceAggregates, error) {
    key := fmt.Sprintf(
        "device:agg:%s:%s:%d:%d",
        deviceID,
        aggregationType,
        startTime.Unix(),
        endTime.Unix(),
    )
    
    val, err := cm.client.Get(ctx, key).Result()
    if err == redis.Nil {
        return nil, ErrNotFound
    }
    if err != nil {
        return nil, err
    }
    
    var agg domain.DeviceAggregates
    if err := json.Unmarshal([]byte(val), &agg); err != nil {
        return nil, err
    }
    
    return &agg, nil
}

func (cm *CacheManager) SetDeviceAggregates(
    ctx context.Context,
    agg *domain.DeviceAggregates,
) error {
    key := fmt.Sprintf(
        "device:agg:%s:%s:%d:%d",
        agg.DeviceID,
        agg.Type,
        agg.StartTime.Unix(),
        agg.EndTime.Unix(),
    )
    
    val, err := json.Marshal(agg)
    if err != nil {
        return err
    }
    
    return cm.client.Set(ctx, key, val, 5*time.Minute).Err()
}

// Actualización de estados desde eventos
func (cm *CacheManager) UpdateDeviceStates(
    ctx context.Context,
    events []*domain.DeviceEvent,
) error {
    // Agrupar eventos por device
    deviceEvents := make(map[uuid.UUID]*domain.DeviceEvent)
    
    for _, event := range events {
        existing, ok := deviceEvents[event.DeviceID]
        if !ok || event.Time.After(existing.Time) {
            deviceEvents[event.DeviceID] = event
        }
    }
    
    // Actualizar cache para cada device
    for deviceID, lastEvent := range deviceEvents {
        state := &domain.DeviceState{
            DeviceID:  deviceID,
            Status:    "online",
            LastEvent: lastEvent,
            UpdatedAt: time.Now(),
        }
        
        if err := cm.SetDeviceState(ctx, state); err != nil {
            log.Error().
                Err(err).
                Str("device_id", deviceID.String()).
                Msg("Failed to update device state in cache")
        }
    }
    
    return nil
}
```

### 8.5 Dead Letter Queue (DLQ)

```go
// internal/application/services/dlq_manager.go
package services

type DLQManager struct {
    redisClient *redis.Client
}

// Guardar en DLQ
func (dlq *DLQManager) SaveFailedEvents(
    ctx context.Context,
    events []*domain.DeviceEvent,
    reason string,
) error {
    for _, event := range events {
        item := DLQItem{
            Event:     event,
            Reason:    reason,
            Timestamp: time.Now(),
            Retries:   0,
        }
        
        itemJSON, err := json.Marshal(item)
        if err != nil {
            continue
        }
        
        dlq.redisClient.LPush(ctx, "events:dlq", itemJSON)
    }
    
    return nil
}

// Procesador de DLQ (retry con backoff exponencial)
func (dlq *DLQManager) StartDLQProcessor(ctx context.Context) {
    ticker := time.NewTicker(30 * time.Second)
    defer ticker.Stop()
    
    for {
        select {
        case <-ctx.Done():
            return
            
        case <-ticker.C:
            dlq.processDLQ(ctx)
        }
    }
}

func (dlq *DLQManager) processDLQ(ctx context.Context) {
    // Obtener items de la DLQ
    items, err := dlq.redisClient.LRange(ctx, "events:dlq", 0, 99).Result()
    if err != nil {
        return
    }
    
    for _, itemJSON := range items {
        var item DLQItem
        if err := json.Unmarshal([]byte(itemJSON), &item); err != nil {
            continue
        }
        
        // Calcular backoff exponencial
        backoff := time.Duration(math.Pow(2, float64(item.Retries))) * time.Second
        if time.Since(item.Timestamp) < backoff {
            continue
        }
        
        // Intentar reprocessar
        if err := dlq.retryEvent(ctx, item.Event); err != nil {
            // Incrementar retries
            item.Retries++
            item.Timestamp = time.Now()
            
            // Si supera max retries, mover a DLQ permanente
            if item.Retries >= 5 {
                dlq.moveToPermDLQ(ctx, item)
                dlq.redisClient.LRem(ctx, "events:dlq", 1, itemJSON)
            } else {
                // Actualizar item en DLQ
                updatedJSON, _ := json.Marshal(item)
                dlq.redisClient.LRem(ctx, "events:dlq", 1, itemJSON)
                dlq.redisClient.LPush(ctx, "events:dlq", updatedJSON)
            }
        } else {
            // Éxito, remover de DLQ
            dlq.redisClient.LRem(ctx, "events:dlq", 1, itemJSON)
        }
    }
}

func (dlq *DLQManager) moveToPermDLQ(ctx context.Context, item DLQItem) {
    itemJSON, _ := json.Marshal(item)
    dlq.redisClient.LPush(ctx, "events:dlq:permanent", itemJSON)
    
    // Alertar equipo
    log.Error().
        Str("device_id", item.Event.DeviceID.String()).
        Int("retries", item.Retries).
        Str("reason", item.Reason).
        Msg("Event moved to permanent DLQ - requires manual intervention")
}
```

---

## 9. Estructura de Carpetas

### 9.1 Monolito Modular (NestJS)

```
hse-backend-monolith/
│
├── src/
│   ├── main.ts
│   ├── app.module.ts
│   │
│   ├── shared/                          # Código compartido
│   │   ├── infrastructure/
│   │   │   ├── database/
│   │   │   │   ├── prisma/
│   │   │   │   │   ├── prisma.service.ts
│   │   │   │   │   ├── schema.prisma
│   │   │   │   │   └── migrations/
│   │   │   │   └── database.module.ts
│   │   │   ├── cache/
│   │   │   │   ├── redis.service.ts
│   │   │   │   └── cache.module.ts
│   │   │   ├── grpc/
│   │   │   │   ├── proto/
│   │   │   │   │   └── devices.proto
│   │   │   │   ├── devices-grpc.client.ts
│   │   │   │   └── grpc.module.ts
│   │   │   └── monitoring/
│   │   │       ├── newrelic.service.ts
│   │   │       └── monitoring.module.ts
│   │   │
│   │   ├── domain/
│   │   │   ├── value-objects/
│   │   │   │   ├── email.vo.ts
│   │   │   │   ├── uuid.vo.ts
│   │   │   │   └── date-range.vo.ts
│   │   │   ├── exceptions/
│   │   │   │   ├── domain.exception.ts
│   │   │   │   ├── validation.exception.ts
│   │   │   │   └── not-found.exception.ts
│   │   │   └── interfaces/
│   │   │       └── repository.interface.ts
│   │   │
│   │   ├── application/
│   │   │   ├── decorators/
│   │   │   │   ├── current-user.decorator.ts
│   │   │   │   └── roles.decorator.ts
│   │   │   ├── guards/
│   │   │   │   ├── jwt-auth.guard.ts
│   │   │   │   ├── permissions.guard.ts
│   │   │   │   └── api-key.guard.ts
│   │   │   ├── interceptors/
│   │   │   │   ├── logging.interceptor.ts
│   │   │   │   └── transform.interceptor.ts
│   │   │   ├── pipes/
│   │   │   │   └── validation.pipe.ts
│   │   │   └── filters/
│   │   │       └── all-exceptions.filter.ts
│   │   │
│   │   └── utils/
│   │       ├── date.utils.ts
│   │       ├── hash.utils.ts
│   │       └── pagination.utils.ts
│   │
│   ├── modules/                         # Módulos de negocio
│   │   │
│   │   ├── auth/
│   │   │   ├── auth.module.ts
│   │   │   │
│   │   │   ├── domain/
│   │   │   │   ├── entities/
│   │   │   │   │   ├── user.entity.ts
│   │   │   │   │   └── api-key.entity.ts
│   │   │   │   ├── value-objects/
│   │   │   │   │   └── password.vo.ts
│   │   │   │   ├── repositories/
│   │   │   │   │   ├── user.repository.interface.ts
│   │   │   │   │   └── api-key.repository.interface.ts
│   │   │   │   └── services/
│   │   │   │       └── password.service.ts
│   │   │   │
│   │   │   ├── application/
│   │   │   │   ├── use-cases/
│   │   │   │   │   ├── login/
│   │   │   │   │   │   ├── login.use-case.ts
│   │   │   │   │   │   └── login.dto.ts
│   │   │   │   │   ├── register/
│   │   │   │   │   │   ├── register.use-case.ts
│   │   │   │   │   │   └── register.dto.ts
│   │   │   │   │   ├── refresh-token/
│   │   │   │   │   │   └── refresh-token.use-case.ts
│   │   │   │   │   └── validate-api-key/
│   │   │   │   │       └── validate-api-key.use-case.ts
│   │   │   │   └── services/
│   │   │   │       └── jwt.service.ts
│   │   │   │
│   │   │   └── infrastructure/
│   │   │       ├── persistence/
│   │   │       │   ├── prisma/
│   │   │       │   │   ├── user.repository.ts
│   │   │       │   │   └── api-key.repository.ts
│   │   │       │   └── mappers/
│   │   │       │       ├── user.mapper.ts
│   │   │       │       └── api-key.mapper.ts
│   │   │       │
│   │   │       ├── http/
│   │   │       │   ├── graphql/
│   │   │       │   │   ├── resolvers/
│   │   │       │   │   │   └── auth.resolver.ts
│   │   │       │   │   └── types/
│   │   │       │   │       └── auth.types.ts
│   │   │       │   └── rest/
│   │   │       │       └── controllers/
│   │   │       │           └── auth.controller.ts
│   │   │       │
│   │   │       └── strategies/
│   │   │           ├── jwt.strategy.ts
│   │   │           └── api-key.strategy.ts
│   │   │
│   │   ├── iot-core/
│   │   │   ├── iot-core.module.ts
│   │   │   │
│   │   │   ├── domain/
│   │   │   │   ├── entities/
│   │   │   │   │   ├── organization.entity.ts
│   │   │   │   │   ├── establishment.entity.ts
│   │   │   │   │   ├── area.entity.ts
│   │   │   │   │   ├── device.entity.ts
│   │   │   │   │   ├── role.entity.ts
│   │   │   │   │   ├── action.entity.ts
│   │   │   │   │   └── permission.entity.ts
│   │   │   │   ├── value-objects/
│   │   │   │   │   ├── device-status.vo.ts
│   │   │   │   │   └── location.vo.ts
│   │   │   │   ├── repositories/
│   │   │   │   │   ├── organization.repository.interface.ts
│   │   │   │   │   ├── establishment.repository.interface.ts
│   │   │   │   │   ├── device.repository.interface.ts
│   │   │   │   │   └── role.repository.interface.ts
│   │   │   │   └── services/
│   │   │   │       ├── permissions.service.ts
│   │   │   │       └── device-commands.service.ts
│   │   │   │
│   │   │   ├── application/
│   │   │   │   ├── use-cases/
│   │   │   │   │   ├── organizations/
│   │   │   │   │   │   ├── create-organization.use-case.ts
│   │   │   │   │   │   ├── list-organizations.use-case.ts
│   │   │   │   │   │   └── *.dto.ts
│   │   │   │   │   ├── establishments/
│   │   │   │   │   │   ├── create-establishment.use-case.ts
│   │   │   │   │   │   ├── assign-user-to-establishment.use-case.ts
│   │   │   │   │   │   └── *.dto.ts
│   │   │   │   │   ├── devices/
│   │   │   │   │   │   ├── control-device.use-case.ts
│   │   │   │   │   │   ├── get-device-status.use-case.ts
│   │   │   │   │   │   └── *.dto.ts
│   │   │   │   │   └── roles/
│   │   │   │   │       ├── create-role.use-case.ts
│   │   │   │   │       ├── assign-permissions.use-case.ts
│   │   │   │   │       └── *.dto.ts
│   │   │   │   └── services/
│   │   │   │       └── energy-analytics.service.ts
│   │   │   │
│   │   │   └── infrastructure/
│   │   │       ├── persistence/
│   │   │       │   ├── prisma/
│   │   │       │   │   ├── organization.repository.ts
│   │   │       │   │   ├── establishment.repository.ts
│   │   │       │   │   ├── device.repository.ts
│   │   │       │   │   └── role.repository.ts
│   │   │       │   └── mappers/
│   │   │       │       └── *.mapper.ts
│   │   │       │
│   │   │       ├── http/
│   │   │       │   ├── graphql/
│   │   │       │   │   ├── resolvers/
│   │   │       │   │   │   ├── organization.resolver.ts
│   │   │       │   │   │   ├── establishment.resolver.ts
│   │   │       │   │   │   ├── device.resolver.ts
│   │   │       │   │   │   └── role.resolver.ts
│   │   │       │   │   └── types/
│   │   │       │   │       └── *.types.ts
│   │   │       │   └── rest/
│   │   │       │       └── controllers/
│   │   │       │           └── *.controller.ts
│   │   │       │
│   │   │       └── grpc/
│   │   │           └── device-commands.client.ts
│   │   │
│   │   ├── booking/
│   │   │   ├── booking.module.ts
│   │   │   │
│   │   │   ├── domain/
│   │   │   │   ├── entities/
│   │   │   │   │   ├── guest.entity.ts
│   │   │   │   │   ├── booking.entity.ts
│   │   │   │   │   ├── notification.entity.ts
│   │   │   │   │   └── doorlock-code.entity.ts
│   │   │   │   ├── value-objects/
│   │   │   │   │   ├── booking-status.vo.ts
│   │   │   │   │   ├── date-range.vo.ts
│   │   │   │   │   └── doorlock-code.vo.ts
│   │   │   │   ├── repositories/
│   │   │   │   │   ├── guest.repository.interface.ts
│   │   │   │   │   └── booking.repository.interface.ts
│   │   │   │   └── services/
│   │   │   │       ├── booking-validation.service.ts
│   │   │   │       ├── doorlock-code-generator.service.ts
│   │   │   │       └── notification.service.ts
│   │   │   │
│   │   │   ├── application/
│   │   │   │   └── use-cases/
│   │   │   │       ├── create-booking.use-case.ts
│   │   │   │       ├── cancel-booking.use-case.ts
│   │   │   │       ├── confirm-booking.use-case.ts
│   │   │   │       ├── check-availability.use-case.ts
│   │   │   │       └── *.dto.ts
│   │   │   │
│   │   │   └── infrastructure/
│   │   │       ├── persistence/
│   │   │       │   └── prisma/
│   │   │       │       ├── guest.repository.ts
│   │   │       │       └── booking.repository.ts
│   │   │       └── http/
│   │   │           └── graphql/
│   │   │               └── resolvers/
│   │   │                   └── booking.resolver.ts
│   │   │
│   │   ├── smart-access/
│   │   │   ├── smart-access.module.ts
│   │   │   ├── domain/
│   │   │   ├── application/
│   │   │   └── infrastructure/
│   │   │
│   │   └── universities/
│   │       ├── universities.module.ts
│   │       ├── domain/
│   │       ├── application/
│   │       └── infrastructure/
│   │
│   └── config/
│       ├── app.config.ts
│       ├── database.config.ts
│       ├── redis.config.ts
│       ├── jwt.config.ts
│       └── graphql.config.ts
│
├── test/
│   ├── unit/
│   │   ├── auth/
│   │   ├── iot-core/
│   │   └── booking/
│   ├── integration/
│   │   └── *.spec.ts
│   └── e2e/
│       └── *.e2e-spec.ts
│
├── prisma/
│   ├── schema.prisma
│   ├── migrations/
│   └── seed.ts
│
├── proto/                               # Proto files para gRPC
│   └── devices.proto
│
├── scripts/
│   ├── generate-proto.sh
│   └── db-migrate.sh
│
├── .env.example
├── .env.development
├── .env.production
├── .gitignore
├── package.json
├── tsconfig.json
├── nest-cli.json
├── Dockerfile
├── docker-compose.yml
└── README.md
```

### 9.2 Devices Service (Golang)

```
hse-devices-service/
│
├── cmd/
│   └── server/
│       └── main.go
│
├── internal/
│   │
│   ├── domain/
│   │   ├── entities/
│   │   │   ├── device.go
│   │   │   ├── device_event.go
│   │   │   ├── device_state.go
│   │   │   ├── connection.go
│   │   │   └── command.go
│   │   ├── valueobjects/
│   │   │   ├── device_status.go
│   │   │   ├── connection_state.go
│   │   │   ├── event_type.go
│   │   │   └── protocol_type.go
│   │   └── repositories/
│   │       ├── device_repository.go
│   │       ├── event_repository.go
│   │       └── query_repository.go
│   │
│   ├── application/
│   │   │
│   │   ├── commands/                    # WRITE SIDE
│   │   │   ├── handlers/
│   │   │   │   ├── save_device_event.go
│   │   │   │   ├── send_device_command.go
│   │   │   │   └── update_device_status.go
│   │   │   └── dto/
│   │   │       └── command_dto.go
│   │   │
│   │   ├── queries/                     # READ SIDE
│   │   │   ├── handlers/
│   │   │   │   ├── get_device_status.go
│   │   │   │   ├── get_device_events.go
│   │   │   │   ├── get_device_analytics.go
│   │   │   │   └── get_device_graph_data.go
│   │   │   └── dto/
│   │   │       └── query_dto.go
│   │   │
│   │   └── services/
│   │       ├── connection_manager.go
│   │       ├── event_queue.go
│   │       ├── event_processor.go        # Batch processor
│   │       ├── cache_manager.go
│   │       └── dlq_manager.go            # Dead Letter Queue
│   │
│   ├── infrastructure/
│   │   │
│   │   ├── persistence/
│   │   │   ├── postgres/
│   │   │   │   ├── timescale/
│   │   │   │   │   ├── event_writer.go   # Batch writes
│   │   │   │   │   ├── event_reader.go   # Queries optimizadas
│   │   │   │   │   └── aggregates.go     # Continuous aggregates
│   │   │   │   ├── device_repository.go
│   │   │   │   └── gorm_config.go
│   │   │   └── redis/
│   │   │       ├── event_queue.go        # Cola de eventos
│   │   │       ├── device_cache.go       # Cache de estados
│   │   │       └── connection_state.go   # Estados de conexión
│   │   │
│   │   ├── protocols/                    # Adaptadores de protocolos
│   │   │   ├── factory.go                # Abstract Factory
│   │   │   ├── tcp/
│   │   │   │   ├── tcp_handler.go
│   │   │   │   ├── tcp_server.go
│   │   │   │   └── tcp_client.go
│   │   │   ├── websocket/
│   │   │   │   ├── ws_handler.go
│   │   │   │   └── ws_server.go
│   │   │   ├── modbus/
│   │   │   │   └── modbus_handler.go
│   │   │   └── mqtt/
│   │   │       └── mqtt_handler.go
│   │   │
│   │   ├── http/
│   │   │   ├── handlers/
│   │   │   │   ├── device_handler.go
│   │   │   │   ├── command_handler.go
│   │   │   │   └── health_handler.go
│   │   │   ├── middleware/
│   │   │   │   ├── auth.go
│   │   │   │   ├── logging.go
│   │   │   │   └── cors.go
│   │   │   └── router.go
│   │   │
│   │   └── grpc/
│   │       ├── proto/
│   │       │   ├── devices.proto
│   │       │   └── commands.proto
│   │       ├── server/
│   │       │   ├── devices_server.go
│   │       │   ├── commands_server.go
│   │       │   └── stream_manager.go     # Gestión de streams
│   │       ├── client/
│   │       │   └── monolith_client.go    # Cliente con circuit breaker
│   │       └── pb/                        # Código generado
│   │           └── *.pb.go
│   │
│   └── config/
│       ├── config.go
│       └── env.go
│
├── pkg/                                  # Código público
│   ├── logger/
│   │   └── logger.go
│   ├── errors/
│   │   └── errors.go
│   └── utils/
│       ├── retry.go
│       └── backoff.go
│
├── scripts/
│   ├── generate_proto.sh
│   ├── migrate.sh
│   └── seed_timescale.sh
│
├── migrations/
│   ├── 001_initial_schema.sql
│   ├── 002_timescale_setup.sql
│   └── 003_continuous_aggregates.sql
│
├── test/
│   ├── unit/
│   │   ├── domain/
│   │   └── application/
│   ├── integration/
│   │   └── *_test.go
│   └── mocks/
│       └── *.go
│
├── .env.example
├── .env.development
├── .env.production
├── .gitignore
├── go.mod
├── go.sum
├── Dockerfile
├── docker-compose.yml
└── README.md
```

---

## 10. Infraestructura Cloud (AWS)

### 10.1 Arquitectura en AWS

```
                    INTERNET
                       │
                       ▼
               ┌───────────────┐
               │  CloudFront   │ CDN
               │    (CDN)      │
               └───────┬───────┘
                       │
                       ▼
               ┌───────────────┐
               │      ALB      │ Load Balancer
               │ (Application  │
               │Load Balancer) │
               └───────┬───────┘
                       │
            ┌──────────┼──────────┐
            │                     │
            ▼                     ▼
    ┌──────────────┐      ┌──────────────┐
    │ ECS Fargate  │      │ ECS Fargate  │
    │  (Monolith)  │◄────►│  (Devices)   │
    │ 2-4 tasks    │ gRPC │ 2-4 tasks    │
    └──────┬───────┘      └──────┬───────┘
           │                     │
           │                     │
           └───────┬─────────────┘
                   │
        ┌──────────┼──────────┐
        │                     │
        ▼                     ▼
┌──────────────┐      ┌──────────────┐
│RDS PostgreSQL│      │ ElastiCache  │
│   (Multi-AZ) │      │    Redis     │
│+ TimescaleDB │      │(Cluster Mode)│
└──────────────┘      └──────────────┘
        │                     │
        └──────────┬──────────┘
                   │
                   ▼
           ┌───────────────┐
           │  CloudWatch   │ Logs
           │+ New Relic    │ Monitoring
           └───────────────┘
                   │
                   ▼
           ┌───────────────┐
           │    Secrets    │ Credenciales
           │   Manager     │
           └───────────────┘
```

### 10.2 Servicios AWS Detallados

#### 10.2.1 Compute

**ECS Fargate** (Monolito)
```yaml
Service: hse-monolith
Tasks: 2-4 (auto-scaling)
CPU: 1 vCPU
Memory: 2 GB
Health Check: /health/readiness
Auto-scaling:
  - Target CPU: 70%
  - Target Memory: 80%
  - Min tasks: 2
  - Max tasks: 10
```