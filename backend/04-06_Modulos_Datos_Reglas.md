# Plan de Migración HSE Backend - Secciones 4-6

**Continuación desde**: `01-03_Introduccion_Arquitectura_Stack.md`

---

## 4. Módulos del Sistema

### 4.1 Módulo de Autenticación (Auth)

**Responsabilidad**: Autenticar y autorizar usuarios y terceros que acceden a la plataforma.

**Tipos de Autenticación**:

#### 4.1.1 Usuarios Finales (Clientes)

**Método**: JWT + Cookies seguras

**Flujo**:
```
1. Usuario envía email + password
2. Backend valida credenciales (Argon2 hash)
3. Genera JWT token + refresh token
4. Almacena refresh token en cookie HttpOnly, SameSite=Strict
5. Retorna access token (vida corta: 15 min)
```

**Por qué cookies HttpOnly**:
- Protección contra XSS (JavaScript no puede leer la cookie)
- SameSite protege contra CSRF
- Refresh token de larga vida (7 días) seguro en cookie
- Access token de vida corta en memoria del cliente

#### 4.1.2 Terceros (Integraciones API)

**Método**: API Keys

**Funcionalidad**:
- Cada cliente puede generar sus propias API Keys
- Cada API Key tiene permisos específicos asignados
- Rate limiting por API Key
- Auditoría de uso (qué endpoints llamó, cuándo)
- Rotación de keys (regenerar sin downtime)

**Estructura de API Key**:
```
Formato: hse_live_abc123def456...
         │    │    │
         │    │    └─ Random string (32 chars)
         │    └─ Ambiente (live/test)
         └─ Prefijo (identificación visual)
```

**Permisos granulares**:
```typescript
// Ejemplo de API Key con permisos
{
  keyId: "key_123",
  clientId: "client_456",
  permissions: [
    "devices:read",      // Puede leer estados de dispositivos
    "devices:control",   // Puede enviar comandos
    "bookings:read",     // Puede consultar reservas
  ],
  rateLimit: {
    requests: 1000,
    window: "1h"
  }
}
```

**Caso de uso real**:
Un hotel tiene su propia app móvil y quiere mostrar:
- Estados de habitaciones en tiempo real
- Permitir check-in remoto (abrir puerta)

→ Generan API Key con permisos específicos
→ Pueden integrar sin acceso completo al sistema

### 4.2 Módulo IoT/Core

**Responsabilidad**: Gestión de eficiencia energética, automatización y control de dispositivos.

**Funcionalidades principales**:

#### 4.2.1 Gestión de Organizaciones
- Crear/actualizar/eliminar organizaciones
- Asignar partner company a organización
- Gestionar establecimientos de la organización

#### 4.2.2 Control de Dispositivos
- Enviar comandos a dispositivos (encender/apagar, ajustar, programar)
- Consultar estados en tiempo real
- Recibir notificaciones de cambios (vía gRPC stream)
- Configurar automatizaciones

#### 4.2.3 Análisis Energético
- Consultar consumo histórico
- Generar gráficas de tendencias
- Reportes de eficiencia
- Alertas de consumo anormal
- Comparativas (mes actual vs anterior)

#### 4.2.4 Automatizaciones
- Escenarios programados (ej: "Apagar todo a las 10 PM")
- Triggers basados en eventos (ej: "Si temperatura > 25°C → encender A/C")
- Horarios por día de semana

**Comunicación con Devices Service**:
```
Caso 1: Usuario controla dispositivo
Usuario → Monolito (IoT Module) → gRPC: SendDeviceCommand() → Devices Service → Dispositivo

Caso 2: Dispositivo reporta cambio
Dispositivo → Devices Service → gRPC stream: NotifyDeviceEvent() → Monolito (IoT Module) → WebSocket → Usuario

Caso 3: Usuario consulta histórico
Usuario → Monolito (IoT Module) → gRPC: GetDeviceEvents() → Devices Service → Response con datos
```

### 4.3 Módulo de Reservas (Booking)

**Responsabilidad**: Gestionar reservas de espacios con dispositivos IoT integrados (ej: hoteles, coworkings).

**Funcionalidades**:

#### 4.3.1 Gestión de Reservas
- Crear reserva con validación de disponibilidad
- Confirmar reserva por parte del huésped
- Cancelar reserva (con reglas de estado)
- Modificar reserva (fechas, huéspedes)
- Consultar historial de reservas

#### 4.3.2 Gestión de Huéspedes
- Registro de huéspedes con datos personales
- Gestión de acompañantes
- Relación recíproca (huésped puede tener acompañantes y viceversa)

#### 4.3.3 Códigos de Acceso Inteligente
- Generación automática de códigos para cerraduras
- Programación de validez temporal (check-in a check-out)
- Envío automático vía notificaciones
- Activación/desactivación automática por fechas

#### 4.3.4 Notificaciones Automáticas
- Email de confirmación con código de acceso
- SMS con instrucciones
- Recordatorios de check-in
- Instrucciones de check-out

**Flujo completo de reserva**:
```
1. Admin crea reserva para huésped
   → Valida disponibilidad (sin solapamiento)
   → Estado: pending

2. Huésped confirma reserva (email/SMS)
   → Estado: confirmed
   → Se genera código de cerradura
   → Código se programa en dispositivo (vía Devices Service)
   → Se envía notificación con instrucciones

3. Fecha de check-in llega
   → Estado: in-progress
   → Código se activa automáticamente

4. Fecha de check-out pasa
   → Estado: completed
   → Código se desactiva automáticamente
```

### 4.4 Módulo de Smart Access

**Responsabilidad**: Control de acceso vehicular y peatonal para residenciales y oficinas.

**Funcionalidades** (a detallar en futuras iteraciones):
- Control de acceso vehicular autorizado
- Reconocimiento de placas (LPR - License Plate Recognition)
- Gestión de permisos por propietarios de apartamentos
- Gestión de permisos por administración
- Registro de entradas/salidas
- Alertas de accesos no autorizados
- Lista de invitados temporales

### 4.5 Módulo de Universidades/Escuelas

**Responsabilidad**: Gestión de accesos para entidades educativas.

**Funcionalidades** (a detallar en futuras iteraciones):
- Control de acceso de estudiantes y personal
- Gestión de horarios de acceso
- Reportes de asistencia
- Integración con sistemas académicos
- Alertas de seguridad

---

## 5. Modelo de Datos

### 5.1 Módulo IoT/Core - Eficiencia Energética

#### 5.1.1 Entidad: Users

**Propósito**: Usuario principal del sistema con la mayor interacción en la plataforma.

```typescript
User {
  id: UUID
  email: string (unique)
  passwordHash: string          // Argon2
  firstName: string
  lastName: string
  roleId: UUID                   // FK to Roles
  partnerCompanyId?: UUID        // FK (null si es root o admin directo)
  organizationId?: UUID          // FK (null si es root o partner)
  isActive: boolean
  emailVerified: boolean
  lastLogin?: DateTime
  createdAt: DateTime
  updatedAt: DateTime
}
```

**Relaciones**:
- Pertenece a un `Role`
- Puede pertenecer a una `PartnerCompany` o `Organization`
- Relación muchos a muchos con `Establishments` vía `UserEstablishments`

**Reglas de negocio**:
- Email debe ser único en todo el sistema
- Password hash usa Argon2 (más seguro que bcrypt)
- `root` users no tienen `partnerCompanyId` ni `organizationId`

#### 5.1.2 Entidad: Roles

**Propósito**: Definir roles del sistema y roles personalizados por organizaciones.

```typescript
Role {
  id: UUID
  name: string
  description: string
  type: 'system' | 'custom'      // system = predefinido, custom = creado por usuario
  hierarchy: number               // 1=root, 2=partner, 3=admin, 4=user, 5+=custom
  organizationId?: UUID           // null para roles de sistema
  establishmentId?: UUID          // para roles específicos de un establecimiento
  createdBy: UUID
  createdAt: DateTime
  updatedAt: DateTime
}
```

**Roles del sistema** (predefinidos):
```typescript
const SYSTEM_ROLES = {
  root: { hierarchy: 1 },      // Acceso total al sistema
  partner: { hierarchy: 2 },   // Gestiona su partner company y organizaciones
  admin: { hierarchy: 3 },     // Gestiona su organización y establecimientos
  user: { hierarchy: 4 },      // Acceso limitado según permisos
}
```

**Jerarquía explicada**:
```
root (1)
  └─ Puede hacer TODO
  └─ Puede crear partners y admins
  └─ No tiene restricciones

partner (2)
  └─ Ve solo su PartnerCompany
  └─ Gestiona organizaciones de su partner
  └─ Puede crear admins y users

admin (3)
  └─ Ve solo su Organization
  └─ Gestiona establecimientos asignados
  └─ Puede crear users y roles custom

user (4)
  └─ Acceso según permisos específicos
  └─ No puede crear otros usuarios

custom (5+)
  └─ Roles específicos creados por admin
  └─ Ejemplos: "Gerente de Piso", "Técnico", "Recepcionista"
```

#### 5.1.3 Entidad: Actions

**Propósito**: Permisos atómicos que pueden ser asignados a roles.

```typescript
Action {
  id: UUID
  name: string                   // 'create_user', 'delete_establishment', etc.
  resource: string               // 'user', 'establishment', 'device', etc.
  action: string                 // 'create', 'read', 'update', 'delete', 'control'
  description: string
  category: string               // 'user_management', 'device_control', etc.
  isExclusive: boolean           // true si solo ciertos roles pueden tenerlo
  minHierarchy?: number          // hierarchy mínimo requerido (si es exclusive)
}
```

**Ejemplos de actions**:
```typescript
[
  {
    name: 'create_user',
    resource: 'user',
    action: 'create',
    category: 'user_management',
    isExclusive: true,
    minHierarchy: 2  // Solo partner y superiores
  },
  {
    name: 'control_device',
    resource: 'device',
    action: 'control',
    category: 'device_control',
    isExclusive: false  // Cualquier rol puede tenerlo
  },
  {
    name: 'view_analytics',
    resource: 'analytics',
    action: 'read',
    category: 'reports',
    isExclusive: false
  }
]
```

#### 5.1.4 Entidad: Permissions

**Propósito**: Tabla pivot entre Roles y Actions (muchos a muchos).

```typescript
Permission {
  roleId: UUID                   // FK to Roles
  actionId: UUID                 // FK to Actions
  grantedAt: DateTime
  grantedBy: UUID                // FK to Users (quien otorgó el permiso)
  
  // Primary Key: (roleId, actionId)
}
```

#### 5.1.5 Entidad: PartnerCompanies

**Propósito**: Empresas asociadas que distribuyen el sistema con su marca.

```typescript
PartnerCompany {
  id: UUID
  name: string
  legalName: string
  taxId: string                  // RUC, NIT, EIN, etc.
  contactEmail: string
  contactPhone: string
  address: string
  website?: string
  logo?: string                  // URL a S3
  isActive: boolean
  createdAt: DateTime
  updatedAt: DateTime
}
```

**Caso de uso**:
```
Ejemplo: "TechEnergy Solutions" es un partner
→ Consigue clientes (hoteles, oficinas)
→ Les vende el sistema HSE con su marca
→ Administrado por Horus en backend
→ TechEnergy ve solo SUS clientes, no los de otros partners
```

#### 5.1.6 Entidad: Organizations

**Propósito**: Organizaciones/empresas que usan el sistema.

```typescript
Organization {
  id: UUID
  name: string
  partnerCompanyId?: UUID        // null si es cliente directo de Horus
  industry: string               // 'hotel', 'office', 'retail', etc.
  taxId: string
  contactEmail: string
  contactPhone: string
  address: string
  isActive: boolean
  settings: JSONB                // configuraciones específicas
  createdAt: DateTime
  updatedAt: DateTime
}
```

**Ejemplo de jerarquía**:
```
PartnerCompany: "TechEnergy Solutions"
  └─ Organization: "Hotel Paradise"
  └─ Organization: "Office Tower Corp"

PartnerCompany: null (cliente directo Horus)
  └─ Organization: "JerÃ³nimo Martins"
```

#### 5.1.7 Entidad: Establishments

**Propósito**: Lugares físicos de una organización (sedes, sucursales).

```typescript
Establishment {
  id: UUID
  organizationId: UUID           // FK
  name: string
  address: string
  location: Point                // PostGIS (lat, lng)
  timezone: string               // 'America/Bogota'
  isActive: boolean
  settings: JSONB                // config específica del establecimiento
  createdAt: DateTime
  updatedAt: DateTime
}
```

**Ejemplo real**:
```
Organization: "JerÃ³nimo Martins" (supermercados)
  └─ Establishment: "Sede Barranquilla Norte"
  └─ Establishment: "Sede Barranquilla Sur"
  └─ Establishment: "Sede Cartagena Centro"
  └─ ... (50 establecimientos en total)
```

#### 5.1.8 Entidad: UserEstablishments

**Propósito**: Tabla pivot que asigna usuarios a establecimientos.

```typescript
UserEstablishment {
  userId: UUID                   // FK
  establishmentId: UUID          // FK
  roleId?: UUID                  // rol específico para este establecimiento
  assignedAt: DateTime
  assignedBy: UUID               // quien hizo la asignación
  
  // Primary Key: (userId, establishmentId)
}
```

**Por qué esta tabla**:
```
Problema: Un admin puede gestionar múltiples establecimientos
Solución: Le asignamos explícitamente cuáles puede ver

Ejemplo:
- Admin "Juan" gestiona: Sede Norte, Sede Sur
- Admin "María" gestiona: Sede Cartagena

→ Juan ve solo Norte y Sur
→ María ve solo Cartagena
```

#### 5.1.9 Entidad: Areas

**Propósito**: Subdivisión de un establecimiento (pisos, zonas, habitaciones).

```typescript
Area {
  id: UUID
  establishmentId: UUID          // FK
  name: string
  type: string                   // 'office', 'room', 'common_area', 'production', etc.
  parentAreaId?: UUID            // para jerarquías (ej: Piso 2 > Habitación 201)
  floor?: number
  capacity?: number              // personas/vehículos
  size?: number                  // m²
  metadata: JSONB                // datos específicos del tipo
  createdAt: DateTime
  updatedAt: DateTime
}
```

**Ejemplo de jerarquía**:
```
Establishment: "Hotel Paradise"
  └─ Area: "Piso 2" (type: floor, parentAreaId: null)
      └─ Area: "Habitación 201" (type: room, parentAreaId: "Piso 2")
      └─ Area: "Habitación 202" (type: room, parentAreaId: "Piso 2")
  └─ Area: "Lobby" (type: common_area, parentAreaId: null)
```

#### 5.1.10 Entidad: Devices

**Propósito**: Dispositivos IoT instalados en áreas.

```typescript
Device {
  id: UUID
  areaId: UUID                   // FK
  deviceType: string             // 'sensor', 'actuator', 'doorlock', 'thermostat', etc.
  manufacturer: string           // 'Siemens', 'Honeywell', etc.
  model: string
  serialNumber: string (unique)
  macAddress: string (unique)
  ipAddress?: string
  protocol: string               // 'tcp', 'modbus', 'mqtt', 'websocket'
  status: string                 // 'online', 'offline', 'error', 'maintenance'
  firmwareVersion?: string
  lastSeen?: DateTime
  metadata: JSONB                // configuración específica
  createdAt: DateTime
  updatedAt: DateTime
}
```

**Estados del dispositivo**:
- `online`: Conectado y funcionando
- `offline`: Sin conexión (> 5 minutos sin responder)
- `error`: Reportando errores
- `maintenance`: En mantenimiento programado

#### 5.1.11 Diagrama de Relaciones Completo

```
PartnerCompanies
    │
    ├──> Organizations
    │       │
    │       ├──> Establishments
    │       │       │
    │       │       ├──> UserEstablishments ←──┐
    │       │       │       ↓                  │
    │       │       │   [asignación]           │
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
                └──> (credentials, profile)
```

### 5.2 Módulo de Booking

#### 5.2.1 Entidad: Guests

```typescript
Guest {
  id: UUID
  firstName: string
  lastName: string
  email: string
  phone: string
  documentType: string           // 'passport', 'id', 'driver_license'
  documentNumber: string
  nationality: string
  dateOfBirth?: Date
  metadata: JSONB                // preferencias, notas
  createdAt: DateTime
  updatedAt: DateTime
}
```

#### 5.2.2 Entidad: GuestCompanions

**Propósito**: Relación recíproca muchos a muchos entre huéspedes.

```typescript
GuestCompanion {
  guestId: UUID                  // FK
  companionId: UUID              // FK
  relationship: string           // 'spouse', 'child', 'friend', 'colleague'
  addedAt: DateTime
  
  // Primary Key: (guestId, companionId)
}
```

**Ejemplo**:
```
Guest A: "John Doe"
  └─ Acompañante: "Jane Doe" (relationship: spouse)
  └─ Acompañante: "Kid Doe" (relationship: child)

Guest B: "Jane Doe" (también es guest independiente)
  └─ Acompañante: "John Doe" (relationship: spouse)
```

#### 5.2.3 Entidad: Bookings

```typescript
Booking {
  id: UUID
  guestId: UUID                  // FK
  areaId: UUID                   // FK (habitación/espacio)
  checkInDate: DateTime
  checkOutDate: DateTime
  status: 'pending' | 'confirmed' | 'in-progress' | 'completed' | 'cancelled'
  confirmationCode: string (unique)
  totalGuests: number
  specialRequests?: string
  metadata: JSONB
  createdBy: UUID                // quien creó la reserva
  createdAt: DateTime
  updatedAt: DateTime
  cancelledAt?: DateTime
  cancelledBy?: UUID
  cancellationReason?: string
}
```

**Estados explicados**:
- `pending`: Creada, esperando confirmación del huésped
- `confirmed`: Huésped confirmó, todo listo
- `in-progress`: Check-in realizado, estadía activa
- `completed`: Check-out realizado, reserva finalizada
- `cancelled`: Cancelada (registro se mantiene para historial)

#### 5.2.4 Entidad: DoorlockCodes

```typescript
DoorlockCode {
  id: UUID
  bookingId: UUID                // FK
  deviceId: UUID                 // FK (doorlock específico)
  code: string                   // código numérico (ej: "123456")
  validFrom: DateTime
  validUntil: DateTime
  isActive: boolean
  usageCount: number             // cuántas veces se ha usado
  maxUsages?: number             // null = ilimitado
  createdAt: DateTime
}
```

#### 5.2.5 Entidad: Notifications

```typescript
Notification {
  id: UUID
  bookingId: UUID                // FK
  guestId: UUID                  // FK
  type: string                   // 'booking_confirmed', 'check_in_reminder', etc.
  channel: string                // 'email', 'sms', 'push'
  recipient: string              // email o teléfono
  subject?: string
  message: string                // puede incluir código de acceso
  status: 'pending' | 'sent' | 'failed'
  sentAt?: DateTime
  error?: string                 // si falló, razón
  metadata: JSONB
  createdAt: DateTime
}
```

#### 5.2.6 Diagrama de Relaciones Booking

```
Guests ──┬──> GuestCompanions (self-referential, many-to-many)
         │
         └──> Bookings ──┬──> Areas ──> Devices (tipo: doorlock)
                         │                  │
                         ├──> DoorlockCodes ┘
                         │
                         └──> Notifications
```

### 5.3 Devices Service - Series Temporales

#### 5.3.1 Tabla: device_events (TimescaleDB Hypertable)

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
```

**Por qué esta estructura**:
- `time` como primera columna: Requerido por TimescaleDB
- `device_id` + `time`: Index compuesto para queries por dispositivo
- `metadata` como JSONB: Flexibilidad para datos específicos del evento
- Sin foreign keys: Performance (alta frecuencia de inserts)

**Tipos de eventos comunes**:
```typescript
const EVENT_TYPES = {
  // Sensores
  'temperature': { unit: '°C' },
  'humidity': { unit: '%' },
  'energy_consumption': { unit: 'kWh' },
  'motion_detected': { value: boolean },
  
  // Actuadores
  'state_change': { metadata: { from: 'off', to: 'on' } },
  'dimmer_adjusted': { value: 0-100 },
  
  // Cerraduras
  'door_opened': { metadata: { code_used: '123456' } },
  'door_closed': {},
  'access_denied': { metadata: { reason: 'invalid_code' } },
  
  // Alertas
  'alarm': { metadata: { alarm_type: 'fire' } },
  'error': { metadata: { error_code: 'E001' } },
}
```

#### 5.3.2 Estados Actuales en Redis

**Key pattern**: `device:state:{device_id}`

```typescript
DeviceState {
  deviceId: string
  status: 'online' | 'offline' | 'error'
  lastEvent: {
    type: string
    value: number
    unit: string
    timestamp: number            // Unix timestamp
  }
  connectionState: {
    connected: boolean
    lastSeen: number             // Unix timestamp
    protocol: string
    remoteAddress: string
  }
  updatedAt: number              // Unix timestamp
}
```

**TTL**: 1 minuto (se actualiza constantemente con eventos nuevos)

---

## 6. Reglas de Negocio y Flujos

### 6.1 Sistema de Roles y Permisos

#### 6.1.1 Jerarquía y Reglas de Creación

**Jerarquía completa**:
```
root (hierarchy: 1)
  └──> partner (hierarchy: 2)
        └──> admin (hierarchy: 3)
              ├──> user (hierarchy: 4)
              └──> custom_roles (hierarchy: 5+)
```

**Regla 1: Creación de roles**
```
Quién puede crear roles:
✅ root
✅ partner
✅ admin
❌ user

Restricción:
- Solo pueden asignar permisos que ellos mismos tienen
- NO pueden asignar permisos exclusivos de niveles superiores

Ejemplo:
admin (hierarchy: 3) crea rol "Gerente de Piso" (hierarchy: 5)
  ✅ Puede asignar: 'control_device', 'view_analytics'
  ❌ NO puede asignar: 'create_organization' (exclusivo de partner)
```

**Regla 2: Creación de usuarios**
```
Quién puede crear usuarios:
✅ root
✅ partner
✅ admin
❌ user

Restricción:
- Solo pueden asignar roles iguales o inferiores

Ejemplo:
admin (hierarchy: 3) crea usuario
  ✅ Puede asignar rol: admin (3), user (4), custom (5+)
  ❌ NO puede asignar rol: root (1), partner (2)
```

#### 6.1.2 Visibilidad por Rol

**Root**:
```typescript
// Ve TODO sin restricciones
const rootQuery = {
  // Sin filtros
}
```

**Partner**:
```typescript
// Ve solo su partner company y organizaciones asociadas
const partnerQuery = {
  partnerCompanyId: currentUser.partnerCompanyId,
  // O organizaciones sin partner (administradas por Horus)
  OR: {
    partnerCompanyId: null
  }
}
```

**Admin**:
```typescript
// Ve solo su organización y establecimientos asignados
const adminQuery = {
  organizationId: currentUser.organizationId,
  establishmentId: {
    in: userAssignedEstablishments // de UserEstablishments
  }
}
```

**User**:
```typescript
// Acceso según permisos específicos
// Cada query verifica permisos antes de ejecutar
if (!hasPermission(user, 'view_analytics')) {
  throw new ForbiddenException();
}
```

### 6.2 Flujo de Onboarding

```
Paso 1: Root crea entidad base
├─ Si es partner → Crea PartnerCompany
└─ Si es admin → Crea Organization

Paso 2: Root crea usuario
├─ Asigna rol (partner o admin)
├─ Asocia con entidad creada
└─ Envía invitación por email

Paso 3: Usuario recibe invitación
├─ Click en link (token temporal válido 48h)
├─ Acepta invitación
└─ Configura credenciales (password)

Paso 4: Primer login
├─ Wizard de configuración inicial
├─ Configuración de establecim ientos (si es admin)
└─ Puede empezar a usar el sistema
```

**Regla de consistencia**:
```
❌ Incorrecto:
1. Crear usuario sin PartnerCompany/Organization
2. Enviar invitación
→ Usuario no tiene contexto organizacional

✅ Correcto:
1. Crear PartnerCompany/Organization
2. Crear usuario asociado
3. Enviar invitación
→ Usuario tiene contexto completo desde el inicio
```

### 6.3 Módulo de Booking - Reglas de Reservas

#### 6.3.1 Estados y Transiciones Válidas

```
Transiciones permitidas:

pending ──────────────────────> confirmed
   │                                │
   │                                │
   └──> cancelled                   └──> in-progress ──> completed
                                         │
                                         X (NO permitido cancelar)

Reglas:
✅ pending → confirmed: Huésped confirma
✅ pending → cancelled: Se puede cancelar antes de confirmar
✅ confirmed → cancelled: Se puede cancelar después de confirmar
✅ confirmed → in-progress: Check-in automático
✅ in-progress → completed: Check-out automático
❌ in-progress → cancelled: NO PERMITIDO (ya está activa)
❌ completed → cancelled: NO PERMITIDO (ya finalizó)
```

#### 6.3.2 Validación de Solapamiento

**Regla principal**: No se pueden crear reservas que se solapen en fechas para la misma área.

**Excepciones** (se permite solapamiento si):
```typescript
  const ALLOWED_OVERLAP_STATES = ['cancelled'];

// Verificación
function canCreateBooking(newBooking, existingBookings) {
  const overlapping = existingBookings.filter(booking => {
    // 1. Verifica solapamiento de fechas
    const datesOverlap = 
      newBooking.checkIn < booking.checkOut && 
      newBooking.checkOut > booking.checkIn;
    
    // 2. Solo cuenta si NO está cancelada
    const isActive = !ALLOWED_OVERLAP_STATES.includes(booking.status);
    
    return datesOverlap && isActive;
  });
  
  return overlapping.length === 0;
}
```

**Margen de tiempo entre reservas**:
```typescript
const CHECK_IN_MARGIN = 60;  // minutos después del checkout
const CHECK_OUT_MARGIN = 30; // minutos antes del siguiente checkin

// Ejemplo:
// Reserva 1: checkout a las 11:00
// Reserva 2: checkin puede ser desde las 12:00 (11:00 + 60 min)
```

#### 6.3.3 Operaciones Permitidas por Estado

**Cancelar reserva**:
```typescript
// Estados permitidos
const CAN_CANCEL = ['pending', 'confirmed'];

// Estados NO permitidos
const CANNOT_CANCEL = ['in-progress', 'completed'];

// Implementación
function cancelBooking(booking) {
  if (CANNOT_CANCEL.includes(booking.status)) {
    throw new Error(
      `Cannot cancel booking in ${booking.status} state`
    );
  }
  
  booking.status = 'cancelled';
  booking.cancelledAt = new Date();
  // Registro se mantiene en DB
}
```

**Eliminar reserva**:
```typescript
// Estados permitidos (mismo que cancelar)
const CAN_DELETE = ['pending', 'confirmed'];

// Implementación
function deleteBooking(booking) {
  if (!CAN_DELETE.includes(booking.status)) {
    throw new Error(
      `Cannot delete booking in ${booking.status} state`
    );
  }
  
  // Eliminación física del registro
  await bookingRepository.delete(booking.id);
  // Ahora se puede crear nueva reserva en ese período
}
```

**Modificar reserva**:
```typescript
const CAN_MODIFY = ['pending', 'confirmed'];

function modifyBooking(booking, newDates) {
  if (!CAN_MODIFY.includes(booking.status)) {
    throw new Error(
      `Cannot modify booking in ${booking.status} state`
    );
  }
  
  // Validar que nuevas fechas no se solapen
  const canBook = canCreateBooking(newDates, otherBookings);
  if (!canBook) {
    throw new Error('New dates overlap with existing booking');
  }
  
  booking.checkInDate = newDates.checkIn;
  booking.checkOutDate = newDates.checkOut;
}
```

#### 6.3.4 Generación de Códigos de Acceso

```typescript
async function generateDoorlockCode(booking: Booking): Promise<string> {
  // 1. Obtener doorlock del área
  const area = await getArea(booking.areaId);
  const doorlock = await getAreaDoorlock(area.id);
  
  if (!doorlock) {
    throw new Error('No doorlock found for this area');
  }
  
  // 2. Generar código único (6 dígitos)
  let code: string;
  let attempts = 0;
  
  do {
    code = generateRandomCode(6); // '123456'
    const exists = await doorlockCodeExists(doorlock.id, code);
    if (!exists) break;
    
    attempts++;
    if (attempts > 10) {
      throw new Error('Could not generate unique code');
    }
  } while (true);
  
  // 3. Guardar en base de datos
  await saveDoorlockCode({
    bookingId: booking.id,
    deviceId: doorlock.id,
    code: code,
    validFrom: booking.checkInDate,
    validUntil: booking.checkOutDate,
    isActive: true
  });
  
  // 4. Programar en dispositivo vía Devices Service
  await devicesGrpcClient.programDoorlockCode({
    deviceId: doorlock.id,
    code: code,
    validFrom: booking.checkInDate.toISOString(),
    validUntil: booking.checkOutDate.toISOString()
  });
  
  // 5. Enviar notificación al huésped
  await sendNotification({
    bookingId: booking.id,
    guestId: booking.guestId,
    type: 'booking_confirmed',
    channel: 'email',
    message: `Your access code is: ${code}. Valid from ${booking.checkInDate} to ${booking.checkOutDate}.`
  });
  
  return code;
}
```

#### 6.3.5 Activación/Desactivación Automática

**Job scheduler** (cron):
```typescript
// Corre cada minuto
@Cron('* * * * *')
async handleBookingStateTransitions() {
  const now = new Date();
  
  // 1. Activar reservas que entran en check-in
  await db.booking.updateMany({
    where: {
      status: 'confirmed',
      checkInDate: { lte: now }
    },
    data: { status: 'in-progress' }
  });
  
  // 2. Completar reservas que pasaron check-out
  await db.booking.updateMany({
    where: {
      status: 'in-progress',
      checkOutDate: { lte: now }
    },
    data: { status: 'completed' }
  });
  
  // 3. Desactivar códigos de acceso expirados
  await db.doorlockCode.updateMany({
    where: {
      isActive: true,
      validUntil: { lte: now }
    },
    data: { isActive: false }
  });
  
  // 4. Notificar a Devices Service para desprogramar códigos
  const expiredCodes = await getExpiredCodes(now);
  for (const code of expiredCodes) {
    await devicesGrpcClient.removeDoorlockCode({
      deviceId: code.deviceId,
      code: code.code
    });
  }
}
```

---
