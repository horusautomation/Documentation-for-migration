# Plan de Migración - Backend HSE

## 1. Introducción y Contexto

Este documento describe el plan de migración del backend HSE hacia una arquitectura más robusta, escalable y mantenible. La aplicación ha crecido significativamente y actualmente incluye múltiples funcionalidades como eficiencia energética, automatización, control de acceso, entre otras.

### 1.1 Objetivos de la Migración

- Migrar el core a tecnologías más firmes y robustas
- Implementar una arquitectura bien definida y documentada
- Facilitar la escalabilidad sin requerir cambios estructurales mayores
- Mejorar la mantenibilidad y separación de responsabilidades

### 1.2 Decisiones Arquitectónicas Principales

1. **Modularización del Backend HSE**: El backend se dividirá en módulos que permitan manejar lógicas diferentes para cada entidad y/o módulo, facilitando el mantenimiento y la escalabilidad.

2. **Separación de Conexiones**: La lógica de conexiones (WebSockets, TCP, etc.) se separará de los respectivos controladores para permitir una manipulación libre, sin depender de servicios que no están directamente relacionados.

3. **Servicios Existentes**: Los servicios de macromedición y VRF se mantendrán tal como se viene trabajando, solo se adaptarán a la nueva estructura que se implementará.

---

## 2. Arquitectura General

### 2.1 Servicio Monolito Modular

El servicio principal será un **monolito modular** con base en arquitectura **hexagonal** (puertos y adaptadores). Esta arquitectura permite:

- Separación clara de responsabilidades
- Facilidad para agregar nuevos módulos en el futuro
- Mantenimiento independiente de cada módulo
- Posibilidad futura de extraer módulos a microservicios si es necesario

**Patrones de Diseño**:
- **Adapter**: Para adaptar interfaces externas
- **Singleton**: Para casos específicos donde se requiera una única instancia

### 2.2 Servicio de Devices/Connection

Servicio independiente que manejará todas las conexiones con dispositivos IoT. Utilizará arquitectura **CQRS** (Command Query Responsibility Segregation) para separar la lógica de consulta y guardado.

**Patrones de Diseño**:
- **Adapter**: Para adaptar diferentes protocolos de comunicación
- **Abstract Factory**: Para crear diferentes tipos de conexiones y handlers

**Funcionalidad**:
- Servidor HTTP para APIs REST
- Servidor TCP para comunicación con controladores
- Servidor WebSocket para comunicación en tiempo real con clientes
- Establecimiento de conexiones con diferentes controladores
- Levantamiento de servidores sockets para clientes

### 2.3 Comunicación entre Servicios

La comunicación entre el **Monolito Modular** y el **Servicio de Devices/Connection** se realizará mediante **gRPC** por las siguientes razones:

- **Rendimiento**: Mayor velocidad de comunicación comparado con REST
- **Seguridad**: Comunicación más segura entre servicios internos
- **Tipado fuerte**: Contratos bien definidos mediante Protocol Buffers
- **Streaming**: Soporte para comunicación bidireccional en tiempo real

**Nota**: Esta comunicación es interna entre servicios. Para clientes externos (terceros), se utilizará GraphQL/REST con autenticación mediante API Keys.

---

## 3. Stack Tecnológico

### 3.1 Servicio Monolito Modular

| Categoría | Tecnología | Propósito |
|-----------|-----------|-----------|
| **Framework** | NestJS con TypeScript | Framework principal del backend |
| **Servidor** | Fastify | Servidor HTTP de alto rendimiento |
| **Base de Datos** | PostgreSQL | Base de datos relacional principal |
| **ORM** | Prisma | Mapeo objeto-relacional |
| **Caché** | Redis | Base de datos de caché |
| **Autenticación** | JWT | Tokens de autenticación |
| **Seguridad** | Argon2 | Hashing de contraseñas |
| **Seguridad** | Cookies (SameSite, HttpOnly) | Manejo seguro de cookies |
| **API** | Apollo Server + GraphQL | API GraphQL |
| **Seguridad** | CORS | Control de acceso cross-origin |
| **Seguridad** | Throttler | Rate limiting |
| **Comunicación** | gRPC | Comunicación con servicio de Devices/Connection |
| **Monitoreo** | New Relic | Observabilidad y monitoreo de aplicación (APM, logs, métricas) |
| **Configuración** | dotenv | Manejo de variables de entorno |
| **Testing** | Jest | Framework de testing (unitario e integración) |

### 3.2 Servicio de Devices/Connection

| Categoría | Tecnología | Propósito |
|-----------|-----------|-----------|
| **Lenguaje** | Golang | Lenguaje de programación |
| **ORM** | GORM | ORM para Go |
| **Base de Datos** | PostgreSQL | Base de datos relacional |
| **Framework Web** | Echo | Framework para servidor HTTP y WebSocket |
| **Protocolo TCP** | Net (stdlib) | Servidor TCP nativo |
| **Comunicación** | gRPC | Comunicación con monolito modular |
| **Monitoreo** | New Relic (opcional) | Observabilidad y monitoreo de aplicación |

---

## 4. Módulos del Sistema

El monolito se dividirá en los siguientes módulos, cada uno con responsabilidades específicas:

### 4.1 Módulo de Autenticación (Auth)

**Responsabilidad**: Autenticar usuarios que se conecten a la plataforma.

**Tipos de Usuarios**:
- **Terceros (API)**: Modo desarrollador para integraciones externas
  - Autenticación mediante **API Keys**
  - Cada API Key tiene permisos específicos asociados
  - Los permisos determinan a qué servicios pueden acceder los terceros
  - Los clientes pueden generar sus propias API Keys con los permisos que se les hayan otorgado
- **Clientes**: Usuarios finales de la plataforma
  - Autenticación mediante JWT y cookies seguras

### 4.2 Módulo IoT/Core

**Responsabilidad**: Gestión de eficiencia energética, automatización de predios y funcionalidades core del sistema.

**Funcionalidades principales**:
- Gestión de usuarios y permisos
- Administración de organizaciones y establecimientos
- Control de dispositivos IoT
- Monitoreo y análisis energético

### 4.3 Módulo de Reservas (Booking/Reservation)

**Responsabilidad**: Gestionar clientes que no tienen acceso directo a la plataforma pero hacen uso de los dispositivos.

**Funcionalidades**:
- Realización de reservaciones de sitios
- Consulta de estados de dispositivos en áreas
- Gestión de huéspedes y acompañantes

### 4.4 Módulo de Acceso Colegios/Universidades

**Responsabilidad**: Gestionar accesos e interacciones específicas para entidades educativas.

**Nota**: Este módulo será detallado en futuras iteraciones del documento.

### 4.5 Módulo de Smart Access

**Responsabilidad**: Gestión de accesos inteligentes a residencias.

**Funcionalidades principales**:
- Control de acceso vehicular autorizado
- Gestión de permisos por propietarios de apartamentos
- Gestión de permisos por administración

**Nota**: Este módulo será detallado en futuras iteraciones del documento.

---

## 5. Modelo de Datos

### 5.1 Módulo IoT/Core - Eficiencia Energética

#### 5.1.1 Entidades Principales

1. **Users**
   - Entidad con mayor interacción en la plataforma
   - Puede gestionar y administrar recursos
   - Puede pertenecer a una `PartnerCompany` o `Organization`, o ser usuario `root`
   - Relación muchos a muchos con `Establishments` a través de `UserEstablishments`

2. **Roles**
   - Roles definidos por el sistema y roles personalizados por organizaciones
   - Roles base: `root`, `partner`, `admin`, `user`
   - Roles personalizados pueden ser creados para establecimientos específicos
   - Jerarquía: `root > partner > admin > user`

3. **Actions**
   - Acciones que puede realizar un rol dentro de la plataforma (permisos atómicos)
   - Ejemplos: `create_user`, `delete_establishment`, `view_reports`, etc.

4. **Permissions**
   - Relación muchos a muchos entre `Roles` y `Actions`
   - Define qué acciones puede realizar cada rol

5. **PartnerCompanies**
   - Entidades competitivas (asociados)
   - Encargados de conseguir sus propios clientes
   - Distribuyen dispositivos con sus marcas propias
   - Administrados por Horus

6. **Organizations**
   - Organizaciones y empresas que utilizan los servicios
   - Pueden ser administradas directamente por Horus
   - Pueden pertenecer a una `PartnerCompany`

7. **Establishments**
   - Lugares físicos o establecimientos de una organización
   - Ejemplo: Jerónimo Martins tiene 50 sedes en Barranquilla, cada sede es un `Establishment`
   - Relación muchos a muchos con `Users` a través de `UserEstablishments`
   - Pueden tener roles específicos asociados

8. **Areas**
   - Subdivisión de un `Establishment`
   - Representa cómo está dividido un sitio para su organización

#### 5.1.2 Diagrama de Relaciones

```
PartnerCompanies
    |
    v
Organizations
    |
    +---> Establishments
    |         |
    |         +---> UserEstablishments (tabla pivot)
    |         |
    |         v
    |       Areas
    |
    v
Users <------+
    |         |
    |         |
    v         |
Roles -------+-----> Permissions <------ Actions
```

**Nota**: Estas son las tablas principales. A medida que surjan más entidades, se anexarán sin mencionar aquí.

### 5.2 Módulo de Booking

#### 5.2.1 Entidades Principales

1. **Guest**
   - Huéspedes que realizan reservaciones
   - Pueden tener acompañantes

2. **GuestCompanions**
   - Tabla intermedia para relación recíproca muchos a muchos entre huéspedes
   - Permite definir acompañantes de un huésped

3. **Reservation/Booking**
   - Reservación de un huésped a una habitación (`Area`)
   - Manejo por estados (ver sección 6.3.1)
   - Control de fechas de check-in y check-out
   - Prevención de solapamiento de reservas (ver reglas en sección 6.3.2)

4. **Notifications**
   - Notificaciones enviadas a huéspedes
   - Contienen instrucciones de acceso a la habitación:
     - Fecha de ingreso
     - Fecha de salida
     - Lugar
     - Código de cerradura para acceso inteligente

5. **DoorlockCodes**
   - Códigos generados para actuación en cerraduras
   - Dependen de un dispositivo específico (Doorlock)

#### 5.2.2 Diagrama de Relaciones

```
GuestCompanions
    ^
    |
    |
Guests ---- > Bookings ---- > Areas ---- > Devices (1 Doorlock) ---> DoorlockCodes
    |
    |
    v
Notifications
```

### 5.3 Otros Módulos

- **Módulo Universidades/Escuelas**: Diagrama y entidades pendientes de detallar
- **Módulo Smart Access**: Diagrama y entidades pendientes de detallar

---

## 6. Reglas de Negocio y Flujos

### 6.1 Sistema de Roles y Permisos

#### 6.1.1 Jerarquía de Roles

```
root
  └──> partner
        └──> admin
              └──> user
              └──> roles creados para un establishment
```

#### 6.1.2 Reglas de Creación y Asignación de Roles

1. **Creación de Roles**:
   - `root`, `partner` y `admin` pueden crear roles
   - `user` NO puede crear roles
   - Al crear un rol, solo se pueden asignar permisos que estén **por debajo** del rol del creador, nunca iguales o superiores
   - Ejemplo: Si `admin` crea un rol, solo puede asignar permisos que él mismo tenga y que NO sean exclusivos de `admin`

2. **Permisos Exclusivos**:
   - Cada rol tiene permisos exclusivos que no pueden ser asignados a roles inferiores
   - **Estado**: Se definirán y añadirán al documento en un momento posterior

3. **Creación de Usuarios**:
   - `root`, `partner` y `admin` pueden crear usuarios
   - Solo pueden asignar roles iguales o inferiores al suyo
   - Ejemplo: Un `admin` solo puede crear usuarios con rol `admin` o inferior, nunca `root` o `partner`

#### 6.1.3 Visibilidad y Acceso por Rol

1. **Root**:
   - Acceso sin restricciones
   - Puede ver y modificar cualquier entidad u organización
   - Sin limitaciones

2. **Partner**:
   - Solo puede ver su `PartnerCompany` asociada
   - Puede ver todas las `Organizations` dentro de su `PartnerCompany`
   - Puede ver todos los `Establishments` dentro de sus organizaciones
   - **NO** puede ver o asignarse establecimientos que:
     - Pertenezcan a otro partner
     - No tengan partner asignado (pertenecen directamente a Horus)

3. **Admin**:
   - Solo puede ver `Establishments` dentro de su organización
   - Solo puede ver establecimientos que hayan sido asignados por su administrador en jefe
   - El administrador en jefe es el usuario que tiene todos los establecimientos asignados

4. **User**:
   - Acceso limitado según permisos asignados
   - Visibilidad restringida a recursos autorizados

### 6.2 Flujo de Inicio y Onboarding

#### 6.2.1 Proceso de Creación de Usuarios

1. **Usuario Root crea entidades base**:
   - Crea `PartnerCompany` (si es para un partner) o `Organization` (si es para un admin)
   - Esto mantiene la consistencia de datos

2. **Usuario Root crea usuario**:
   - Asigna el rol correspondiente (`partner`, `admin`, etc.)
   - Envía invitación por correo electrónico

3. **Usuario invitado**:
   - Recibe la invitación por correo
   - Puede aceptar o rechazar la invitación
   - Si acepta, debe configurar sus credenciales
   - Inicia sesión en la plataforma

**Regla de Consistencia**: Para que exista una invitación, el `root` debe haber creado previamente la `PartnerCompany` o `Organization` a la que el usuario va a pertenecer.

### 6.3 Módulo de Booking

#### 6.3.1 Estados de Reservas

Las reservas pueden tener los siguientes estados:

| Estado | Descripción |
|--------|-------------|
| **pending** | Reserva creada pero aún no ha sido confirmada por el huésped |
| **confirmed** | El huésped ha confirmado la reservación |
| **cancelled** | El huésped ha cancelado la reservación (estado modificado, registro no eliminado) |
| **in-progress** | La reserva está actualmente activa (dentro del rango de fechas de check-in y check-out) |
| **completed** | La reserva ya pasó su fecha y fue ocupada (estado final histórico) |

**Nota sobre terminología**: El estado `completed` indica que la reserva ya cumplió su ciclo completo y pasó su fecha de reservación. Este estado es útil para mantener un historial de reservas ocupadas.

#### 6.3.2 Reglas de Creación y Solapamiento de Reservas

1. **Permisos para Crear Reservas**:
   - Todos los roles pueden crear reservas para un guest, **excepto** el rol `user`
   - Los roles que pueden crear reservas deben tener el permiso correspondiente asignado
   - Roles permitidos: `root`, `partner`, `admin`, y roles personalizados (con el permiso adecuado)

2. **Prevención de Solapamiento**:
   - **Regla Principal**: No se pueden crear reservas que se solapen en fechas para la misma área/habitación
   - **Excepciones**: Se permite crear una reserva sobre otra si:
     - La reserva existente tiene estado `cancelled`
     - La reserva existente ha sido **eliminada** (registro físico eliminado de la base de datos)
   
3. **Manejo de Márgenes de Tiempo**:
   - Se debe considerar un margen de tiempo (minutos) entre reservas
   - No se permite crear una reserva en un rango de fechas donde ya existe una reserva activa
   - El sistema debe validar que no haya solapamiento considerando:
     - Fechas de check-in y check-out
     - Márgenes de tiempo configurados
     - Estados de reservas existentes (excluyendo `cancelled` y eliminadas)

4. **Operaciones sobre Reservas**:
   - **Cancelar**: Modifica el estado de la reserva a `cancelled` (el registro se mantiene en la base de datos)
     - **Restricción**: Solo se puede cancelar una reserva si está en estado `pending` o `confirmed`
     - **No permitido**: No se puede cancelar si la reserva está en estado `in-progress` o `completed`
   - **Eliminar**: Elimina físicamente el registro de la base de datos
     - **Restricción**: Solo se puede eliminar una reserva si está en estado `pending` o `confirmed`
     - **No permitido**: No se puede eliminar si la reserva está en estado `in-progress` o `completed`
   - Ambas operaciones permiten que se pueda crear una nueva reserva en ese rango de fechas (solo si se cumplen las restricciones de estado)

#### 6.3.3 Flujo de Estados de Reservas

```
pending → confirmed → in-progress → completed
   ↓
cancelled
```

**Transiciones válidas**:
- `pending` → `confirmed`: Cuando el huésped confirma
- `confirmed` → `in-progress`: Cuando llega la fecha de check-in
- `in-progress` → `completed`: Cuando pasa la fecha de check-out
- `pending` → `cancelled`: Cancelación antes de confirmar (permitido)
- `confirmed` → `cancelled`: Cancelación después de confirmar, antes de check-in (permitido)
- `in-progress` → `cancelled`: **NO PERMITIDO** - No se puede cancelar una reserva en progreso
- `completed` → `cancelled`: **NO PERMITIDO** - No se puede cancelar una reserva completada

**Nota**: Las operaciones de cancelar y eliminar solo están permitidas para reservas en estados `pending` o `confirmed`. Una vez que la reserva está `in-progress` o `completed`, no se puede cancelar ni eliminar.

---

## 7. Próximos Pasos y Pendientes

### 7.1 Definiciones Pendientes

1. **Permisos Exclusivos por Rol**: 
   - Definir las acciones exclusivas para cada rol (`root`, `partner`, `admin`, `user`)
   - **Estado**: Se añadirán al documento cuando estén listos

2. **Modelo de Datos - Módulo Universidades/Escuelas**:
   - Definir entidades principales
   - Crear diagrama de relaciones

3. **Modelo de Datos - Módulo Smart Access**:
   - Definir entidades principales
   - Crear diagrama de relaciones

4. **Servicios de Macromedición y VRF**:
   - Detallar cómo se adaptarán a la nueva estructura
   - Definir puntos de integración

### 7.2 Estrategia de Migración de Datos

La migración se realizará de forma gradual y controlada:

1. **Fase de Coexistencia**:
   - Ambos servidores (antiguo y nuevo) estarán funcionando en paralelo
   - Ambos sistemas guardarán y almacenarán datos de forma simultánea
   - Esto permite validar que el nuevo sistema funciona correctamente

2. **Fase de Transición**:
   - Migración selectiva: Solo se migrarán algunos datos críticos
   - Los datos no migrados se perderán intencionalmente
   - Validación continua de la integridad de los datos migrados

3. **Fase Final**:
   - Una vez validado el nuevo sistema, se apagará el servidor antiguo
   - El nuevo sistema quedará como único sistema activo
   - Monitoreo intensivo durante las primeras semanas

**Ventajas de este enfoque**:
- Permite validación gradual del nuevo sistema
- Minimiza el riesgo de pérdida de datos críticos
- Facilita el rollback si es necesario

### 7.3 Estrategia de Testing

La estrategia de testing incluirá múltiples niveles para garantizar una aplicación robusta y lista para escalar:

1. **Testing Unitario**:
   - Cobertura de lógica de negocio
   - Testing de servicios y casos de uso
   - Framework: Jest (NestJS) y testing nativo de Go

2. **Testing de Integración**:
   - Testing de comunicación entre módulos
   - Testing de comunicación entre servicios (gRPC)
   - Testing de integración con base de datos
   - Testing de integración con Redis
   - Validación de flujos completos de negocio

3. **Testing End-to-End (E2E)**:
   - Testing de flujos completos desde el cliente hasta la base de datos
   - Testing de APIs GraphQL y REST
   - Testing de comunicación WebSocket

**Objetivo**: Mantener una aplicación robusta, confiable y lista para escalar en todo momento.

### 7.4 Monitoreo y Observabilidad con New Relic

**New Relic** será integrado en el monolito modular (y opcionalmente en el servicio de devices) para proporcionar:

1. **Application Performance Monitoring (APM)**:
   - Monitoreo en tiempo real del rendimiento de la aplicación
   - Identificación de consultas lentas y cuellos de botella
   - Análisis de transacciones y endpoints

2. **Detección de Problemas**:
   - Identificación automática de partes de la aplicación que están fallando
   - Alertas proactivas sobre errores y excepciones
   - Análisis detallado de errores y stack traces

3. **Métricas y Dashboards**:
   - Métricas de rendimiento de base de datos (PostgreSQL)
   - Métricas de uso de Redis
   - Métricas de APIs (GraphQL, REST, gRPC)
   - Dashboards personalizados para monitoreo de módulos

4. **Logs**:
   - Centralización y análisis de logs
   - Búsqueda y filtrado avanzado
   - Correlación entre logs, métricas y traces

5. **Beneficios**:
   - Resolución de problemas 5x más rápida
   - Visibilidad completa del stack (frontend, backend, infraestructura)
   - Detección proactiva de problemas antes de que afecten a los usuarios

**Referencia**: [New Relic Platform](https://newrelic.com/)

### 7.5 Consideraciones Técnicas Adicionales

1. **Documentación**: Documentación de APIs (GraphQL schema, REST endpoints, gRPC contracts)
2. **CI/CD**: Pipeline de despliegue y automatización
3. **Seguridad**: Revisión de seguridad adicional (rate limiting, validación de inputs, etc.)
4. **Performance**: Estrategias de optimización (caché, índices de BD, etc.)

### 7.5 Sugerencias y Mejoras Propuestas

1. **Versionado de API**: Considerar versionado para GraphQL y REST APIs
2. **Event Sourcing**: Evaluar si CQRS podría beneficiarse de Event Sourcing para auditoría
3. **Message Queue**: Considerar agregar un message queue (RabbitMQ, Kafka) para comunicación asíncrona entre servicios en el futuro
4. **Documentación de Dominio**: Crear un glosario de términos del dominio (DDD)
5. **Protocol Buffers**: Documentar y versionar los contratos gRPC mediante Protocol Buffers
6. **API Keys Management**: Sistema robusto para gestión de API Keys con rotación, revocación y auditoría

---

## 8. Notas Finales

Este documento es una base fundamental para el proyecto de migración. Está diseñado para evolucionar y completarse a medida que se avance en la implementación y se definan más detalles técnicos y de negocio.

**Última actualización**: [Fecha pendiente]  
**Versión**: 1.0
