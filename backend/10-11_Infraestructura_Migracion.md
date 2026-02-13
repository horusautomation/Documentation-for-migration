# Plan de Migración HSE Backend - Secciones 10-11

**Continuación desde**: `07-09_CQRS_Patrones_Estructura.md`

---

## 10. Infraestructura Cloud (AWS)

### 10.1 Arquitectura en AWS

```
                    INTERNET
                       │
                       ▼
               ┌───────────────┐
               │  CloudFront   │ CDN + DDoS protection
               │    (CDN)      │
               └───────┬───────┘
                       │
                       ▼
               ┌───────────────┐
               │      ALB      │ Application Load Balancer
               │ (Layer 7 LB)  │ SSL Termination
               └───────┬───────┘
                       │
            ┌──────────┼──────────┐
            │                     │
            ▼                     ▼
    ┌──────────────┐      ┌──────────────┐
    │ ECS Fargate  │      │ ECS Fargate  │
    │  (Monolith)  │◄────►│  (Devices)   │
    │ Auto-scaling │ gRPC │ Auto-scaling │
    │  2-10 tasks  │      │  2-8 tasks   │
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
│   Multi-AZ   │      │    Redis     │
│+ TimescaleDB │      │(Cluster Mode)│
└──────────────┘      └──────────────┘
        │                     │
        └──────────┬──────────┘
                   │
                   ▼
           ┌───────────────┐
           │  CloudWatch   │ Logs + Metrics
           │+ New Relic    │ APM + Traces
           └───────────────┘
                   │
                   ▼
           ┌───────────────┐
           │    Secrets    │ Credentials
           │   Manager     │ + Rotation
           └───────────────┘
```

### 10.2 Servicios AWS Detallados

#### 10.2.1 ECS Fargate (Compute)

**Monolito (NestJS)**:
```yaml
Service: hse-monolith
Cluster: hse-cluster

Task Definition:
  CPU: 1024 (1 vCPU)
  Memory: 2048 MB (2 GB)
  Container:
    Image: <account>.dkr.ecr.us-east-1.amazonaws.com/hse-monolith:latest
    Port: 8080
    Environment:
      - NODE_ENV=production
      - NEW_RELIC_LICENSE_KEY=from-secrets-manager
    HealthCheck:
      Command: ["CMD-SHELL", "curl -f http://localhost:8080/health/readiness || exit 1"]
      Interval: 30s
      Timeout: 5s
      Retries: 3

Service Config:
  Desired Count: 2
  Min Capacity: 2
  Max Capacity: 10
  
  Auto-scaling:
    Target CPU: 70%
    Target Memory: 80%
    Scale-out cooldown: 60s
    Scale-in cooldown: 300s
  
  Load Balancer:
    Target Group: hse-monolith-tg
    Health Check Path: /health/readiness
    Healthy Threshold: 2
    Unhealthy Threshold: 3
```

**Devices Service (Go)**:
```yaml
Service: hse-devices
Cluster: hse-cluster

Task Definition:
  CPU: 2048 (2 vCPU)
  Memory: 4096 MB (4 GB)
  Container:
    Image: <account>.dkr.ecr.us-east-1.amazonaws.com/hse-devices:latest
    Ports:
      - 8080 (HTTP)
      - 50051 (gRPC)
      - 5000 (TCP devices)
    Environment:
      - ENVIRONMENT=production
    HealthCheck:
      Command: ["CMD-SHELL", "curl -f http://localhost:8080/health/readiness || exit 1"]
      Interval: 30s
      Timeout: 5s
      Retries: 3

Service Config:
  Desired Count: 2
  Min Capacity: 2
  Max Capacity: 8
  
  Auto-scaling:
    Target Metric: Active Connections
    Target Value: 1000 connections
    Scale-out cooldown: 60s
    Scale-in cooldown: 300s
```

**Por qué Fargate y no EC2**:
- Sin gestión de servers (serverless containers)
- Auto-scaling automático
- Pago por uso (solo por recursos usados)
- Security patches automáticos
- Menos complejidad operacional

#### 10.2.2 RDS PostgreSQL + TimescaleDB

**Configuración Staging**:
```yaml
Instance Class: db.t3.medium
  vCPU: 2
  RAM: 4 GB
  
Storage:
  Type: gp3 (SSD)
  Size: 100 GB
  IOPS: 3000
  Auto-scaling: Enabled (up to 500 GB)

Multi-AZ: No (costo reducido)
Backup Retention: 7 días
```

**Configuración Production**:
```yaml
Instance Class: db.r6g.xlarge
  vCPU: 4
  RAM: 32 GB
  
Storage:
  Type: gp3 (SSD)
  Size: 500 GB
  IOPS: 12000
  Auto-scaling: Enabled (up to 1 TB)

Multi-AZ: Yes (alta disponibilidad)
  Primary: us-east-1a
  Standby: us-east-1b
  Failover: Automático (< 60 segundos)

Backups:
  Automated: Daily (retención 30 días)
  Point-in-time recovery: Enabled
  Snapshots: Manual antes de migraciones

Security:
  Encryption at rest: AES-256
  Encryption in transit: SSL/TLS
  
Parameter Group:
  shared_preload_libraries: 'timescaledb'
  max_connections: 200
  work_mem: 16MB
```

**Instalación de TimescaleDB**:
```sql
-- 1. Conectar como superuser
psql -h your-rds-endpoint.rds.amazonaws.com -U postgres -d hse_production

-- 2. Crear extensión (requiere rds_superuser role)
CREATE EXTENSION IF NOT EXISTS timescaledb CASCADE;

-- 3. Verificar instalación
SELECT extname, extversion FROM pg_extension WHERE extname = 'timescaledb';

-- 4. Configurar continuous aggregates refresh
SELECT add_continuous_aggregate_policy('device_events_hourly',
  start_offset => INTERVAL '3 hours',
  end_offset => INTERVAL '1 hour',
  schedule_interval => INTERVAL '10 minutes'
);
```

#### 10.2.3 ElastiCache Redis

**Configuración Production**:
```yaml
Cluster Mode: Enabled
Node Type: cache.r6g.large
  vCPU: 2
  RAM: 13.07 GB

Topology:
  Shards: 2
  Replicas per shard: 2
  Total nodes: 6 (2 primary + 4 replicas)
  Total capacity: ~78 GB

Configuration:
  Engine Version: 7.0
  Port: 6379
  
Security:
  Encryption at rest: Yes
  Encryption in transit: Yes
  Auth token: Enabled
  
  Security Group:
    Inbound: Port 6379 from ECS-SG only

Automatic Failover: Enabled
Backup:
  Daily snapshot
  Retention: 7 días

Maintenance Window: sun:03:00-sun:04:00
```

**Uso de Redis**:
```
Shard 1 (Queue + DLQ):
├─ events:queue         → Cola de eventos (LPUSH/BRPOP)
├─ events:dlq           → Dead Letter Queue
└─ events:dlq:permanent → DLQ permanente

Shard 2 (Cache):
├─ device:state:*       → Estados actuales (TTL: 1 min)
├─ device:agg:*         → Agregaciones (TTL: 5 min)
└─ session:*            → Sesiones de usuarios
```

#### 10.2.4 Application Load Balancer (ALB)

```yaml
Scheme: Internet-facing
IP Address Type: ipv4

Listeners:
  - Port: 443 (HTTPS)
    Protocol: HTTPS
    SSL Certificate: ACM Certificate
    Default Action: Forward to hse-monolith-tg
    
  - Port: 80 (HTTP)
    Protocol: HTTP
    Default Action: Redirect to HTTPS

Target Groups:
  hse-monolith-tg:
    Protocol: HTTP
    Port: 8080
    Health Check:
      Path: /health/readiness
      Interval: 30s
      Timeout: 5s
      Healthy Threshold: 2
      Unhealthy Threshold: 3
      Matcher: 200
    
  hse-devices-tg:
    Protocol: HTTP
    Port: 8080
    Health Check:
      Path: /health/readiness
      Interval: 30s
      Timeout: 5s
      Healthy Threshold: 2
      Unhealthy Threshold: 3
      Matcher: 200

Security Groups:
  ALB-SG:
    Inbound:
      - Port 443 from 0.0.0.0/0
      - Port 80 from 0.0.0.0/0
    Outbound:
      - Port 8080 to ECS-SG

Attributes:
  Idle timeout: 60 seconds
  Cross-zone load balancing: Enabled
  Access logs: Enabled (S3 bucket)
```

#### 10.2.5 VPC y Networking

```yaml
VPC:
  CIDR: 10.0.0.0/16
  DNS Hostnames: Enabled
  DNS Resolution: Enabled

Subnets:
  Public Subnets (ALB):
    - public-subnet-1a: 10.0.1.0/24 (us-east-1a)
    - public-subnet-1b: 10.0.2.0/24 (us-east-1b)
    
  Private Subnets (ECS):
    - private-subnet-1a: 10.0.10.0/24 (us-east-1a)
    - private-subnet-1b: 10.0.11.0/24 (us-east-1b)
    
  Database Subnets:
    - db-subnet-1a: 10.0.20.0/24 (us-east-1a)
    - db-subnet-1b: 10.0.21.0/24 (us-east-1b)

Internet Gateway:
  Attached to VPC
  Routes: Public subnets → IGW

NAT Gateway:
  Location: public-subnet-1a
  Elastic IP: Allocated
  Routes: Private subnets → NAT

Security Groups:
  ALB-SG:
    Inbound: 443, 80 from 0.0.0.0/0
    Outbound: 8080 to ECS-SG
  
  ECS-SG:
    Inbound: 8080 from ALB-SG, 50051 from ECS-SG
    Outbound: 5432 to DB-SG, 6379 to Redis-SG, All to 0.0.0.0/0 (Internet)
  
  DB-SG:
    Inbound: 5432 from ECS-SG
    Outbound: None
  
  Redis-SG:
    Inbound: 6379 from ECS-SG
    Outbound: None
```

#### 10.2.6 Secrets Manager

```json
{
  "database": {
    "host": "hse-prod.c9dnexample.us-east-1.rds.amazonaws.com",
    "port": 5432,
    "username": "hse_admin",
    "password": "auto-generated-secure-password",
    "database": "hse_production"
  },
  "redis": {
    "host": "hse-prod.cache.amazonaws.com",
    "port": 6379,
    "password": "auto-generated-auth-token"
  },
  "jwt": {
    "secret": "auto-generated-jwt-secret",
    "expiresIn": "15m",
    "refreshExpiresIn": "7d"
  },
  "newrelic": {
    "licenseKey": "your-newrelic-license-key",
    "appName": "HSE Backend Production"
  },
  "devices_grpc": {
    "url": "hse-devices.local:50051"
  }
}
```

**Rotación automática**:
```yaml
Rotation Policy:
  Enabled: Yes
  Rotation Interval: 30 días
  Lambda Function: auto-rotation-function
```

### 10.3 CI/CD Pipeline (GitHub Actions)

**Workflow completo** (`.github/workflows/deploy.yml`):
```yaml
name: Deploy to AWS

on:
  push:
    branches: [main, develop]

env:
  AWS_REGION: us-east-1
  ENVIRONMENT: ${{ github.ref == 'refs/heads/main' && 'production' || 'staging' }}

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Test Monolith
        run: |
          cd hse-backend-monolith
          npm ci
          npm run test
          npm run test:e2e
      
      - name: Test Devices
        run: |
          cd hse-devices-service
          go test ./... -v

  build-and-push:
    needs: test
    runs-on: ubuntu-latest
    outputs:
      monolith-image: ${{ steps.meta-monolith.outputs.tags }}
      devices-image: ${{ steps.meta-devices.outputs.tags }}
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}
      
      - name: Login to ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v1
      
      - name: Build and push Monolith
        uses: docker/build-push-action@v4
        with:
          context: ./hse-backend-monolith
          push: true
          tags: |
            ${{ steps.login-ecr.outputs.registry }}/hse-monolith:${{ github.sha }}
            ${{ steps.login-ecr.outputs.registry }}/hse-monolith:latest
      
      - name: Build and push Devices
        uses: docker/build-push-action@v4
        with:
          context: ./hse-devices-service
          push: true
          tags: |
            ${{ steps.login-ecr.outputs.registry }}/hse-devices:${{ github.sha }}
            ${{ steps.login-ecr.outputs.registry }}/hse-devices:latest

  deploy:
    needs: build-and-push
    runs-on: ubuntu-latest
    steps:
      - name: Deploy Monolith to ECS
        run: |
          aws ecs update-service \
            --cluster hse-cluster \
            --service hse-monolith-${{ env.ENVIRONMENT }} \
            --force-new-deployment \
            --region ${{ env.AWS_REGION }}
      
      - name: Deploy Devices to ECS
        run: |
          aws ecs update-service \
            --cluster hse-cluster \
            --service hse-devices-${{ env.ENVIRONMENT }} \
            --force-new-deployment \
            --region ${{ env.AWS_REGION }}
      
      - name: Wait for deployment
        run: |
          aws ecs wait services-stable \
            --cluster hse-cluster \
            --services hse-monolith-${{ env.ENVIRONMENT }} hse-devices-${{ env.ENVIRONMENT }} \
            --region ${{ env.AWS_REGION }}
      
      - name: Verify health
        run: |
          curl -f https://api-${{ env.ENVIRONMENT }}.hse.com/health/readiness || exit 1

  notify:
    needs: [test, build-and-push, deploy]
    if: always()
    runs-on: ubuntu-latest
    steps:
      - name: Notify team
        run: |
          # Send Slack notification
          curl -X POST ${{ secrets.SLACK_WEBHOOK_URL }} \
            -H 'Content-Type: application/json' \
            -d '{
              "text": "Deploy ${{ needs.deploy.result }} - ${{ env.ENVIRONMENT }}",
              "attachments": [{
                "color": "${{ needs.deploy.result == 'success' && 'good' || 'danger' }}",
                "fields": [{
                  "title": "Environment",
                  "value": "${{ env.ENVIRONMENT }}",
                  "short": true
                }, {
                  "title": "Commit",
                  "value": "${{ github.sha }}",
                  "short": true
                }]
              }]
            }'
```

### 10.4 Costos Estimados Mensuales

#### Staging Environment
```
ECS Fargate:
├─ Monolith: 1 task × 1 vCPU × 2 GB × 730h = $40
└─ Devices: 1 task × 2 vCPU × 4 GB × 730h = $75

RDS PostgreSQL:
└─ db.t3.medium (100 GB storage) = $120

ElastiCache Redis:
└─ cache.t3.small (1 node) = $50

ALB:
└─ Load Balancer + Data Transfer = $25

Data Transfer:
└─ Outbound (estimado) = $20

CloudWatch Logs:
└─ Ingestion + Storage = $15

Secrets Manager:
└─ 5 secrets = $2

──────────────────────────────
TOTAL STAGING: ~$347/mes
```

#### Production Environment
```
ECS Fargate (promedio 4 tasks cada uno):
├─ Monolith: 4 × 1 vCPU × 2 GB × 730h = $160
└─ Devices: 3 × 2 vCPU × 4 GB × 730h = $225

RDS PostgreSQL:
└─ db.r6g.xlarge Multi-AZ (500 GB) = $850

ElastiCache Redis:
└─ cache.r6g.large cluster (6 nodes) = $450

ALB:
└─ Load Balancer + Data Transfer = $60

Data Transfer:
└─ Outbound (estimado 500 GB/mes) = $45

CloudWatch Logs:
└─ Ingestion (50 GB) + Storage = $30

New Relic:
└─ Pro plan (100 GB/mes) = $99

Secrets Manager:
└─ 10 secrets + rotations = $5

Backups (RDS snapshots):
└─ Snapshots storage = $20

──────────────────────────────
TOTAL PRODUCTION: ~$1,944/mes
```

**Optimizaciones de costos**:
- Reserved Instances para RDS (ahorro 30-40%)
- Savings Plans para Fargate (ahorro 20%)
- Lifecycle policies para logs (retención 30 días)
- S3 Intelligent-Tiering para backups

---

## 11. Estrategia de Migración (Strangler Fig Pattern)

### 11.1 Principios Fundamentales

**Strangler Fig Pattern**: Migración incremental que "estrangula" gradualmente al sistema viejo.

```
┌────────────────────────────────────────────────┐
│              Sistema Viejo                      │
│  (Monolito actual con todas las features)      │
└────────────────────────────────────────────────┘
                     │
                     ▼
┌────────────────────────────────────────────────┐
│         Fase 1: Coexistencia                    │
│   Viejo (100%) + Nuevo (shadow mode)           │
│   │                                             │
│   ├─ Nuevo recibe copia de eventos             │
│   └─ Comparación de resultados                 │
└────────────────────────────────────────────────┘
                     │
                     ▼
┌────────────────────────────────────────────────┐
│         Fase 2: Traffic Split                   │
│   Viejo (90%) + Nuevo (10%)                    │
│   │                                             │
│   ├─ Routing por feature flags                 │
│   ├─ Monitoreo intensivo                       │
│   └─ Rollback rápido si hay issues             │
└────────────────────────────────────────────────┘
                     │
                     ▼
┌────────────────────────────────────────────────┐
│         Fase 3: Incremento Gradual              │
│   25% → 50% → 75% → 100%                       │
│   │                                             │
│   └─ Validación continua de métricas           │
└────────────────────────────────────────────────┘
                     │
                     ▼
┌────────────────────────────────────────────────┐
│              Sistema Nuevo                      │
│   (100% de tráfico) + Viejo en standby        │
└────────────────────────────────────────────────┘
```

**Por qué este enfoque**:
- **Riesgo minimizado**: Podemos hacer rollback en cualquier momento
- **Validación continua**: Comparamos comportamiento entre sistemas
- **Sin downtime**: Ambos sistemas corren en paralelo
- **Aprendizaje iterativo**: Cada fase nos da feedback

### 11.2 Fases Detalladas

#### Fase 0: Preparación (Semanas 1-2)

**Objetivo**: Infraestructura lista y operacional.

**Tareas**:
```
□ Infraestructura AWS
  ├─ Crear VPC con subnets
  ├─ Provisionar RDS PostgreSQL
  ├─ Instalar TimescaleDB extension
  ├─ Provisionar ElastiCache Redis
  ├─ Configurar Security Groups
  └─ Setup ALB

□ CI/CD
  ├─ Configurar GitHub Actions
  ├─ Setup ECR repositories
  ├─ Test pipeline en staging
  └─ Documentar proceso de deploy

□ Monitoring
  ├─ Configurar New Relic
  ├─ Setup CloudWatch dashboards
  ├─ Configurar alertas críticas
  └─ Test de notificaciones

□ Secrets
  ├─ Migrar credenciales a Secrets Manager
  ├─ Configurar rotación automática
  └─ Test de acceso desde ECS
```

**Criterios de éxito**:
- ✅ Deploy automático funciona end-to-end
- ✅ Health checks responden correctamente
- ✅ Logs llegan a CloudWatch y New Relic
- ✅ Alertas se disparan correctamente

#### Fase 1: Devices Service en Shadow Mode (Semanas 3-4)

**Objetivo**: Devices Service corriendo en producción pero sin servir tráfico real.

**Estrategia**:
```
Evento de dispositivo
    │
    ├──> Sistema Viejo (guarda en su DB)
    │
    └──> Sistema Nuevo (guarda en TimescaleDB)
         └─ Comparación automática cada hora
```

**Tareas**:
```
□ Deploy Devices Service
  ├─ Deploy a staging
  ├─ Tests de carga (simular 200 eventos/seg)
  ├─ Deploy a production
  └─ Verificar health checks

□ Dual Write Mode
  ├─ Configurar webhook en sistema viejo
  ├─ Enviar copia de eventos a Devices Service
  ├─ NO usar datos del nuevo sistema aún
  └─ Solo guardar y comparar

□ Script de Validación
  ├─ Comparar conteo de eventos por hora
  ├─ Comparar valores de eventos
  ├─ Detectar discrepancias
  └─ Alertar si drift > 1%

□ Monitoreo
  ├─ Dashboard de comparación
  ├─ Latencia de escritura (debe ser < 10ms)
  ├─ Batch processing lag (debe ser < 5 seg)
  └─ DLQ size (debe ser ~0)
```

**Script de validación**:
```bash
#!/bin/bash
# scripts/validate-shadow-mode.sh

START_DATE="2024-01-04 00:00:00"
END_DATE="2024-01-04 23:59:59"

# Contar eventos en sistema viejo
OLD_COUNT=$(psql $OLD_DB -t -c "
  SELECT COUNT(*) FROM device_events 
  WHERE time BETWEEN '$START_DATE' AND '$END_DATE'
")

# Contar eventos en sistema nuevo
NEW_COUNT=$(psql $NEW_DB -t -c "
  SELECT COUNT(*) FROM device_events 
  WHERE time BETWEEN '$START_DATE' AND '$END_DATE'
")

# Calcular drift
DRIFT=$(echo "scale=2; (1 - $NEW_COUNT / $OLD_COUNT) * 100" | bc)

echo "Old system: $OLD_COUNT events"
echo "New system: $NEW_COUNT events"
echo "Drift: $DRIFT%"

if (( $(echo "$DRIFT > 1" | bc -l) )); then
  echo "❌ ALERT: Drift > 1%"
  exit 1
else
  echo "✅ OK: Systems in sync"
fi
```

**Criterios de éxito**:
- ✅ 99.9%+ de eventos coinciden entre sistemas
- ✅ Latencia de escritura < 10ms
- ✅ Batch processing lag < 5 segundos
- ✅ DLQ vacía o casi vacía
- ✅ Sin errores en logs por 7 días consecutivos

#### Fase 2: Migración Módulo IoT/Core (Semanas 5-7)

**Objetivo**: Migrar lógica de negocio del módulo IoT/Core con traffic split gradual.

**Semana 5: Traffic Split 10%**
```
Feature Flag: traffic_split_iot_core = 10

Implementación:
- Hash del user ID determina routing
- 10% de usuarios → Sistema nuevo
- 90% de usuarios → Sistema viejo
- Ambos sistemas registran métricas
```

**Implementación de Traffic Split**:
```typescript
// src/shared/application/interceptors/traffic-split.interceptor.ts
@Injectable()
export class TrafficSplitInterceptor implements NestInterceptor {
  constructor(
    private configService: ConfigService,
    private metricsService: MetricsService,
  ) {}

  async intercept(context: ExecutionContext, next: CallHandler) {
    const request = context.switchToHttp().getRequest();
    const userId = request.user?.id;
    
    // Obtener porcentaje de traffic para nuevo sistema
    const splitPercent = await this.configService.get('TRAFFIC_SPLIT_PERCENT');
    
    // Determinar si usar nuevo sistema
    const useNewSystem = this.shouldUseNewSystem(userId, splitPercent);
    
    if (useNewSystem) {
      request.useNewSystem = true;
      this.metricsService.increment('traffic_split.new_system');
    } else {
      request.useNewSystem = false;
      this.metricsService.increment('traffic_split.old_system');
    }
    
    const startTime = Date.now();
    
    try {
      const result = await next.handle().toPromise();
      
      const duration = Date.now() - startTime;
      this.metricsService.histogram(
        useNewSystem ? 'latency.new_system' : 'latency.old_system',
        duration
      );
      
      return result;
    } catch (error) {
      this.metricsService.increment(
        useNewSystem ? 'errors.new_system' : 'errors.old_system'
      );
      throw error;
    }
  }

  private shouldUseNewSystem(userId: string, percent: number): boolean {
    if (!userId) return false;
    
    // Hash consistente basado en user ID
    const hash = crypto.createHash('md5').update(userId).digest('hex');
    const hashInt = parseInt(hash.substring(0, 8), 16);
    
    return (hashInt % 100) < percent;
  }
}
```

**Monitoreo durante traffic split**:
```
Dashboards críticos:
├─ Error rate (nuevo vs viejo)
├─ Latency p50, p95, p99 (nuevo vs viejo)
├─ Throughput (requests/min)
├─ Discrepancias en respuestas
└─ User complaints (support tickets)

Alertas configuradas:
├─ Error rate nuevo > error rate viejo + 0.5%
├─ Latency p99 nuevo > latency p99 viejo × 1.5
└─ Discrepancy rate > 1%
```

**Semana 6: Incremento a 50%**

Si métricas son buenas en semana 5:
```
Feature Flag: traffic_split_iot_core = 50

Validación:
□ Error rate nuevo ≈ error rate viejo
□ Latency nuevo ≤ latency viejo
□ Sin incremento en support tickets
□ Feedback de usuarios positivo/neutral
```

**Semana 7: Incremento a 100%**

Si semana 6 es exitosa:
```
Feature Flag: traffic_split_iot_core = 100

Todos los usuarios → Sistema nuevo
Sistema viejo en standby (ready para rollback)
```

**Plan de Rollback**:
```bash
# Si hay problemas CRÍTICOS en cualquier momento:

# 1. Reducir tráfico inmediatamente (< 30 segundos)
aws ssm put-parameter \
  --name /hse/traffic-split-percent \
  --value "0" \
  --overwrite

# 2. Verificar que tráfico vuelve a sistema viejo
curl https://api.hse.com/health

# 3. Investigar issue en nuevo sistema sin presión

# 4. Comunicar a stakeholders
```

**Criterios de éxito Fase 2**:
- ✅ Error rate < 0.1%
- ✅ Latency p99 < 500ms
- ✅ 7 días de operación estable al 100%
- ✅ Sin incidentes críticos
- ✅ Feedback positivo de usuarios

#### Fase 3: Migración Módulo Booking (Semanas 8-9)

**Objetivo**: Migrar sistema de reservas.

**Estrategia**: Similar a IoT/Core pero con validaciones adicionales.

**Semana 8: Traffic Split 10-50%**
```
Validaciones específicas:
├─ Sin reservas duplicadas
├─ Sin solapamientos inválidos
├─ Códigos de acceso funcionando
├─ Notificaciones enviadas correctamente
└─ Transiciones de estado correctas
```

**Datos históricos** (opcional):
```sql
-- Migrar solo reservas activas y futuras
-- Reservas pasadas (completed) no son necesarias

INSERT INTO new_system.bookings
SELECT * FROM old_system.bookings
WHERE status IN ('pending', 'confirmed', 'in-progress')
   OR check_out_date >= CURRENT_DATE;

-- Validar integridad
SELECT 
  COUNT(*) as total,
  status,
  COUNT(DISTINCT guest_id) as unique_guests
FROM new_system.bookings
GROUP BY status;
```

**Semana 9: Traffic Split 100%**

**Criterios de éxito**:
- ✅ Sin reservas duplicadas en 7 días
- ✅ Códigos de acceso 100% funcionales
- ✅ Sin quejas de huéspedes
- ✅ Todas las notificaciones enviadas

#### Fase 4: Apagar Sistema Viejo (Semana 10+)

**Objetivo**: Eliminar sistema viejo completamente.

**Precondiciones**:
```
□ 2 semanas de operación estable al 100%
□ Sin incidentes críticos
□ Feedback positivo de usuarios
□ Métricas dentro de rangos esperados
□ Equipo confiado en nuevo sistema
```

**Proceso**:
```
Día 1-7: Monitoreo intensivo
├─ On-call 24/7
├─ Revisión diaria de métricas
└─ Reunión diaria de status

Día 8: Backup completo sistema viejo
├─ Snapshot de base de datos
├─ Export de configuraciones
└─ Documentar procedimiento de rollback

Día 9: Sistema viejo en read-only
├─ Deshabilitar escrituras
├─ Mantener disponible para consultas
└─ Monitorear si se usa

Día 10-16: Validación final
├─ Confirmar que nadie usa sistema viejo
├─ Validar integridad de datos
└─ Aprobación de stakeholders

Día 17: Apagar sistema viejo
├─ Stop de servicios
├─ Mantener backups por 30 días
└─ Documentar apagado
```

**Rollback de emergencia** (si es absolutamente necesario):
```bash
#!/bin/bash
# scripts/emergency-rollback.sh

echo "🚨 EMERGENCY ROLLBACK 🚨"

# 1. Reactivar sistema viejo
aws ecs update-service \
  --cluster hse-cluster \
  --service hse-old-system \
  --desired-count 4

# 2. Traffic split a 0
aws ssm put-parameter \
  --name /hse/use-new-system \
  --value "false" \
  --overwrite

# 3. Esperar a que sistema viejo esté ready
aws ecs wait services-stable \
  --cluster hse-cluster \
  --services hse-old-system

# 4. Verificar salud
curl -f https://api-old.hse.com/health || exit 1

echo "✅ Rollback completado"
echo "⚠️  Investigar issue en nuevo sistema"
```

### 11.3 Checklist de Migración

#### Pre-Migración
```
□ Backup completo del sistema viejo
□ Plan de rollback documentado y probado
□ Feature flags configurados y testeados
□ Monitoring dashboards listos
□ Alertas configuradas en New Relic
□ On-call rotation definida
□ Comunicación a stakeholders programada
□ Ventana de mantenimiento (si es necesaria)
□ Runbook de incidentes actualizado
```

#### Durante Migración
```
□ Monitoreo activo 24/7
□ Daily standup con equipo técnico
□ Comparación automática de respuestas
□ Logs centralizados y accesibles
□ Métricas en tiempo real visibles
□ Canal de comunicación de emergencia abierto
□ Feedback de usuarios monitoreado
□ Traffic split ajustable en tiempo real
```

#### Post-Migración
```
□ Análisis de métricas vs baseline
□ Documento de lecciones aprendidas
□ Identificar optimizaciones
□ Documentar deuda técnica
□ Celebrar con el equipo 🎉
□ Comunicar éxito a stakeholders
□ Planear próximas mejoras
```

### 11.4 Métricas de Éxito

**Métricas técnicas**:
```
Uptime:
Target: > 99.9%
Current: (se mide post-migración)

Error Rate:
Target: < 0.1%
Baseline viejo: 0.15%
Current: (se mide post-migración)

Response Time (p99):
Target: < 500ms
Baseline viejo: 650ms
Current: (se mide post-migración)

Eventos Procesados:
Target: 100+ eventos/segundo sin lag
Lag aceptable: < 5 segundos
Current: (se mide post-migración)

Data Loss:
Target: 0 eventos perdidos
Método: Validación automática cada hora
Current: (se mide post-migración)

Cache Hit Rate:
Target: > 80%
Current: (se mide post-migración)

Deploy Frequency:
Target: Daily deploys sin downtime
Current: (se mide post-migración)

MTTR (Mean Time To Recovery):
Target: < 30 minutos
Current: (se mide post-migración)
```

**Métricas de negocio**:
```
User Satisfaction:
Target: > 4.5/5 en encuestas
Método: NPS surveys post-migración

Support Tickets:
Target: No incremento vs baseline
Baseline: X tickets/semana
Current: (se mide post-migración)

API Uptime (para terceros):
Target: > 99.95%
SLA commitment: 99.9%
Current: (se mide post-migración)

Feature Velocity:
Target: 2x más rápido desarrollar features
Método: Tiempo promedio de feature
Current: (se mide post-migración)
```

### 11.5 Gestión de Riesgos

| Riesgo | Probabilidad | Impacto | Mitigación | Plan B |
|--------|--------------|---------|------------|--------|
| **Pérdida de datos** | Media | Crítico | - Dual write mode<br>- Validación automática<br>- DLQ para eventos fallidos | - Rollback inmediato<br>- Recuperar de backups<br>- Re-procesar eventos |
| **Performance degradado** | Media | Alto | - Load testing previo<br>- Traffic split gradual<br>- Auto-scaling configurado | - Rollback a sistema viejo<br>- Escalar recursos<br>- Optimizar queries |
| **Incompatibilidad de datos** | Baja | Alto | - Schema validation<br>- Tests con datos reales<br>- Dry-run migrations | - Rollback<br>- Ajustar mappers<br>- Re-migrar datos |
| **Downtime prolongado** | Baja | Crítico | - Migración sin downtime<br>- Health checks robustos<br>- Failover automático | - Activar DR site<br>- Comunicar a usuarios<br>- Compensación por SLA |
| **Bugs críticos en producción** | Media | Alto | - Extensive testing<br>- Staged rollout<br>- Feature flags | - Rollback inmediato<br>- Hotfix<br>- Post-mortem |
| **Equipo sin suficiente conocimiento** | Media | Medio | - Training antes de migración<br>- Pair programming<br>- Documentación completa | - Contratar consultores<br>- Extended timeline<br>- Mentoring |
| **Costos AWS exceden presupuesto** | Media | Medio | - Cost monitoring<br>- Reserved instances<br>- Auto-scaling limits | - Ajustar recursos<br>- Optimizar uso<br>- Negociar con AWS |
| **Fallo en comunicación gRPC** | Baja | Alto | - Circuit breaker<br>- Retry logic<br>- Fallback a REST | - Usar REST temporalmente<br>- Debug gRPC<br>- Escalar timeout |

### 11.6 Comunicación con Stakeholders

**Pre-Migración** (1 semana antes):
```
Email a stakeholders:
─────────────────────
Subject: Migración Backend HSE - Inicio [Fecha]

Resumen:
- Qué se migrará
- Cuándo comenzará
- Impacto esperado (ninguno/mínimo)
- Ventanas de mantenimiento (si aplica)
- Contacto de emergencia
```

**Durante Migración** (updates diarios):
```
Slack channel: #migration-updates

Daily update format:
┌─────────────────────────────────┐
│ Migración Backend - Día X       │
├─────────────────────────────────┤
│ Estado: ✅ On track             │
│ Traffic split: 50%              │
│ Error rate: 0.05% (✅ target)  │
│ Issues: Ninguno                 │
│ Próximos pasos: Incremento 75% │
└─────────────────────────────────┘
```

**Post-Migración** (1 semana después):
```
Email de cierre:
─────────────────────
Subject: ✅ Migración Backend HSE - Completada

Resumen:
- Estado final: Exitosa
- Métricas alcanzadas
- Mejoras obtenidas
- Próximos pasos
- Agradecimientos al equipo
```

---

**📄 Continúa en**: `12-13_Monitoreo_Ejecucion.md`