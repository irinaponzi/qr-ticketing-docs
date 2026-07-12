# Descripción general de la arquitectura

EntradasQR sigue una **arquitectura de microservicios** con dos contextos delimitados y tres servicios que se comunican mediante eventos asíncronos a través de RabbitMQ.

---

## Arquitectura de alto nivel

```mermaid
C4Context
    title EntradasQR - System Context

    Person(admin, "Admin", "Creates events, cancels tickets, validates at venue")
    Person(buyer, "Buyer", "Purchases event tickets")

    System(ticketSvc, "Ticket Service", "Manages events, purchases, and ticket lifecycle")
    System(qrWorker, "QR Worker", "Generates QR codes and sends confirmation emails")
    System(validatorSvc, "Validator Service", "Validates tickets at entry points")
    System_Ext(cognito, "AWS Cognito", "JWT issuer — admin and user groups")

    SystemQueue(rmq, "RabbitMQ", "Async event bus for ticket lifecycle events")
    SystemDb(ticketsDb, "Tickets DB", "MySQL - Source of truth for tickets")
    SystemDb(redisDb, "Redis", "Key-value store - Fast ticket lookups")
    System_Ext(mailhog, "MailHog", "SMTP server for dev email")

    Rel(admin, ticketSvc, "Creates events, cancels tickets", "HTTP + JWT (admin)")
    Rel(buyer, ticketSvc, "Purchases tickets", "HTTP + JWT (user)")
    Rel(admin, validatorSvc, "Validates ticket QR codes", "HTTP + JWT (admin)")
    Rel(cognito, ticketSvc, "Issues JWTs, provides JWKS", "HTTPS")
    Rel(cognito, validatorSvc, "Issues JWTs, provides JWKS", "HTTPS")
    Rel(ticketSvc, rmq, "Publishes ticket.created / ticket.cancelled / purchase.completed")
    Rel(validatorSvc, rmq, "Publishes ticket.used")
    Rel(rmq, validatorSvc, "Delivers ticket events")
    Rel(rmq, qrWorker, "Delivers purchase.completed")
    Rel(rmq, ticketSvc, "Delivers ticket.used")
    Rel(qrWorker, mailhog, "Sends ticket emails", "SMTP")
    Rel(ticketSvc, ticketsDb, "Read/Write")
    Rel(validatorSvc, redisDb, "Read/Write")
    Rel(validatorSvc, ticketSvc, "Live fallback", "HTTP")
```

---

## Principios de diseño

### 1. Domain-Driven Design (DDD)
Cada servicio encapsula su propio contexto delimitado con entidades de dominio ricas, objetos de valor e interfaces de repositorio. Las reglas de negocio viven **exclusivamente** en la capa de dominio.

### 2. Ports & Adapters (Arquitectura Hexagonal)
El dominio define **puertos** (interfaces) para las dependencias externas. Las implementaciones concretas (**adaptadores**) se inyectan al inicio, manteniendo el dominio libre de preocupaciones de infraestructura.

### 3. Consistencia eventual
El Validator Service mantiene una **copia local** de los datos de tickets, sincronizada de forma asíncrona a través de RabbitMQ. Esto permite validaciones en sub-milisegundos en los puntos de entrada al evento sin depender de que el Ticket Service esté disponible.

### 4. Fallback en vivo
Cuando un ticket no se encuentra en la caché local (por ejemplo, por una condición de carrera durante la sincronización), el Validator recurre a una **llamada HTTP síncrona** al Ticket Service con un timeout de 3 segundos.

### 5. Consumidores idempotentes
Los consumidores de RabbitMQ están diseñados para manejar **reprocesamientos de mensajes** de forma segura. Crear un ticket ya existente o cancelar un ticket ya cancelado no produce ningún efecto.

### 6. Reconciliación bidireccional
Cuando el Validator Service marca un ticket como usado, publica un evento `ticket.used` de vuelta a RabbitMQ. El Ticket Service consume este evento y actualiza el estado del ticket en su propia base de datos, manteniendo ambos servicios sincronizados.

### 7. Rate limiting
La Validator API usa **rate limiting por IP** (algoritmo token bucket) para protegerse contra ataques de fuerza bruta que intenten adivinar UUIDs en el endpoint de validación.

---

## Límites de servicio

```mermaid
graph TB
    subgraph "Ticket Bounded Context"
        E[Event]
        T[Ticket]
        P[Purchase]
        E -->|"has many"| T
        P -->|"has many"| T
    end

    subgraph "Validator Bounded Context"
        VT[ValidTicket]
    end

    T -.->|"synced via RabbitMQ"| VT
```

| Aspecto | Ticket Service | QR Worker | Validator Service |
|---|---|---|---|
| **Responsabilidad** | CRUD de eventos, compra y cancelación de tickets | Generación de QR y envío de emails | Validación de tickets en el venue |
| **Base de datos** | `tickets_db` MySQL (puerto 3306) | Ninguna | Redis (puerto 6379) |
| **Puerto HTTP** | 8080 | — (solo consumidor) | 8081 |
| **Entidades** | Event, Ticket, Purchase | — (usa eventos de dominio) | ValidTicket |
| **Rol** | Fuente de verdad | Procesador asíncrono | Réplica optimizada para lectura |

---

## Flujo de datos

### Flujo de compra

```mermaid
sequenceDiagram
    participant B as Buyer
    participant TA as Ticket API
    participant TS as TicketService
    participant DB as tickets_db
    participant RMQ as RabbitMQ
    participant QW as QR Worker
    participant MH as MailHog
    participant VS as ValidatorService
    participant RD as Redis

    B->>TA: POST /purchases
    TA->>TS: Purchase(input)
    TS->>DB: Get Event
    TS->>DB: Reserve Tickets (sold_count += qty)
    TS->>DB: Add Purchase
    loop For each ticket
        TS->>DB: Add Ticket
        TS->>RMQ: Publish ticket.created
    end
    TS->>RMQ: Publish purchase.completed
    TS-->>TA: PurchaseResult
    TA-->>B: 201 Created

    RMQ-->>VS: Deliver ticket.created
    VS->>RD: SET ticket:{code} (idempotent)

    RMQ-->>QW: Deliver purchase.completed
    QW->>QW: Sign code (HMAC) + Generate QR
    QW->>MH: Send email with QR attachments
```

### Flujo de validación

```mermaid
sequenceDiagram
    participant S as Scanner
    participant VA as Validator API
    participant VS as ValidatorService
    participant RD as Redis
    participant TA as Ticket API

    S->>VA: POST /validate {token: "code.sig"}
    VA->>VA: Rate limit check (IP)
    VA->>VA: Verify HMAC signature
    VA->>VS: ValidateTicket(code)
    VS->>RD: GET ticket:{code}

    alt Found in Redis
        VS->>VS: Check status (active/used/cancelled)
        alt Active
            VS->>RD: SET ticket:{code} (used)
            VS->>RMQ: Publish ticket.used
            VS-->>VA: Valid ✅
        else Used/Cancelled
            VS-->>VA: Invalid ❌
        end
    else Not found in Redis
        VS->>TA: POST /tickets/lookup (fallback)
        alt Found & active
            VS->>RD: SET ticket:{code} (sync)
            VS->>RMQ: Publish ticket.used
            VS-->>VA: Valid ✅
        else Not found
            VS-->>VA: Invalid ❌
        end
    end

    VA-->>S: JSON response

    RMQ-->>TA: Deliver ticket.used
    TA->>DB: MarkAsUsed (idempotent)
```
