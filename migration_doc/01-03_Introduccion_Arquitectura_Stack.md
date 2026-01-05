# Plan de Migración HSE Backend - Secciones 1-3

**Versión**: 2.0  
**Fecha**: 2024-01-04  
**Estado**: ✅ Aprobado para ejecución

---

## Tabla de Contenidos General

1. **[Introducción y Contexto](#1-introducción-y-contexto)** ← En este documento
2. **[Arquitectura General](#2-arquitectura-general)** ← En este documento
3. **[Stack Tecnológico](#3-stack-tecnológico)** ← En este documento
4. Módulos del Sistema → Ver documento `04-06_Modulos_Datos_Reglas.md`
5. Modelo de Datos → Ver documento `04-06_Modulos_Datos_Reglas.md`
6. Reglas de Negocio y Flujos → Ver documento `04-06_Modulos_Datos_Reglas.md`
7. Arquitectura CQRS en Devices Service → Ver documento `07-09_CQRS_Patrones_Estructura.md`
8. Patrones Críticos de Implementación → Ver documento `07-09_CQRS_Patrones_Estructura.md`
9. Estructura de Carpetas → Ver documento `07-09_CQRS_Patrones_Estructura.md`
10. Infraestructura Cloud → Ver documento `10-11_Infraestructura_Migracion.md`
11. Estrategia de Migración → Ver documento `10-11_Infraestructura_Migracion.md`
12. Monitoreo y Observabilidad → Ver documento `12-13_Monitoreo_Ejecucion.md`
13. Plan de Ejecución → Ver documento `12-13_Monitoreo_Ejecucion.md`

---

## 1. Introducción y Contexto

### 1.1 Objetivos de la Migración

Este plan describe la migración del backend HSE hacia una arquitectura moderna que permita:

- **Migrar el core a tecnologías más robustas**: Transición de la arquitectura actual a un stack más maduro y mantenible
- **Implementar arquitectura bien definida**: Separación clara de responsabilidades con arquitectura hexagonal y CQRS
- **Facilitar escalabilidad**: Diseño que permite crecer sin cambios estructurales mayores
- **Mejorar mantenibilidad**: Código organizado, testeado y documentado
- **Garantizar alta disponibilidad**: Cero pérdida de datos durante deploys y actualizaciones

### 1.2 Contexto del Sistema

El sistema HSE maneja operaciones críticas de IoT con requisitos específicos:

**Volumen de Datos**:
- **100+ eventos por segundo** de controladores IoT en tiempo real
- Datos de series temporales para análisis y generación de gráficas
- Almacenamiento histórico para auditoría y reportes

**Funcionalidades Actuales**:
- Eficiencia energética y monitoreo en tiempo real
- Automatización de predios y espacios
- Control de acceso inteligente (puertas, vehículos)
- Sistema de reservas con integración IoT
- Análisis y reportes de consumo energético

**Desafíos Actuales**:
- Pérdida de datos durante deploys del backend
- Dificultad para escalar componentes independientemente
- Acoplamiento entre lógica de negocio y manejo de conexiones
- Falta de separación clara entre escrituras y lecturas

### 1.3 Decisiones Arquitectónicas Principales

#### 1.3.1 Separación de Servicios

**Decisión**: Dividir el sistema en dos servicios principales:

1. **Monolito Modular (NestJS)**: Contiene toda la lógica de negocio
2. **Devices Service (Golang)**: Maneja conexiones y eventos de dispositivos IoT

**Justificación**:

Esta separación resuelve varios problemas críticos:

- **Evita pérdida de datos durante deploys**: Cuando actualizamos lógica de negocio en el monolito, el Devices Service sigue recibiendo y almacenando eventos sin interrupción
- **Permite actualizaciones independientes**: Podemos actualizar reglas de negocio sin afectar la recolección de datos
- **Alta disponibilidad garantizada**: Si un servicio cae, el otro mantiene operación
- **Escalabilidad independiente**: Podemos escalar el Devices Service (muchas conexiones) sin escalar el monolito (lógica de negocio)

**Ejemplo del problema que resuelve**:
```
ANTES (Monolito único):
Deploy de nueva feature → Reinicio del servidor → 2-3 minutos sin recibir eventos
→ Pérdida de 12,000-18,000 eventos (100 eventos/seg × 120-180 segundos)

DESPUÉS (Servicios separados):
Deploy del monolito → Devices Service sigue activo → CERO eventos perdidos
```

#### 1.3.2 Golang para Devices Service

**Decisión**: Usar Golang en vez de Node.js para el servicio de dispositivos.

**Justificación técnica**:

**1. Alta concurrencia nativa**:
- Goroutines son mucho más livianas que threads (2KB vs 2MB de memoria)
- Scheduler eficiente para manejar 100,000+ goroutines simultáneas
- Channel-based communication para coordinación sin locks

**2. Bajo consumo de memoria**:
- Crítico para mantener miles de conexiones TCP persistentes
- Garbage collector optimizado para latencia baja
- Mejor uso de CPU comparado con Node.js para operaciones concurrentes

**3. Performance para I/O bound operations**:
- Runtime optimizado para operaciones de red
- syscalls eficientes para TCP/WebSocket
- Sin overhead del event loop de Node.js

**4. Simplicidad de deploy**:
- Binario único compilado, sin runtime ni node_modules
- Cross-compilation fácil para diferentes plataformas
- Menor superficie de ataque en seguridad

**Contexto de decisión**:
Con 100+ eventos por segundo constantes, necesitamos un lenguaje que:
- Maneje concurrencia sin penalizar performance
- Mantenga latencia baja (< 10ms) para ACKs
- Use memoria eficientemente para miles de conexiones

Go cumple estos requisitos mejor que Node.js para este caso de uso específico.

#### 1.3.3 CQRS en Devices Service

**Decisión**: Implementar patrón CQRS (Command Query Responsibility Segregation) en el servicio de dispositivos.

**Justificación**:

**Problema**: Tenemos dos patrones de acceso completamente diferentes:

1. **Escrituras (Commands)**:
   - Alta frecuencia: 100+ eventos/segundo
   - Operaciones simples: INSERT
   - Prioridad: Latencia ultra baja (< 10ms)
   - No pueden fallar (cada evento es valioso)

2. **Lecturas (Queries)**:
   - Frecuencia media: Usuarios consultando dashboards, gráficas
   - Operaciones complejas: Agregaciones, JOINs, filtros temporales
   - Prioridad: Resultado correcto y completo
   - Pueden cachear resultados

**Solución con CQRS**:

```
WRITE SIDE (optimizado para velocidad):
Evento → Validación mínima → Redis Queue → ACK (< 10ms)
Background: Batch processor → TimescaleDB

READ SIDE (optimizado para análisis):
Query → Redis Cache → TimescaleDB (raw + aggregates)
```

**Beneficios**:
- **Escalabilidad independiente**: Podemos añadir más workers para writes sin afectar reads
- **Optimización específica**: Base de datos de escritura vs base de datos de lectura pueden tener índices diferentes
- **Consistencia eventual aceptable**: Los dashboards pueden tener 1-5 segundos de delay, no es crítico
- **Sin bloqueos**: Writes y reads no compiten por los mismos recursos

#### 1.3.4 gRPC Bidireccional

**Decisión**: Usar gRPC con streaming bidireccional para comunicación entre servicios.

**Justificación**:

**1. Performance superior**:
- Protocol Buffers (binario) vs JSON (texto): ~7-10x más rápido
- HTTP/2 multiplexing: Múltiples requests en una conexión
- Header compression: Reduce overhead

**2. Streaming bidireccional nativo**:
```
Caso de uso real:
Monolito necesita ser notificado cuando cambia estado de dispositivo
→ Sin gRPC: Polling cada N segundos (ineficiente)
→ Con gRPC: Stream abierto, notificaciones instantáneas
```

**3. Contratos bien definidos**:
- Protocol Buffers define schema estricto
- Generación automática de código (tipos seguros)
- Versionado de APIs más fácil

**4. Comunicación eficiente**:
```
Ejemplo: 1000 notificaciones/minuto
REST: 1000 conexiones HTTP + parsing JSON = ~500ms overhead
gRPC: 1 stream + mensajes binarios = ~50ms overhead
```

**Por qué NO usamos REST**:
- REST es excelente para APIs públicas (terceros)
- Para comunicación interna alta frecuencia, gRPC es superior
- No necesitamos que sea "human-readable" (es interno)

---

## 2. Arquitectura General

### 2.1 Vista de Alto Nivel

```
┌─────────────────────────────────────────────────────────────┐
│                         CLIENTES                             │
│  (Web App, Mobile App, APIs de Terceros)                    │
└────────────────────┬────────────────────────────────────────┘
                     │
                     │ HTTP/HTTPS
                     │ GraphQL (usuarios) / REST (terceros)
                     │
┌────────────────────▼────────────────────────────────────────┐
│              MONOLITO MODULAR (NestJS)                       │
│                                                              │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │   Auth   │  │   IoT    │  │ Booking  │  │  Smart   │   │
│  │  Module  │  │   Core   │  │  Module  │  │  Access  │   │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘   │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │      Shared Infrastructure Layer                     │  │
│  │  (Database, Cache, gRPC Client, Monitoring)          │  │
│  └──────────────────────────────────────────────────────┘  │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       │ gRPC (Bidireccional, Port 50051)
                       │ - Monolito envía comandos
                       │ - Devices notifica eventos
                       │
┌──────────────────────▼──────────────────────────────────────┐
│              DEVICES SERVICE (Golang)                        │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐   │
│  │         WRITE SIDE (Commands - Alta Frecuencia)     │   │
│  │  TCP ─┐                                             │   │
│  │  WS  ─┤→ Event Queue (Redis) → Batch Processor     │   │
│  │  gRPC─┘                                             │   │
│  └─────────────────────────────────────────────────────┘   │
│                          ↓                                   │
│  ┌─────────────────────────────────────────────────────┐   │
│  │         READ SIDE (Queries - Optimizado)            │   │
│  │  Redis Cache → TimescaleDB (raw + aggregates)      │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          │ TCP/WebSocket (Ports 5000-5010)
                          │
┌─────────────────────────▼───────────────────────────────────┐
│              DISPOSITIVOS IoT                                │
│  (Controladores, Sensores, Actuadores, Cerraduras)          │
└─────────────────────────────────────────────────────────────┘
```

**Flujo de datos explicado**:

1. **Usuario → Monolito**: GraphQL para usuarios finales, REST para terceros
2. **Monolito → Devices**: gRPC para enviar comandos y recibir notificaciones
3. **Dispositivos → Devices**: TCP/WebSocket para enviar eventos
4. **Devices → Monolito**: gRPC stream para notificar cambios importantes

### 2.2 Servicio Monolito Modular

**Arquitectura Base**: Hexagonal (Puertos y Adaptadores)

**Principios**:
- **Dominio en el centro**: Lógica de negocio pura, sin dependencias externas
- **Puertos (interfaces)**: Definen contratos que el dominio necesita
- **Adaptadores**: Implementaciones concretas de los puertos (DB, HTTP, gRPC)

**Estructura por capas**:
```
┌─────────────────────────────────────┐
│     HTTP/GraphQL Adapters           │ ← Entrada (controllers, resolvers)
└──────────────┬──────────────────────┘
               ↓
┌─────────────────────────────────────┐
│     Application Layer                │ ← Casos de uso (use-cases)
│  (Orquesta el dominio)               │
└──────────────┬──────────────────────┘
               ↓
┌─────────────────────────────────────┐
│     Domain Layer                     │ ← Lógica de negocio pura
│  (Entities, Value Objects, Services) │
└──────────────┬──────────────────────┘
               ↓
┌─────────────────────────────────────┐
│     Infrastructure Adapters          │ ← Salida (DB, cache, gRPC)
└─────────────────────────────────────┘
```

**Características clave**:
- **Separación clara**: Cada módulo es independiente
- **Bajo acoplamiento**: Módulos se comunican via eventos o interfaces
- **Alta cohesión**: Todo lo relacionado a una funcionalidad está junto
- **Testeable**: Domain layer sin dependencias, fácil de testear

**Patrones de Diseño aplicados**:
- **Repository Pattern**: Abstracción de persistencia
- **Use Case Pattern**: Un caso de uso = una acción del usuario
- **Adapter Pattern**: Adapta interfaces externas al dominio
- **Singleton Pattern**: Para servicios compartidos (ej: Redis client)

**Ventaja de módulos**:
```typescript
// Cada módulo es independiente
modules/
  ├── auth/        ← Puede extraerse a microservicio
  ├── iot-core/    ← Puede extraerse a microservicio
  └── booking/     ← Puede extraerse a microservicio
```

### 2.3 Servicio de Devices/Connection

**Arquitectura Base**: CQRS (Command Query Responsibility Segregation)

**Por qué CQRS aquí**:
- Write side: Recibe 100+ eventos/seg, necesita ser ultra rápido
- Read side: Consultas complejas con agregaciones, necesita estar optimizado
- Son dos problemas completamente diferentes → dos soluciones optimizadas

**Write Side (Commands)**:
```
Características:
- Prioridad: Latencia (< 10ms)
- Operación: INSERT simple
- Volumen: Alto (100+ ops/seg)
- Estrategia: Queue + batch processing
```

**Read Side (Queries)**:
```
Características:
- Prioridad: Resultado completo y correcto
- Operación: SELECT con agregaciones
- Volumen: Medio (usuarios consultando)
- Estrategia: Cache agresivo + continuous aggregates
```

**Patrones de Diseño aplicados**:
- **Command Pattern**: Cada comando es un objeto
- **Abstract Factory**: Crea handlers según protocolo (TCP, WebSocket, Modbus)
- **Observer Pattern**: Notifica cambios a subscribers (monolito)
- **Circuit Breaker**: Protege contra fallos en cascada

**Ventaja de Go para este servicio**:
```go
// Goroutines permiten manejar miles de conexiones fácilmente
for {
    conn, _ := listener.Accept()
    go handleConnection(conn) // Nueva goroutine por conexión
}
```

### 2.4 Comunicación entre Servicios

**Canal Principal**: gRPC bidireccional

**Flujos de comunicación**:

1. **Monolito → Devices (Comandos)**:
```
Usuario clickea "Encender luz"
→ Monolito: CreateDeviceCommand use-case
→ gRPC call: SendDeviceCommand()
→ Devices: Encola comando y lo envía al dispositivo
→ Response: Command acknowledged
```

2. **Devices → Monolito (Notificaciones)**:
```
Dispositivo reporta "Alarma activada"
→ Devices: Procesa evento
→ gRPC stream: NotifyDeviceEvent()
→ Monolito: Recibe notificación en tiempo real
→ Acción: Envía notificación push a usuarios
```

3. **Monolito → Devices (Queries)**:
```
Usuario abre dashboard de consumo
→ Monolito: GetDeviceAnalytics use-case
→ gRPC call: GetDeviceEvents()
→ Devices: Query a cache/TimescaleDB
→ Response: Datos agregados
```

**Por qué NO usamos message queue (RabbitMQ, Kafka)**:
- Para comunicación request-response, gRPC es más simple
- Message queue añadiría complejidad innecesaria
- gRPC ya tiene retry, timeout, load balancing built-in
- Consideraremos message queue en el futuro si necesitamos:
  - Event sourcing completo
  - Múltiples consumers del mismo evento
  - Replay de eventos históricos

---

## 3. Stack Tecnológico

### 3.1 Monolito Modular (NestJS)

| Categoría | Tecnología | Versión | Justificación |
|-----------|-----------|---------|---------------|
| **Framework** | NestJS | ^10.0.0 | Framework maduro con arquitectura modular, DI nativo, excelente TypeScript support |
| **Lenguaje** | TypeScript | ^5.0.0 | Type safety, mejor DX, refactoring seguro |
| **Servidor HTTP** | Fastify | ^4.0.0 | 2x más rápido que Express, mejor para high throughput |
| **Base de Datos** | PostgreSQL | 15+ | ACID, mature, excelente para datos relacionales |
| **ORM** | Prisma | ^5.0.0 | Type-safe queries, migraciones automáticas, excelente DX |
| **Caché** | Redis | 7.0+ | In-memory store, soporte para pub/sub, persistencia opcional |
| **Autenticación** | JWT + Passport | - | Estándar industry, stateless, fácil de escalar |
| **Hashing** | Argon2 | latest | Más seguro que bcrypt, resistant a GPU attacks |
| **API** | Apollo Server | ^4.0.0 | GraphQL server maduro, buena integración con NestJS |
| **Comunicación** | @grpc/grpc-js | ^1.9.0 | Cliente gRPC oficial para Node.js |
| **Seguridad** | Helmet + Throttler | latest | Headers de seguridad + rate limiting |
| **Validación** | class-validator | ^0.14.0 | Validación declarativa con decorators |
| **Monitoreo** | New Relic | ^11.0.0 | APM completo, traces distribuidos, dashboards |
| **Testing** | Jest | ^29.0.0 | Testing framework estándar para Node.js |

**Por qué NestJS y no Express puro**:
- Arquitectura modular built-in
- Dependency injection nativo
- Soporte first-class para TypeScript
- Decorators para reducir boilerplate
- Ecosystem maduro (GraphQL, gRPC, WebSockets)

**Por qué Fastify y no Express**:
- Performance: ~2x más requests/segundo
- Schema-based validation built-in
- Async/await nativo
- Plugin architecture superior

**Por qué Prisma y no TypeORM**:
- Type safety superior (genera tipos desde schema)
- Migraciones más confiables
- Query performance mejor
- DX (Developer Experience) excelente

### 3.2 Devices Service (Golang)

| Categoría | Tecnología | Versión | Justificación |
|-----------|-----------|---------|---------------|
| **Lenguaje** | Go | 1.21+ | Concurrencia nativa, performance, simplicidad |
| **Base de Datos** | PostgreSQL + TimescaleDB | 15+ / 2.13+ | Series temporales optimizadas, compatible con PG |
| **ORM** | GORM | v1.25.0+ | ORM maduro para Go, migrations, associations |
| **Framework Web** | Echo | v4.11.0+ | Ligero, rápido, middleware robusto |
| **Redis Client** | go-redis | v9.0.0+ | Cliente Redis oficial para Go |
| **gRPC** | google.golang.org/grpc | v1.58.0+ | Implementación oficial de gRPC |
| **Protocol Buffers** | protobuf | v1.31.0+ | Serialización eficiente |
| **Circuit Breaker** | sony/gobreaker | v2.0.0+ | Patrón circuit breaker battle-tested |
| **Logging** | zerolog | v1.31.0+ | Structured logging, zero allocation |
| **Validación** | validator | v10.15.0+ | Struct validation con tags |
| **Testing** | testify | v1.8.0+ | Assertions y mocks |
| **Monitoreo** | go-agent (New Relic) | v3.25.0+ | APM para Go (opcional) |

**Por qué Echo y no Gin**:
- Middleware más flexible
- Mejor manejo de errors
- WebSocket support built-in
- Menos opinionated, más control

**Por qué GORM y no sqlx**:
- Migrations automáticas
- Associations (foreign keys) built-in
- Hooks para lifecycle events
- Balance entre abstracción y control

**Por qué zerolog**:
- Zero allocation logging (importante para performance)
- Structured logging (JSON output)
- Sampling support
- Context-aware logging

### 3.3 Base de Datos - TimescaleDB

**Decisión crítica**: PostgreSQL + extensión TimescaleDB

**Por qué TimescaleDB y NO PostgreSQL plain**:

1. **Particionamiento automático por tiempo**:
```sql
-- TimescaleDB hace esto automáticamente
SELECT create_hypertable('device_events', 'time');
-- Crea particiones (chunks) automáticas cada 7 días
```

2. **Compresión de datos antiguos**:
```sql
-- Comprime datos > 7 días (ahorra 90% de espacio)
ALTER TABLE device_events SET (timescaledb.compress);
-- Queries siguen funcionando, transparente para la app
```

3. **Continuous Aggregates** (pre-cálculos automáticos):
```sql
-- Vista materializada que se actualiza sola
CREATE MATERIALIZED VIEW device_events_hourly
WITH (timescaledb.continuous) AS
SELECT 
  time_bucket('1 hour', time) as hour,
  device_id,
  AVG(value) as avg_value
FROM device_events
GROUP BY hour, device_id;

-- Query instantáneo vs minutos en PostgreSQL plain
SELECT * FROM device_events_hourly WHERE hour >= NOW() - INTERVAL '7 days';
```

4. **Retención automática**:
```sql
-- Elimina datos > 2 años automáticamente
SELECT add_retention_policy('device_events', INTERVAL '2 years');
```

**Comparación de performance**:
```
Query: Promedio de consumo por hora, últimos 30 días

PostgreSQL plain:
- Scan: 260M rows
- Tiempo: ~45 segundos
- Uso de RAM: 4GB

TimescaleDB (con continuous aggregate):
- Scan: 720 rows (30 días × 24 horas)
- Tiempo: ~50ms
- Uso de RAM: 10MB

Mejora: 900x más rápido
```

**Instalación en RDS**:
```sql
-- TimescaleDB es una extensión de PostgreSQL
CREATE EXTENSION IF NOT EXISTS timescaledb;

-- Verificar
SELECT extname, extversion FROM pg_extension WHERE extname = 'timescaledb';
```

### 3.4 Infraestructura Base

| Servicio | Tecnología | Justificación |
|----------|-----------|---------------|
| **Contenedores** | Docker | Estándar industry, portabilidad |
| **Orquestación** | AWS ECS Fargate | Serverless containers, sin gestión de servers |
| **Base de Datos Managed** | AWS RDS PostgreSQL | Backups automáticos, Multi-AZ, patches automáticos |
| **Caché Managed** | AWS ElastiCache Redis | Redis gestionado, cluster mode, auto-failover |
| **Load Balancer** | AWS ALB | Layer 7 LB, SSL termination, health checks |
| **CDN** | CloudFront | Baja latencia global, DDoS protection |
| **Secrets** | AWS Secrets Manager | Rotación automática, encriptado, audit logs |
| **Logs** | CloudWatch + New Relic | Logs centralizados, búsqueda, alertas |
| **CI/CD** | GitHub Actions | Integración con GitHub, free para repos públicos |

**Por qué ECS Fargate y no Kubernetes**:
- Menos complejidad operacional
- Sin gestión de nodes
- Auto-scaling más simple
- Suficiente para nuestro caso de uso
- Consideraremos K8s cuando tengamos 20+ microservicios

**Por qué RDS y no PostgreSQL en EC2**:
- Backups automáticos
- Multi-AZ con failover automático (< 60 segundos)
- Patches de seguridad automáticos
- Read replicas fáciles
- Point-in-time recovery
- Monitoring built-in

**Por qué CloudWatch + New Relic y no solo CloudWatch**:
- CloudWatch: Logs e infraestructura (incluido con AWS)
- New Relic: APM, distributed tracing, mejor UX
- Combinación da visibilidad completa

---

**📄 Continúa en**: `04-06_Modulos_Datos_Reglas.md`