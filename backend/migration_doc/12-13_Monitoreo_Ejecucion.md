# Plan de Migración HSE Backend - Secciones 12-13

**Continuación desde**: `10-11_Infraestructura_Migracion.md`

---

## 12. Monitoreo y Observabilidad

### 12.1 New Relic - Application Performance Monitoring

**Por qué New Relic**:
- APM completo para Node.js y Go
- Distributed tracing entre servicios
- Dashboards personalizables
- Alertas inteligentes con machine learning
- Mejor UX que CloudWatch

#### 12.1.1 Configuración NestJS (Monolito)

**Instalación**:
```typescript
// newrelic.js (en la raíz del proyecto)
'use strict'

exports.config = {
  app_name: ['HSE Backend Monolith - Production'],
  license_key: process.env.NEW_RELIC_LICENSE_KEY,
  
  logging: {
    level: 'info',
    filepath: 'stdout'
  },
  
  distributed_tracing: {
    enabled: true
  },
  
  transaction_tracer: {
    enabled: true,
    transaction_threshold: 0.5 // 500ms
  },
  
  error_collector: {
    enabled: true,
    ignore_status_codes: [404, 401]
  },
  
  attributes: {
    enabled: true,
    include: [
      'request.parameters.*',
      'request.headers.user-agent'
    ],
    exclude: [
      'request.headers.authorization',
      'request.headers.cookie'
    ]
  }
}
```

**Inicialización** (debe ser lo primero):
```typescript
// main.ts
import 'newrelic'; // DEBE ser la primera línea

async function bootstrap() {
  const app = await NestFactory.create(AppModule, 
    new FastifyAdapter()
  );
  // ...
}
```

#### 12.1.2 Configuración Go (Devices Service)

```go
// cmd/server/main.go
import (
    "github.com/newrelic/go-agent/v3/newrelic"
)

func main() {
    // Inicializar New Relic
    app, err := newrelic.NewApplication(
        newrelic.ConfigAppName("HSE Devices Service - Production"),
        newrelic.ConfigLicense(os.Getenv("NEW_RELIC_LICENSE_KEY")),
        newrelic.ConfigDistributedTracerEnabled(true),
    )
    if err != nil {
        log.Fatal("Failed to initialize New Relic:", err)
    }
    
    // Instrumentar HTTP handlers
    http.HandleFunc(newrelic.WrapHandleFunc(app, "/api/devices", handleDevices))
    
    // Instrumentar queries de base de datos
    db, err := sql.Open("postgres", dsn)
    db = newrelic.WrapDatabase(db)
}
```

#### 12.1.3 Custom Metrics

**Métricas de negocio importantes**:
```typescript
// src/shared/infrastructure/monitoring/metrics.service.ts
@Injectable()
export class MetricsService {
  private newrelic = require('newrelic');
  
  // Contar eventos de negocio
  recordBusinessEvent(eventType: string, attributes: Record<string, any>) {
    this.newrelic.recordCustomEvent(eventType, attributes);
  }
  
  // Ejemplo: Reserva creada
  recordBookingCreated(booking: Booking) {
    this.recordBusinessEvent('BookingCreated', {
      bookingId: booking.id,
      guestId: booking.guestId,
      areaId: booking.areaId,
      checkInDate: booking.checkInDate,
      totalGuests: booking.totalGuests
    });
  }
  
  // Ejemplo: Comando a dispositivo
  recordDeviceCommand(deviceId: string, commandType: string, success: boolean) {
    this.recordBusinessEvent('DeviceCommand', {
      deviceId,
      commandType,
      success,
      timestamp: Date.now()
    });
  }
  
  // Métricas personalizadas
  recordMetric(name: string, value: number) {
    this.newrelic.recordMetric(`Custom/${name}`, value);
  }
}
```

### 12.2 Dashboards Críticos

#### 12.2.1 Dashboard de Overview

```
┌─────────────────────────────────────────────────┐
│         HSE Backend - Production Overview        │
├─────────────────────────────────────────────────┤
│                                                  │
│  Uptime (30d):  99.95% ✅                       │
│  Error Rate:    0.08%  ✅                       │
│  Apdex Score:   0.92   ✅                       │
│                                                  │
├─────────────────────────────────────────────────┤
│  Response Time (last hour)                       │
│  ┌────────────────────────────────────────┐    │
│  │ p50: 120ms                             │    │
│  │ p95: 380ms                             │    │
│  │ p99: 620ms ⚠️                          │    │
│  └────────────────────────────────────────┘    │
│                                                  │
│  Throughput:                                     │
│  ├─ Monolith:  450 rpm                          │
│  └─ Devices:   850 rpm                          │
│                                                  │
│  Active Users: 1,240                             │
│  Events/sec:   125                               │
└─────────────────────────────────────────────────┘
```

**Queries NRQL**:
```sql
-- Error rate
SELECT percentage(count(*), WHERE error IS true) 
FROM Transaction 
SINCE 1 hour ago

-- Response time percentiles
SELECT percentile(duration, 50, 95, 99) 
FROM Transaction 
SINCE 1 hour ago 
FACET appName

-- Throughput
SELECT rate(count(*), 1 minute) 
FROM Transaction 
SINCE 1 hour ago 
TIMESERIES

-- Active users (últimos 5 minutos)
SELECT uniqueCount(userId) 
FROM Transaction 
WHERE userId IS NOT NULL 
SINCE 5 minutes ago
```

#### 12.2.2 Dashboard de Database

```
┌─────────────────────────────────────────────────┐
│         Database Performance                     │
├─────────────────────────────────────────────────┤
│  PostgreSQL (RDS)                                │
│  ├─ Connections: 45/200 (22%)                   │
│  ├─ CPU: 35%                                     │
│  ├─ IOPS: 2,400/12,000                          │
│  └─ Storage: 180GB/500GB                        │
│                                                  │
│  Slow Queries (> 1s):                            │
│  ┌────────────────────────────────────────┐    │
│  │ SELECT * FROM device_events             │    │
│  │ WHERE time > ... AND device_id = ...    │    │
│  │ Duration: 1.2s | Count: 15 (last hour) │    │
│  │ ⚠️  Consider adding index               │    │
│  └────────────────────────────────────────┘    │
│                                                  │
│  Query Time by Operation:                        │
│  ├─ SELECT: avg 45ms                            │
│  ├─ INSERT: avg 12ms                            │
│  ├─ UPDATE: avg 28ms                            │
│  └─ DELETE: avg 35ms                            │
└─────────────────────────────────────────────────┘
```

#### 12.2.3 Dashboard de Devices Service (CQRS)

```
┌─────────────────────────────────────────────────┐
│         Devices Service - CQRS Metrics           │
├─────────────────────────────────────────────────┤
│  WRITE SIDE                                      │
│  ├─ Events/sec: 125                             │
│  ├─ Queue depth: 245 events                     │
│  ├─ Batch processing lag: 2.3s ✅              │
│  └─ DLQ size: 0 events ✅                       │
│                                                  │
│  READ SIDE                                       │
│  ├─ Cache hit rate: 87% ✅                      │
│  ├─ Query latency (p95): 45ms                   │
│  └─ Active queries: 12                          │
│                                                  │
│  gRPC                                            │
│  ├─ Connection state: READY ✅                  │
│  ├─ Streams active: 3                           │
│  ├─ Messages/sec: 45                            │
│  └─ Error rate: 0.02%                           │
└─────────────────────────────────────────────────┘
```

### 12.3 Alertas Críticas

#### 12.3.1 Configuración de Alertas

**New Relic Alert Policies**:
```yaml
policies:
  - name: "HSE Backend - Critical"
    incident_preference: "PER_CONDITION"
    channels: ["pagerduty", "slack-critical"]
    
    conditions:
      - name: "Error Rate High"
        type: "apm_app_metric"
        metric: "error_percentage"
        condition_scope: "application"
        threshold:
          critical: 1.0  # 1% error rate
          duration: 5    # minutos
        
      - name: "Response Time Degraded"
        type: "apm_app_metric"
        metric: "response_time_web"
        percentile: 99
        threshold:
          critical: 2000  # 2 segundos
          warning: 1000   # 1 segundo
          duration: 10    # minutos
      
      - name: "Event Processing Lag"
        type: "nrql"
        query: "SELECT average(processingLag) FROM Custom/Devices WHERE appName = 'HSE Devices Service'"
        threshold:
          critical: 60  # 60 segundos de lag
          duration: 5
      
      - name: "Database Connections Exhausted"
        type: "nrql"
        query: "SELECT percentage(count(*), WHERE databaseConnectionsUsed > 180) FROM Transaction"
        threshold:
          critical: 90  # 90% de conexiones usadas
          duration: 2
      
      - name: "DLQ Size Growing"
        type: "nrql"
        query: "SELECT latest(dlqSize) FROM Custom/Devices"
        threshold:
          critical: 1000  # 1000 eventos en DLQ
          warning: 100
          duration: 10
      
      - name: "gRPC Connection Down"
        type: "nrql"
        query: "SELECT latest(grpcConnectionState) FROM Custom/Devices WHERE grpcConnectionState != 'READY'"
        threshold:
          critical: 1  # cualquier estado != READY
          duration: 1

  - name: "HSE Backend - Warning"
    incident_preference: "PER_CONDITION"
    channels: ["slack-warnings", "email"]
    
    conditions:
      - name: "Cache Hit Rate Low"
        type: "nrql"
        query: "SELECT percentage(count(*), WHERE cacheHit IS true) FROM Custom/Devices"
        threshold:
          warning: 70  # < 70% hit rate
          duration: 15
      
      - name: "CPU Usage High"
        type: "infra_metric"
        metric: "cpuPercent"
        threshold:
          critical: 90
          warning: 80
          duration: 10
      
      - name: "Memory Usage High"
        type: "infra_metric"
        metric: "memoryUsedPercent"
        threshold:
          critical: 90
          warning: 85
          duration: 10
```

#### 12.3.2 Notificaciones

**Slack Integration**:
```
Canal: #hse-alerts-critical
Formato de mensaje:

🚨 CRITICAL: Error Rate High
──────────────────────────────
App: HSE Backend Monolith
Metric: Error rate is 1.5% (threshold: 1%)
Duration: 7 minutes
Incident: #12345

Actions:
• View in New Relic: https://...
• Runbook: https://wiki.../runbook-high-error-rate
• Acknowledge: Reply with "ack #12345"
```

**PagerDuty Integration** (para alertas críticas):
```
Escalation Policy:
├─ Level 1: On-call engineer (immediate)
├─ Level 2: Team lead (after 15 min)
└─ Level 3: Engineering manager (after 30 min)

Auto-resolve: No (requiere ack manual)
```

### 12.4 Logging Strategy

#### 12.4.1 Niveles de Log

```typescript
// Structured logging con contexto
import { Logger } from '@nestjs/common';

export class DeviceService {
  private readonly logger = new Logger(DeviceService.name);
  
  async controlDevice(deviceId: string, command: string) {
    const correlationId = uuid();
    
    // INFO: Eventos importantes de negocio
    this.logger.log({
      message: 'Device command initiated',
      correlationId,
      deviceId,
      command,
      userId: this.currentUser.id
    });
    
    try {
      const result = await this.devicesGrpc.sendCommand(deviceId, command);
      
      // INFO: Resultado exitoso
      this.logger.log({
        message: 'Device command successful',
        correlationId,
        deviceId,
        result
      });
      
      return result;
      
    } catch (error) {
      // ERROR: Fallos que requieren atención
      this.logger.error({
        message: 'Device command failed',
        correlationId,
        deviceId,
        command,
        error: error.message,
        stack: error.stack
      });
      
      throw error;
    }
  }
}
```

**Por qué structured logging**:
- Fácil de buscar y filtrar
- Contexto completo siempre disponible
- Correlation IDs para tracing
- Compatible con herramientas de análisis

#### 12.4.2 Retención y Costos

```
CloudWatch Logs:
├─ Retention: 30 días
├─ Ingestion: ~50 GB/mes
└─ Costo: ~$30/mes

New Relic Logs:
├─ Retention: 90 días
├─ Ingestion: ~50 GB/mes
└─ Costo: Incluido en plan Pro ($99/mes)

Lifecycle policy:
├─ Hot tier (0-7 días): CloudWatch + New Relic
├─ Warm tier (8-30 días): Solo New Relic
└─ Cold tier (31-90 días): Solo New Relic (compressed)
```

### 12.5 Distributed Tracing

**Trace completo de un request**:
```
User Request: POST /api/bookings
│
├─ [Monolith] Auth Middleware (5ms)
│   └─ Validate JWT token
│
├─ [Monolith] BookingController (2ms)
│   └─ Parse request body
│
├─ [Monolith] CreateBookingUseCase (180ms)
│   ├─ Validate availability (25ms)
│   │   └─ Query PostgreSQL
│   │
│   ├─ Create booking record (15ms)
│   │   └─ INSERT INTO bookings
│   │
│   ├─ Generate doorlock code (120ms)
│   │   ├─ Generate random code (1ms)
│   │   ├─ Save to database (10ms)
│   │   │
│   │   └─ [gRPC Call] Program doorlock (105ms) ←─┐
│   │       │                                      │
│   │       └─ [Devices Service] ReceiveCommand   │
│   │           ├─ Validate device (5ms)          │
│   │           ├─ Queue command (2ms)            │
│   │           ├─ Send to device (85ms) ← TCP    │
│   │           └─ Return ACK (2ms)               │
│   │                                              │
│   └─ Send notification (15ms)                   │
│       └─ Queue email job                        │
│                                                  │
└─ Total: 202ms                                    │
                                                   │
[Trace ID: abc-123-def-456] ──────────────────────┘
```

**Beneficios del distributed tracing**:
- Ver el request completo end-to-end
- Identificar cuellos de botella
- Debuggear fallos en servicios diferentes
- Entender dependencias entre servicios

---

## 13. Plan de Ejecución

### 13.1 Timeline Completo

```
┌────────────────────────────────────────────────────────────┐
│                    MIGRATION TIMELINE                       │
│                     (12 semanas)                            │
├────────────────────────────────────────────────────────────┤
│                                                             │
│ Semana 1-2: Preparación                                    │
│ ├─ Provisionar infraestructura AWS                        │
│ ├─ Configurar CI/CD                                       │
│ ├─ Setup monitoring (New Relic + CloudWatch)              │
│ └─ Tests de la infraestructura                            │
│                                                             │
│ Semana 3-4: Devices Service Standalone                     │
│ ├─ Deploy a staging                                        │
│ ├─ Load testing (200+ eventos/seg)                        │
│ ├─ Deploy a production (shadow mode)                      │
│ ├─ Dual write: viejo + nuevo                              │
│ └─ Validación continua de datos                           │
│                                                             │
│ Semana 5: Migración IoT/Core - Traffic Split 10%           │
│ ├─ Feature flag: 10% tráfico                              │
│ ├─ Monitoreo intensivo 24/7                               │
│ ├─ Comparación de métricas                                │
│ └─ Validación de funcionalidad                            │
│                                                             │
│ Semana 6: Migración IoT/Core - Traffic Split 50%           │
│ ├─ Incremento a 50% si semana 5 OK                        │
│ ├─ Monitoreo continuo                                      │
│ └─ Ajustes basados en feedback                            │
│                                                             │
│ Semana 7: Migración IoT/Core - Traffic Split 100%          │
│ ├─ Incremento a 100% si semana 6 OK                       │
│ ├─ Sistema viejo en standby                               │
│ └─ Celebración interna 🎉                                 │
│                                                             │
│ Semana 8: Migración Booking - Traffic Split 10-50%         │
│ ├─ Implementar módulo Booking                             │
│ ├─ Traffic split gradual                                  │
│ ├─ Validación de reglas de negocio                        │
│ └─ Tests con usuarios reales                              │
│                                                             │
│ Semana 9: Migración Booking - Traffic Split 100%           │
│ ├─ Incremento a 100%                                       │
│ ├─ Validación de códigos de acceso                        │
│ └─ Monitoreo de notificaciones                            │
│                                                             │
│ Semana 10: Monitoreo y Estabilización                      │
│ ├─ 7 días de operación 100% estable                       │
│ ├─ Revisión de métricas                                   │
│ ├─ Identificar optimizaciones                             │
│ └─ Preparar para apagar sistema viejo                     │
│                                                             │
│ Semana 11: Apagar Sistema Viejo                            │
│ ├─ Backup completo del sistema viejo                      │
│ ├─ Sistema viejo en read-only                             │
│ ├─ Validación final                                       │
│ └─ Apagado definitivo                                     │
│                                                             │
│ Semana 12: Optimizaciones Post-Migración                   │
│ ├─ Análisis de performance                                │
│ ├─ Implementar mejoras identificadas                      │
│ ├─ Documentación actualizada                              │
│ └─ Post-mortem y lecciones aprendidas                     │
│                                                             │
└────────────────────────────────────────────────────────────┘
```

### 13.2 Equipo Requerido

**Roles y responsabilidades**:

#### 13.2.1 Tech Lead (1 persona, full-time)
```
Responsabilidades:
├─ Decisiones arquitectónicas
├─ Code reviews críticos
├─ Resolución de blockers técnicos
├─ Comunicación con stakeholders
└─ Gestión de riesgos

Habilidades requeridas:
├─ Experiencia en migraciones grandes
├─ Conocimiento de NestJS y Go
├─ Experiencia con AWS
└─ Liderazgo técnico
```

#### 13.2.2 Backend Developers (2-3 personas, full-time)
```
Responsabilidades:
├─ Implementación de features
├─ Writing tests (unit + integration)
├─ Bug fixes
├─ Code reviews
└─ Documentación de código

Habilidades requeridas:
├─ TypeScript/NestJS
├─ Go (deseable, se puede aprender)
├─ PostgreSQL/TimescaleDB
└─ Testing (Jest, Go testing)
```

#### 13.2.3 DevOps Engineer (1 persona, 50% time)
```
Responsabilidades:
├─ Infraestructura AWS (IaC)
├─ CI/CD pipelines
├─ Monitoring setup
├─ Incident response
└─ Performance tuning

Habilidades requeridas:
├─ AWS (ECS, RDS, ElastiCache)
├─ Terraform o CloudFormation
├─ GitHub Actions
└─ New Relic / CloudWatch
```

#### 13.2.4 QA Engineer (1 persona, 50% time)
```
Responsabilidades:
├─ Test planning
├─ Manual testing crítico
├─ Automated E2E tests
├─ Load testing
└─ Bug reporting

Habilidades requeridas:
├─ Testing de APIs (Postman, etc)
├─ E2E testing (Cypress, Playwright)
├─ Load testing (k6, JMeter)
└─ SQL para validación de datos
```

#### 13.2.5 Product Owner (1 persona, 25% time)
```
Responsabilidades:
├─ Priorización de features
├─ Comunicación con clientes
├─ UAT (User Acceptance Testing)
├─ Documentación de negocio
└─ Gestión de expectativas

Habilidades requeridas:
├─ Conocimiento del dominio HSE
├─ Comunicación con stakeholders
└─ Gestión de producto
```

**Distribución del tiempo**:
```
Preparación (Sem 1-2):
├─ Tech Lead: 100%
├─ DevOps: 80%
├─ Developers: 60%
└─ QA: 40%

Migración activa (Sem 3-9):
├─ Tech Lead: 100%
├─ Developers: 100%
├─ DevOps: 50%
├─ QA: 60%
└─ PO: 30%

Estabilización (Sem 10-12):
├─ Tech Lead: 80%
├─ Developers: 80%
├─ DevOps: 40%
├─ QA: 40%
└─ PO: 20%
```

### 13.3 Presupuesto Total

#### 13.3.1 Infraestructura (3 meses)
```
AWS:
├─ Staging: $347 × 3 meses = $1,041
├─ Production: $1,944 × 3 meses = $5,832
└─ Subtotal: $6,873

Herramientas:
├─ New Relic Pro: $99 × 3 = $297
├─ GitHub Teams: $0 (si ya tienen)
└─ Subtotal: $297

Total Infraestructura: $7,170
```

#### 13.3.2 Personal (3 meses - estimación)
```
Salarios mensuales (estimado Colombia):
├─ Tech Lead: $8,000 × 3 = $24,000
├─ Backend Dev (3): $5,000 × 3 × 3 = $45,000
├─ DevOps (50%): $6,000 × 0.5 × 3 = $9,000
├─ QA (50%): $4,000 × 0.5 × 3 = $6,000
└─ PO (25%): $5,000 × 0.25 × 3 = $3,750

Total Personal: $87,750
```

#### 13.3.3 Contingencia
```
Imprevistos (10%): $8,775
Consultores externos (si necesario): $10,000

Total Contingencia: $18,775
```

#### 13.3.4 Total Inversión
```
┌────────────────────────────────────┐
│ PRESUPUESTO TOTAL MIGRACIÓN         │
├────────────────────────────────────┤
│ Infraestructura:     $7,170        │
│ Personal:            $87,750       │
│ Contingencia:        $18,775       │
├────────────────────────────────────┤
│ TOTAL:               $113,695      │
└────────────────────────────────────┘

ROI Esperado:
├─ Reducción downtime: ~$50,000/año
├─ Faster time-to-market: ~$30,000/año
├─ Reducción bugs críticos: ~$20,000/año
└─ Total beneficio anual: ~$100,000/año

Payback period: ~14 meses
```

### 13.4 Métricas de Éxito Post-Migración

**3 meses después de completar**:
```yaml
Technical Metrics:
  uptime:
    target: "> 99.9%"
    measurement: "New Relic Synthetics"
  
  error_rate:
    target: "< 0.1%"
    baseline_old: "0.15%"
    measurement: "New Relic APM"
  
  response_time_p99:
    target: "< 500ms"
    baseline_old: "650ms"
    measurement: "New Relic APM"
  
  events_processed:
    target: "100+ events/sec with < 5s lag"
    measurement: "Custom metrics"
  
  data_loss:
    target: "0 events lost"
    measurement: "Daily validation scripts"
  
  deploy_frequency:
    target: "Daily deploys without downtime"
    measurement: "GitHub Actions history"
  
  mttr:
    target: "< 30 minutes"
    measurement: "PagerDuty metrics"

Business Metrics:
  customer_satisfaction:
    target: "> 4.5/5"
    measurement: "NPS surveys"
  
  api_uptime_sla:
    target: "> 99.95%"
    commitment: "99.9%"
    measurement: "New Relic Synthetics"
  
  support_tickets:
    target: "No increase vs baseline"
    measurement: "Support system"
  
  feature_velocity:
    target: "2x faster feature development"
    measurement: "JIRA cycle time"
```

### 13.5 Lecciones Aprendidas (Post-Mortem Template)

**Documento a completar al finalizar**:
```markdown
# Post-Mortem: Migración Backend HSE

## Resumen Ejecutivo
- Duración real: X semanas
- Presupuesto: $X (vs estimado $113,695)
- Estado final: Exitosa/Con issues

## ¿Qué funcionó bien?
1.
2.
3.

## ¿Qué no funcionó?
1.
2.
3.

## Incidentes Críticos
### Incidente 1: [Título]
- Fecha/hora:
- Impacto:
- Root cause:
- Resolución:
- Acción preventiva:

## Métricas Alcanzadas
| Métrica | Target | Actual | Status |
|---------|--------|--------|--------|
| Uptime | 99.9% | X% | ✅/❌ |
| Error rate | < 0.1% | X% | ✅/❌ |
| ...

## Deuda Técnica Identificada
1.
2.
3.

## Recomendaciones para Futuras Migraciones
1.
2.
3.

## Agradecimientos
- Equipo técnico
- Stakeholders
- Clientes que participaron en beta
```

### 13.6 Siguientes Pasos Post-Migración

**Roadmap post-migración**:
```
Mes 1-2: Optimizaciones
├─ Implementar mejoras identificadas
├─ Refactoring de código con deuda técnica
├─ Optimización de queries lentas
└─ Mejora de tests coverage

Mes 3-4: Nuevas Features
├─ Módulo Smart Access (completo)
├─ Módulo Universities (completo)
├─ APIs para terceros mejoradas
└─ Dashboard de analytics avanzado

Mes 5-6: Escalabilidad
├─ Evaluar extracción de microservicios
├─ Implementar caching más agresivo
├─ Optimizar costs AWS (Reserved Instances)
└─ Horizontal scaling automático
```

---

## Conclusión

Este plan de migración proporciona una ruta completa, detallada y segura para evolucionar el backend HSE hacia una arquitectura moderna y escalable.
