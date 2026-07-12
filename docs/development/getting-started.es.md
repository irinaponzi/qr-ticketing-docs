# Primeros pasos

Guía paso a paso para configurar y ejecutar el proyecto EntradasQR localmente.

---

## Requisitos previos

| Herramienta | Versión | Propósito |
|---|---|---|
| **Go** | 1.21+ | Runtime de la aplicación |
| **Docker** + **Docker Compose** | Latest | Servicios de infraestructura |
| **Make** | Cualquiera | Automatización de build |
| **golangci-lint** | v2+ | Análisis estático (se instala automáticamente con `make lint`) |

---

## Inicio rápido

### 1. Clonar el repositorio

```bash
git clone https://github.com/iponzi/entradasQR.git
cd entradasQR
```

### 2. Levantar la infraestructura

```bash
make infra
```

Inicia MySQL, Redis, RabbitMQ, MailHog, Prometheus, Loki y Grafana.

Esperar ~10 segundos para que MySQL inicialice los esquemas.

### 3. Compilar todos los servicios

```bash
make build
```

Compila tres binarios en `bin/`:

- `bin/ticket-api`
- `bin/validator-api`
- `bin/qr-worker`

### 4. Configurar el entorno

```bash
cp .env.example .env
# Editar .env — configurar COGNITO_USER_POOL_ID con el ID del User Pool de Cognito
```

El archivo `.env` está en **gitignore** — nunca commitear. `.env.example` es la plantilla segura commiteada al repo.

### 5. Ejecutar los servicios

Abrir tres terminales:

=== "Terminal 1 — Ticket API"

    ```bash
    make run-ticket
    ```

=== "Terminal 2 — Validator API"

    ```bash
    make run-validator
    ```

=== "Terminal 3 — QR Worker"

    ```bash
    make run-qr-worker
    ```

### 6. Probar el flujo

Todos los endpoints requieren un JWT de Cognito. Usar **boto3** para obtener uno:

```python
# Instalar boto3 una vez: pip install boto3 --user
python3 << 'EOF'
import boto3
from botocore import UNSIGNED
from botocore.config import Config

client = boto3.client('cognito-idp', region_name='us-east-1', config=Config(signature_version=UNSIGNED))

admin = client.initiate_auth(
    AuthFlow='USER_PASSWORD_AUTH',
    ClientId='<YOUR_APP_CLIENT_ID>',
    AuthParameters={'USERNAME': 'admin@test.com', 'PASSWORD': '<ADMIN_PASSWORD>'}
)
user = client.initiate_auth(
    AuthFlow='USER_PASSWORD_AUTH',
    ClientId='<YOUR_APP_CLIENT_ID>',
    AuthParameters={'USERNAME': 'user@test.com', 'PASSWORD': '<USER_PASSWORD>'}
)
print('ADMIN_TOKEN:', admin['AuthenticationResult']['AccessToken'])
print('USER_TOKEN:', user['AuthenticationResult']['AccessToken'])
EOF
```

!!! note "Por qué boto3 y no curl"
    Los pools de Cognito creados con la nueva consola (2024+) requieren headers internos del SDK.
    `curl` devuelve `UnknownOperationException`. El SDK de cada plataforma (Amplify JS, Android, iOS) también funciona.

Copiar los tokens impresos en:

```bash
ADMIN_TOKEN="<output-of-ADMIN_TOKEN>"
USER_TOKEN="<output-of-USER_TOKEN>"

# Crear un evento (admin)
curl -X POST http://localhost:8080/events \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -d '{"name":"Rock Festival","location":"Luna Park","date":"2026-07-15T20:00:00Z","capacity":1000,"ticket_price":150.00}'

# Comprar entradas (user)
curl -X POST http://localhost:8080/purchases \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $USER_TOKEN" \
  -d '{"buyer_email":"fan@example.com","event_id":1,"quantity":2}'

# Verificar el email en MailHog (los códigos QR contienen tokens firmados con HMAC)
open http://localhost:8025

# Validar una entrada — usar el token HMAC del código QR del email (admin)
curl -X POST http://localhost:8081/validate \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -d '{"token":"<signed-token-from-qr>"}'
```

También se puede importar `docs/EntradasQR.postman_collection.json` en Postman y configurar las variables de colección `admin_token` / `user_token`.

---

## Variables de entorno

Copiar `.env.example` a `.env` y completar los valores. El `.env` real está en gitignore.

### Compartidas (todos los servicios)

| Variable | Default | Descripción |
|---|---|---|
| `COGNITO_REGION` | `us-east-1` | Región AWS del User Pool de Cognito |
| `COGNITO_USER_POOL_ID` | _(requerido)_ | ID del User Pool de Cognito, ej. `us-east-1_AbCdEf` |
| `HMAC_SECRET` | `change-me-in-production` | Secreto para firma HMAC-SHA256 de tokens QR |
| `RABBITMQ_URL` | `amqp://guest:guest@localhost:5672/` | Cadena de conexión a RabbitMQ |

### Ticket API

| Variable | Default | Descripción |
|---|---|---|
| `PORT` | `8080` | Puerto HTTP de escucha |
| `DB_HOST` | `localhost` | Host de MySQL |
| `DB_PORT` | `3306` | Puerto de MySQL |
| `DB_USER` | `root` | Usuario de MySQL |
| `DB_PASSWORD` | `root` | Contraseña de MySQL |
| `DB_NAME` | `tickets_db` | Nombre de la base de datos |

### Validator API

| Variable | Default | Descripción |
|---|---|---|
| `PORT` | `8081` | Puerto HTTP de escucha |
| `REDIS_HOST` | `localhost` | Hostname del servidor Redis |
| `REDIS_PORT` | `6379` | Puerto del servidor Redis |
| `TICKET_SERVICE_URL` | `http://localhost:8080` | URL de fallback al Ticket API |

### QR Worker

| Variable | Default | Descripción |
|---|---|---|
| `SMTP_HOST` | `localhost` | Host del servidor SMTP |
| `SMTP_PORT` | `1025` | Puerto del servidor SMTP |
| `SMTP_FROM` | `tickets@entradasqr.local` | Dirección de email del remitente |
| `SMTP_USER` | _(vacío)_ | Usuario SMTP (vacío = sin autenticación, ej. MailHog) |
| `SMTP_PASSWORD` | _(vacío)_ | Contraseña SMTP |
| `QR_SIZE` | `256` | Tamaño de la imagen QR en píxeles |

---

## Targets del Makefile

| Target | Descripción |
|---|---|
| `make build` | Compila los tres binarios |
| `make run-ticket` | Ejecuta el Ticket API |
| `make run-validator` | Ejecuta el Validator API |
| `make run-qr-worker` | Ejecuta el QR Worker |
| `make test` | Ejecuta todos los tests con salida detallada |
| `make test-cover` | Ejecuta tests con reporte de cobertura |
| `make lint` | Ejecuta golangci-lint (se instala si no está presente) |
| `make infra` | Levanta la infraestructura Docker |
| `make infra-down` | Detiene la infraestructura Docker |
| `make tidy` | Ejecuta `go mod tidy` |

---

## Estructura del proyecto

```
entradasQR/
├── cmd/
│   ├── ticket-api/          # Punto de entrada del Ticket API
│   ├── validator-api/       # Punto de entrada del Validator API
│   └── qr-worker/           # Punto de entrada del QR Worker
├── internal/
│   ├── ticket/              # Contexto acotado de tickets
│   │   ├── adapter/         # Publisher RabbitMQ, generador QR, email sender
│   │   ├── handler/         # Handlers HTTP
│   │   └── storage/         # Repositorios MySQL
│   ├── validator/           # Contexto acotado de validación
│   │   ├── adapter/         # Consumer RabbitMQ, cliente HTTP
│   │   ├── handler/         # Handlers HTTP
│   │   └── storage/         # Repositorio Redis
│   └── platform/            # Infraestructura compartida
│       ├── config/          # Configuración por entorno
│       ├── database/        # Conexión MySQL
│       ├── metrics/         # Métricas Prometheus y middleware
│       ├── middleware/      # Middleware de autenticación JWT + rate limiting por IP
│       └── rabbitmq/        # Conexión y topología RabbitMQ
├── migrations/              # Archivos de esquema SQL
├── configs/                 # Configuraciones de Prometheus, Loki y Grafana
├── docs/                    # Specs OpenAPI + colección Postman
├── docker-compose.yml       # Servicios de infraestructura
└── Makefile                 # Automatización de build
```
