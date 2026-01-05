# Plan de Migración HSE Backend - Secciones 7-9

**Continuación desde**: `04-06_Modulos_Datos_Reglas.md`

---

## 7. Arquitectura CQRS en Devices Service

### 7.1 Visión General de CQRS

**CQRS** (Command Query Responsibility Segregation) separa las operaciones de escritura (commands) de las operaciones de lectura (queries).

**Por qué lo necesitamos**:
```
Problema actual:
- 100+ eventos/segundo que debemos GUARDAR (writes)
- Usuarios consultando dashboards y gráficas (reads)
- Ambos compiten por los mismos recursos

Con CQRS:
WRITE SIDE                          READ SIDE
- Optimizado para velocidad         - Optimizado para consultas
- Queue + batch processing          - Cache + agregaciones pre-calculadas
- Latencia < 10ms                   - Queries complejas permitidas
- Sin bloqueos de lectura           - Sin impacto de escrituras
```

### 7.2 Arquitectura Completa

```
┌─────────────────────────────────────────────────────────────┐
│                  WRITE SIDE (Commands)                       │
│                                                              │
│  Eventos entrantes                                           │
│  ┌────────────┐   ┌────────────┐   ┌────────────┐         │
│  │TCP Server  │   │ WS Server  │   │gRPC Server │         │
│  │(Port 5000) │   │(Port 5001) │   │(Port 50051)│         │
│  └──────┬─────┘   └──────┬─────┘   └──────┬─────┘         │
│         │                │                │                 │
│         └────────────────┼────────────────┘                 │
│                          ▼                                   │
│                 ┌─────────────────┐                         │
│                 │ Event Validator │                         │
│                 │ (Schema check)  │                         │
│                 └────────┬────────┘                         │
│                          │                                   │
│                 ┌────────▼────────┐                         │
│                 │  Redis Queue    │ ← LPUSH (< 1ms)        │
│                 │  "events:queue" │                         │
│                 └────────┬────────┘                         │
│                          │                                   │
│                  [ACK inmediato]                            │
│                          │                                   │
│         ┌────────────────┴────────────────┐                │
│         ▼                                  ▼                 │
│  ┌──────────────┐                 ┌──────────────┐         │
│  │Redis Cache   │                 │Event Processor│         │
│  │(Update state)│                 │(Background)   │         │
│  └──────────────┘                 └──────┬───────┘         │
│                                           │                  │
│                                           ▼                  │
│                                  ┌─────────────────┐        │
│                                  │  TimescaleDB    │        │
│                                  │ (Batch inserts) │        │
│                                  └─────────────────┘        │
│                                                              │
│  Latencia total: < 10ms (solo Redis)                       │
│  Throughput: 1000+ eventos/seg                             │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                   READ SIDE (Queries)                        │
│                                                              │
│  ┌────────────┐                                             │
│  │   gRPC     │                                             │
│  │  Request   │                                             │
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
│ │ (L1)    │ │                  │                          │
│ │         │ │ Raw events       │                          │
│ │Hot data │ │ +                │                          │
│ │TTL: 1min│ │ Continuous       │                          │
│ │         │ │ aggregates       │                          │
│ │Hit: 85% │ │ (pre-calculated) │                          │
│ └─────────┘ └──────────────────┘                          │
│                                                             │
│  Query latency:                                            │
│  - Cache hit: < 5ms                                        │
│  - Cache miss: < 200ms                                     │
│  - Aggregate queries: < 100ms                              │
└─────────────────────────────────────────────────────────────┘
```

### 7.3 Write Side - Flujo Detallado

#### 7.3.1 Recepción de Eventos

```go
// internal/infrastructure/protocols/tcp/tcp_handler.go
func (h *TCPHandler) HandleConnection(conn net.Conn) {
    defer conn.Close()
    
    scanner := bufio.NewScanner(conn)
    for scanner.Scan() {
        rawData := scanner.Bytes()
        
        // Parsear evento
        event, err := h.parseEvent(rawData)
        if err != nil {
            log.Error().Err(err).Msg("Invalid event format")
            continue
        }
        
        // Validación básica
        if err := h.validator.Validate(event); err != nil {
            log.Error().Err(err).Msg("Validation failed")
            continue
        }
        
        // Encolar en Redis (< 1ms)
        if err := h.eventQueue.Enqueue(event); err != nil {
            log.Error().Err(err).Msg("Failed to enqueue")
            continue
        }
        
        // ACK inmediato al dispositivo
        conn.Write([]byte("ACK\n"))
    }
}
```

**Por qué ACK inmediato**:
- Dispositivo no espera a que guardemos en DB
- Sin timeouts en dispositivos
- Sin re-envíos innecesarios
- Redis es confiable (persistencia en disco)

#### 7.3.2 Event Queue en Redis

```go
// internal/infrastructure/persistence/redis/event_queue.go
type EventQueue struct {
    client *redis.Client
}

func (eq *EventQueue) Enqueue(event *domain.DeviceEvent) error {
    eventJSON, err := json.Marshal(event)
    if err != nil {
        return err
    }
    
    // LPUSH a la lista
    return eq.client.LPush(
        context.Background(),
        "events:queue",
        eventJSON,
    ).Err()
}

func (eq *EventQueue) DequeueMultiple(count int64) ([]*domain.DeviceEvent, error) {
    // RPOP múltiples elementos (batch)
    results, err := eq.client.RPopCount(
        context.Background(),
        "events:queue",
        count,
    ).Result()
    
    if err != nil {
        return nil, err
    }
    
    events := make([]*domain.DeviceEvent, 0, len(results))
    for _, result := range results {
        var event domain.DeviceEvent
        if err := json.Unmarshal([]byte(result), &event); err != nil {
            continue
        }
        events = append(events, &event)
    }
    
    return events, nil
}
```

#### 7.3.3 Batch Processor

```go
// internal/application/services/event_processor.go
func (ep *EventProcessor) Start(ctx context.Context) error {
    ticker := time.NewTicker(ep.flushInterval) // 5 segundos
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
            // Flush periódico cada 5 segundos
            if len(batch) > 0 {
                ep.flushBatch(ctx, batch)
                batch = batch[:0]
            }
            
        default:
            // Intentar leer eventos (max 100 por vez)
            events, err := ep.eventQueue.DequeueMultiple(100)
            if err != nil || len(events) == 0 {
                time.Sleep(100 * time.Millisecond)
                continue
            }
            
            batch = append(batch, events...)
            
            // Flush si batch está lleno
            if len(batch) >= ep.batchSize {
                ep.flushBatch(ctx, batch)
                batch = batch[:0]
            }
        }
    }
}

func (ep *EventProcessor) flushBatch(
    ctx context.Context,
    events []*domain.DeviceEvent,
) error {
    start := time.Now()
    
    // 1. Batch insert a TimescaleDB
    if err := ep.eventWriter.InsertBatch(ctx, events); err != nil {
        log.Error().Err(err).Msg("Batch insert failed")
        ep.dlqManager.SaveFailedEvents(ctx, events, err.Error())
        return err
    }
    
    // 2. Actualizar cache (estados actuales)
    ep.cacheManager.UpdateDeviceStates(ctx, events)
    
    // 3. Notificar eventos relevantes al monolito
    relevantEvents := ep.filterRelevantEvents(events)
    if len(relevantEvents) > 0 {
        ep.grpcNotifier.NotifyEvents(ctx, relevantEvents)
    }
    
    duration := time.Since(start)
    log.Info().
        Int("count", len(events)).
        Dur("duration", duration).
        Msg("Batch processed")
    
    return nil
}
```

**Ventajas del batch processing**:
```
Sin batching (1 insert por evento):
- 100 eventos = 100 queries a DB
- Overhead de red: 100 × 5ms = 500ms
- Tiempo total: ~1 segundo

Con batching (1 insert para 100 eventos):
- 100 eventos = 1 query a DB
- Overhead de red: 1 × 5ms = 5ms
- Tiempo total: ~50ms

Mejora: 20x más rápido
```

### 7.4 Read Side - Optimización de Consultas

#### 7.4.1 Caché en Capas

```go
// internal/application/queries/handlers/get_device_status.go
func (h *GetDeviceStatusHandler) Handle(
    ctx context.Context,
    deviceID uuid.UUID,
) (*DeviceStatus, error) {
    // Nivel 1: Redis cache (estados actuales)
    cached, err := h.cacheManager.GetDeviceState(ctx, deviceID)
    if err == nil {
        return cached, nil // Hit: < 5ms
    }
    
    // Nivel 2: TimescaleDB (último evento)
    lastEvent, err := h.eventReader.GetLatestEvent(ctx, deviceID)
    if err != nil {
        return nil, err
    }
    
    // Construir estado y cachear
    status := &DeviceStatus{
        DeviceID:  deviceID,
        Status:    determineStatus(lastEvent),
        LastEvent: lastEvent,
        UpdatedAt: time.Now(),
    }
    
    h.cacheManager.SetDeviceState(ctx, status)
    
    return status, nil
}
```

#### 7.4.2 TimescaleDB - Continuous Aggregates

**Problema**: Gráfica de consumo promedio por hora de los últimos 30 días.

**Sin continuous aggregates**:
```sql
-- Escanea millones de rows
SELECT 
  date_trunc('hour', time) as hour,
  AVG(value) as avg_consumption
FROM device_events
WHERE device_id = 'abc-123'
  AND time >= NOW() - INTERVAL '30 days'
  AND event_type = 'energy_consumption'
GROUP BY hour
ORDER BY hour;

-- Tiempo: 10-30 segundos (depende de data)
```

**Con continuous aggregates**:
```sql
-- 1. Crear vista materializada (una vez)
CREATE MATERIALIZED VIEW device_events_hourly
WITH (timescaledb.continuous) AS
SELECT 
  time_bucket('1 hour', time) AS hour,
  device_id,
  event_type,
  AVG(value) AS avg_value,
  MAX(value) AS max_value,
  MIN(value) AS min_value,
  COUNT(*) AS event_count
FROM device_events
GROUP BY hour, device_id, event_type;

-- 2. Política de refresh automático
SELECT add_continuous_aggregate_policy(
  'device_events_hourly',
  start_offset => INTERVAL '3 hours',
  end_offset => INTERVAL '1 hour',
  schedule_interval => INTERVAL '10 minutes'
);

-- 3. Query usa la vista (instantáneo)
SELECT hour, avg_value
FROM device_events_hourly
WHERE device_id = 'abc-123'
  AND event_type = 'energy_consumption'
  AND hour >= NOW() - INTERVAL '30 days'
ORDER BY hour;

-- Tiempo: < 100ms
```

**Agregaciones disponibles**:
```sql
-- Por hora (para gráficas de hoy/ayer)
device_events_hourly

-- Por día (para gráficas de semana/mes)
device_events_daily

-- Por semana (para gráficas de año)
device_events_weekly
```

### 7.5 Consistencia Eventual

**Trade-off aceptado**:
```
Evento ocurre en dispositivo → Llega a Devices Service → Se guarda en queue
                                                              ↓
                                         [1-5 segundos de delay]
                                                              ↓
                                     Batch processor → TimescaleDB → Cache actualizado
                                                              ↓
                                                    Usuario ve el evento

Delay total: 1-5 segundos
```

**Por qué es aceptable**:
- Dashboards no requieren actualización instantánea
- Usuarios entienden "datos casi en tiempo real"
- Para acciones críticas (alarmas), usamos notificaciones directas vía gRPC

**Para eventos críticos** (bypass del delay):
```go
func (ep *EventProcessor) processEvent(event *domain.DeviceEvent) {
    // Eventos críticos se notifican inmediatamente
    if event.Type == "alarm" || event.Type == "error" {
        ep.grpcNotifier.NotifyImmediately(event) // < 100ms
    }
    
    // Todos los eventos van a queue normal
    ep.eventQueue.Enqueue(event)
}
```

---

## 8. Patrones Críticos de Implementación

### 8.1 Circuit Breaker

**Problema**: Si el monolito cae, el Devices Service no debe colapsar intentando notificarlo.

**Solución**: Circuit Breaker que detecta fallos y "abre" el circuito.

```go
// internal/infrastructure/grpc/client/monolith_client.go
import "github.com/sony/gobreaker"

type MonolithClient struct {
    client  pb.DeviceStreamClient
    breaker *gobreaker.CircuitBreaker
}

func NewMonolithClient(conn *grpc.ClientConn) *MonolithClient {
    settings := gobreaker.Settings{
        Name:        "monolith-grpc",
        MaxRequests: 3,              // requests en half-open
        Interval:    10 * time.Second,
        Timeout:     30 * time.Second,
        ReadyToTrip: func(counts gobreaker.Counts) bool {
            // Abre si 60% de requests fallan
            failureRatio := float64(counts.TotalFailures) / float64(counts.Requests)
            return counts.Requests >= 3 && failureRatio >= 0.6
        },
        OnStateChange: func(name string, from, to gobreaker.State) {
            log.Warn().
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
        // Circuito abierto, guardar en DLQ
        log.Warn().Msg("Circuit open, saving to DLQ")
        return mc.dlqManager.SaveFailedEvents(events)
    }
    
    return err
}
```

**Estados del Circuit Breaker**:
```
CLOSED (normal)
  ↓ (múltiples fallos)
OPEN (no intenta llamadas)
  ↓ (después de timeout)
HALF-OPEN (permite algunas llamadas de prueba)
  ↓
  ├─ Si funcionan → CLOSED
  └─ Si fallan → OPEN
```

### 8.2 Dead Letter Queue (DLQ)

**Problema**: Si el batch insert falla, no podemos perder esos eventos.

**Solución**: Guardar eventos fallidos en DLQ para retry posterior.

```go
// internal/application/services/dlq_manager.go
type DLQItem struct {
    Event     *domain.DeviceEvent
    Reason    string
    Timestamp time.Time
    Retries   int
}

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
        
        itemJSON, _ := json.Marshal(item)
        dlq.redisClient.LPush(ctx, "events:dlq", itemJSON)
    }
    
    return nil
}

// Processor de DLQ (retry con exponential backoff)
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
    items, _ := dlq.redisClient.LRange(ctx, "events:dlq", 0, 99).Result()
    
    for _, itemJSON := range items {
        var item DLQItem
        json.Unmarshal([]byte(itemJSON), &item)
        
        // Exponential backoff
        backoff := time.Duration(math.Pow(2, float64(item.Retries))) * time.Second
        if time.Since(item.Timestamp) < backoff {
            continue
        }
        
        // Reintentar
        err := dlq.eventWriter.InsertBatch(ctx, []*domain.DeviceEvent{item.Event})
        if err != nil {
            item.Retries++
            item.Timestamp = time.Now()
            
            if item.Retries >= 5 {
                // Mover a DLQ permanente (requiere intervención manual)
                dlq.moveToPermDLQ(ctx, item)
                dlq.redisClient.LRem(ctx, "events:dlq", 1, itemJSON)
            } else {
                // Actualizar retry count
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
```

**Backoff exponencial**:
```
Retry 1: 1 segundo después
Retry 2: 2 segundos después
Retry 3: 4 segundos después
Retry 4: 8 segundos después
Retry 5: 16 segundos después
→ Si falla 5 veces: DLQ permanente (alerta al equipo)
```

### 8.3 gRPC Streaming Bidireccional

#### 8.3.1 Protocol Buffers Definition

```protobuf
// internal/infrastructure/grpc/proto/devices.proto
syntax = "proto3";
package devices;

service DeviceStream {
  // Monolito se suscribe a cambios de dispositivos
  rpc SubscribeToDeviceChanges(SubscriptionRequest) 
    returns (stream DeviceChangeEvent);
  
  // Monolito envía comandos
  rpc SendDeviceCommand(DeviceCommand) 
    returns (CommandResponse);
  
  // Queries de datos
  rpc GetDeviceStatus(DeviceStatusRequest)
    returns (DeviceStatusResponse);
}

message SubscriptionRequest {
  repeated string device_ids = 1;
  repeated string event_types = 2;  // 'alarm', 'error', 'state_change'
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
  string request_id = 4;
}

message CommandResponse {
  bool success = 1;
  string message = 2;
  string request_id = 3;
}
```

#### 8.3.2 Server Implementation (Go)

```go
// internal/infrastructure/grpc/server/devices_server.go
func (s *DeviceServer) SubscribeToDeviceChanges(
    req *pb.SubscriptionRequest,
    stream pb.DeviceStream_SubscribeToDeviceChangesServer,
) error {
    ctx := stream.Context()
    
    // Canal para este cliente
    clientChan := make(chan *domain.DeviceEvent, 100)
    clientID := req.ClientId
    
    // Registrar subscriber
    s.connectionManager.RegisterSubscriber(
        clientID,
        clientChan,
        req.DeviceIds,
        req.EventTypes,
    )
    defer s.connectionManager.UnregisterSubscriber(clientID)
    
    log.Info().Str("client_id", clientID).Msg("Client subscribed")
    
    // Stream eventos
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
                return err
            }
            
        case <-ctx.Done():
            log.Info().Str("client_id", clientID).Msg("Client disconnected")
            return nil
        }
    }
}
```

#### 8.3.3 Client Implementation (TypeScript)

```typescript
// src/shared/infrastructure/grpc/devices-grpc.client.ts
@Injectable()
export class DevicesGrpcClient implements OnModuleInit {
  private client: any;
  private deviceChanges$ = new Subject<DeviceChangeEvent>();
  private stream: any;

  async onModuleInit() {
    const packageDef = protoLoader.loadSync('./proto/devices.proto');
    const proto = grpc.loadPackageDefinition(packageDef);
    
    this.client = new proto.devices.DeviceStream(
      process.env.DEVICES_GRPC_URL,
      grpc.credentials.createInsecure(),
    );
    
    await this.subscribeToDeviceChanges();
  }

  private async subscribeToDeviceChanges() {
    const request = {
      device_ids: [],  // vacío = todos
      event_types: ['alarm', 'state_change', 'error'],
      client_id: `monolith-${process.env.INSTANCE_ID}`,
    };

    this.stream = this.client.SubscribeToDeviceChanges(request);

    this.stream.on('data', (event: any) => {
      this.deviceChanges$.next({
        deviceId: event.device_id,
        eventType: event.event_type,
        timestamp: new Date(event.timestamp * 1000),
        value: event.value,
        unit: event.unit,
      });
    });

    this.stream.on('error', (error: any) => {
      console.error('gRPC stream error:', error);
      setTimeout(() => this.subscribeToDeviceChanges(), 5000);
    });

    this.stream.on('end', () => {
      console.log('gRPC stream ended, reconnecting...');
      setTimeout(() => this.subscribeToDeviceChanges(), 1000);
    });
  }

  // Observable para módulos que necesiten escuchar cambios
  onDeviceChange(): Observable<DeviceChangeEvent> {
    return this.deviceChanges$.asObservable();
  }

  // Enviar comando
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
          if (error) reject(error);
          else resolve(response);
        },
      );
    });
  }
}
```

### 8.4 Health Checks Robustos

```go
// cmd/server/main.go
func setupHealthChecks(e *echo.Echo, deps *Dependencies) {
    // Liveness: Solo verifica que el proceso esté vivo
    e.GET("/health/liveness", func(c echo.Context) error {
        return c.JSON(200, map[string]string{
            "status": "alive",
        })
    })
    
    // Readiness: Verifica que todas las dependencias estén listas
    e.GET("/health/readiness", func(c echo.Context) error {
        checks := make(map[string]bool)
        
        // Check PostgreSQL
        ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
        defer cancel()
        err := deps.DB.PingContext(ctx)
        checks["postgres"] = (err == nil)
        
        // Check Redis
        err = deps.Redis.Ping(ctx).Err()
        checks["redis"] = (err == nil)
        
        // Check gRPC server
        checks["grpc_server"] = deps.GRPCServer.IsRunning()
        
        // Check TCP server
        checks["tcp_server"] = deps.TCPServer.IsRunning()
        
        // Determinar si está ready
        allHealthy := true
        for _, healthy := range checks {
            if !healthy {
                allHealthy = false
                break
            }
        }
        
        status := 200
        if !allHealthy {
            status = 503
        }
        
        return c.JSON(status, map[string]interface{}{
            "status": allHealthy,
            "checks": checks,
        })
    })
}
```

**Por qué dos endpoints**:
- **Liveness**: Kubernetes usa esto para saber si debe reiniciar el pod
- **Readiness**: Kubernetes usa esto para saber si debe enviar tráfico

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
│   ├── shared/                          # Compartido entre módulos
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
│   │   │   │   ├── proto/devices.proto
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
│   ├── modules/                         # Módulos de negocio (DDD)
│   │   │
│   │   ├── auth/                        # Módulo de autenticación
│   │   │   ├── auth.module.ts
│   │   │   │
│   │   │   ├── domain/                  # Capa de dominio
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
│   │   │   ├── application/             # Casos de uso
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
│   │   │   └── infrastructure/          # Adaptadores
│   │   │       ├── persistence/
│   │   │       │   ├── prisma/
│   │   │       │   │   ├── user.repository.ts
│   │   │       │   │   └── api-key.repository.ts
│   │   │       │   └── mappers/
│   │   │       │       ├── user.mapper.ts
│   │   │       │       └── api-key.mapper.ts
│   │   │       ├── http/
│   │   │       │   ├── graphql/
│   │   │       │   │   ├── resolvers/
│   │   │       │   │   │   └── auth.resolver.ts
│   │   │       │   │   └── types/
│   │   │       │   │       └── auth.types.ts
│   │   │       │   └── rest/
│   │   │       │       └── controllers/
│   │   │       │           └── auth.controller.ts
│   │   │       └── strategies/
│   │   │           ├── jwt.strategy.ts
│   │   │           └── api-key.strategy.ts
│   │   │
│   │   ├── iot-core/                    # Módulo IoT/Core
│   │   │   ├── iot-core.module.ts
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
│   │   │   ├── application/
│   │   │   │   ├── use-cases/
│   │   │   │   │   ├── organizations/
│   │   │   │   │   │   ├── create-organization.use-case.ts
│   │   │   │   │   │   ├── list-organizations.use-case.ts
│   │   │   │   │   │   └── *.dto.ts
│   │   │   │   │   ├── establishments/
│   │   │   │   │   │   ├── create-establishment.use-case.ts
│   │   │   │   │   │   ├── assign-user.use-case.ts
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
│   │   │   └── infrastructure/
│   │   │       ├── persistence/
│   │   │       │   ├── prisma/
│   │   │       │   │   ├── organization.repository.ts
│   │   │       │   │   ├── establishment.repository.ts
│   │   │       │   │   ├── device.repository.ts
│   │   │       │   │   └── role.repository.ts
│   │   │       │   └── mappers/
│   │   │       ├── http/
│   │   │       │   ├── graphql/
│   │   │       │   │   ├── resolvers/
│   │   │       │   │   │   ├── organization.resolver.ts
│   │   │       │   │   │   ├── establishment.resolver.ts
│   │   │       │   │   │   ├── device.resolver.ts
│   │   │       │   │   │   └── role.resolver.ts
│   │   │       │   │   └── types/
│   │   │       │   └── rest/
│   │   │       │       └── controllers/
│   │   │       └── grpc/
│   │   │           └── device-commands.client.ts
│   │   │
│   │   ├── booking/                     # Módulo de reservas
│   │   │   ├── booking.module.ts
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
│   │   │   ├── application/
│   │   │   │   └── use-cases/
│   │   │   │       ├── create-booking.use-case.ts
│   │   │   │       ├── cancel-booking.use-case.ts
│   │   │   │       ├── confirm-booking.use-case.ts
│   │   │   │       ├── check-availability.use-case.ts
│   │   │   │       └── *.dto.ts
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
│   │   ├── smart-access/                # A desarrollar
│   │   │   ├── smart-access.module.ts
│   │   │   ├── domain/
│   │   │   ├── application/
│   │   │   └── infrastructure/
│   │   │
│   │   └── universities/                # A desarrollar
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
│   ├── unit/                            # Tests unitarios
│   │   ├── auth/
│   │   ├── iot-core/
│   │   └── booking/
│   ├── integration/                     # Tests de integración
│   │   └── *.spec.ts
│   └── e2e/                             # Tests end-to-end
│       └── *.e2e-spec.ts
│
├── prisma/
│   ├── schema.prisma
│   ├── migrations/
│   └── seed.ts
│
├── proto/                               # Protocol Buffers
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

**Explicación de la estructura**:

**`shared/`**: Código reutilizable entre módulos
- `infrastructure/`: Servicios de infraestructura (DB, cache, gRPC)
- `domain/`: Value objects y excepciones comunes
- `application/`: Guards, interceptors, pipes globales
- `utils/`: Funciones utilitarias

**`modules/`**: Cada módulo sigue arquitectura hexagonal
- `domain/`: Lógica de negocio pura (sin dependencias)
- `application/`: Casos de uso (orquesta el dominio)
- `infrastructure/`: Adaptadores (DB, HTTP, gRPC)

**Por qué esta organización**:
- **Separación por feature**: Cada módulo es autocontenido
- **Fácil de testear**: Domain layer sin dependencias
- **Escalable**: Módulos pueden extraerse a microservicios
- **Mantenible**: Cambios en un módulo no afectan otros

### 9.2 Devices Service (Golang)

```
hse-devices-service/
│
├── cmd/
│   └── server/
│       └── main.go                      # Entry point
│
├── internal/                            # Código privado
│   │
│   ├── domain/                          # Capa de dominio
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
│   │   └── repositories/                # Interfaces (puertos)
│   │       ├── device_repository.go
│   │       ├── event_repository.go
│   │       └── query_repository.go
│   │
│   ├── application/                     # Capa de aplicación
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
│   │       ├── connection_manager.go    # Pool de conexiones
│   │       ├── event_queue.go           # Cola Redis
│   │       ├── event_processor.go       # Batch processor
│   │       ├── cache_manager.go         # Gestión de cache
│   │       └── dlq_manager.go           # Dead Letter Queue
│   │
│   ├── infrastructure/                  # Capa de infraestructura
│   │   │
│   │   ├── persistence/
│   │   │   ├── postgres/
│   │   │   │   ├── timescale/
│   │   │   │   │   ├── event_writer.go      # Batch writes
│   │   │   │   │   ├── event_reader.go      # Queries
│   │   │   │   │   └── aggregates.go        # Continuous aggregates
│   │   │   │   ├── device_repository.go
│   │   │   │   └── gorm_config.go
│   │   │   └── redis/
│   │   │       ├── event_queue.go           # Cola de eventos
│   │   │       ├── device_cache.go          # Cache de estados
│   │   │       └── connection_state.go      # Estados de conexión
│   │   │
│   │   ├── protocols/                   # Adaptadores de protocolos
│   │   │   ├── factory.go               # Abstract Factory
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
│   │       │   └── stream_manager.go
│   │       ├── client/
│   │       │   └── monolith_client.go   # Con circuit breaker
│   │       └── pb/                       # Código generado
│   │           └── *.pb.go
│   │
│   └── config/
│       ├── config.go
│       └── env.go
│
├── pkg/                                 # Código público/reutilizable
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

**Explicación de la estructura**:

**`internal/`**: Código privado del servicio
- **`domain/`**: Entidades y lógica de negocio pura
- **`application/`**: Separación CQRS (commands/queries) + services
- **`infrastructure/`**: Adaptadores (DB, protocols, HTTP, gRPC)

**`pkg/`**: Código que podría ser reutilizado por otros servicios
- Logger, utils, errors comunes

**Por qué esta organización**:
- **CQRS explícito**: Commands y queries claramente separados
- **Clean architecture**: Dependencias apuntan hacia adentro
- **Testeable**: Domain sin dependencias externas
- **Escalable**: Fácil añadir nuevos protocolos o handlers

### 9.3 Migraciones de Base de Datos

**TimescaleDB Setup** (`migrations/002_timescale_setup.sql`):
```sql
-- Activar extensión TimescaleDB
CREATE EXTENSION IF NOT EXISTS timescaledb CASCADE;

-- Crear tabla de eventos
CREATE TABLE device_events (
  time          TIMESTAMPTZ NOT NULL,
  device_id     UUID NOT NULL,
  event_type    VARCHAR(50) NOT NULL,
  value         NUMERIC,
  unit          VARCHAR(20),
  metadata      JSONB,
  source        VARCHAR(50)
);

-- Convertir a hypertable
SELECT create_hypertable('device_events', 'time');

-- Índices optimizados
CREATE INDEX idx_device_events_device_time 
  ON device_events (device_id, time DESC);

CREATE INDEX idx_device_events_type 
  ON device_events (event_type, time DESC);

-- Compresión automática (datos > 7 días)
ALTER TABLE device_events SET (
  timescaledb.compress,
  timescaledb.compress_segmentby = 'device_id'
);

SELECT add_compression_policy('device_events', INTERVAL '7 days');

-- Retención automática (eliminar datos > 2 años)
SELECT add_retention_policy('device_events', INTERVAL '2 years');
```

**Continuous Aggregates** (`migrations/003_continuous_aggregates.sql`):
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

-- Política de refresh (cada 10 minutos)
SELECT add_continuous_aggregate_policy(
  'device_events_hourly',
  start_offset => INTERVAL '3 hours',
  end_offset => INTERVAL '1 hour',
  schedule_interval => INTERVAL '10 minutes'
);

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

-- Política de refresh (cada hora)
SELECT add_continuous_aggregate_policy(
  'device_events_daily',
  start_offset => INTERVAL '3 days',
  end_offset => INTERVAL '1 day',
  schedule_interval => INTERVAL '1 hour'
);
```

---

**📄 Continúa en**: `10-11_Infraestructura_Migracion.md`