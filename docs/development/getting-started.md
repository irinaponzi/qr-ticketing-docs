# Getting Started

Step-by-step guide to set up and run the EntradasQR project locally.

---

## Prerequisites

| Tool | Version | Purpose |
|---|---|---|
| **Go** | 1.21+ | Application runtime |
| **Docker** + **Docker Compose** | Latest | Infrastructure services |
| **Make** | Any | Build automation |
| **golangci-lint** | v2+ | Code linting (auto-installed by `make lint`) |

---

## Quick Start

### 1. Clone the repository

```bash
git clone https://github.com/iponzi/entradasQR.git
cd entradasQR
```

### 2. Start infrastructure

```bash
make infra
```

This launches MySQL, Redis, RabbitMQ, MailHog, Prometheus, Loki, and Grafana.

Wait ~10 seconds for MySQL to initialize the schemas.

### 3. Build all services

```bash
make build
```

Compiles three binaries into `bin/`:

- `bin/ticket-api`
- `bin/validator-api`
- `bin/qr-worker`

### 4. Configure environment

```bash
cp .env.example .env
# Edit .env — set COGNITO_USER_POOL_ID to your Cognito User Pool ID
```

The `.env` file is **gitignored** — never commit it. `.env.example` is the safe template committed to the repo.

### 5. Run the services

Open three terminals:

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

### 6. Test the flow

All endpoints require a Cognito JWT. Use **boto3** to obtain one:

```python
# Install boto3 once: pip install boto3 --user
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

Copy the printed tokens into:

```bash
ADMIN_TOKEN="<output-of-ADMIN_TOKEN>"
USER_TOKEN="<output-of-USER_TOKEN>"

# Create an event (admin)
curl -X POST http://localhost:8080/events \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -d '{"name":"Rock Festival","location":"Luna Park","date":"2026-07-15T20:00:00Z","capacity":1000,"ticket_price":150.00}'

# Purchase tickets (user)
curl -X POST http://localhost:8080/purchases \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $USER_TOKEN" \
  -d '{"buyer_email":"fan@example.com","event_id":1,"quantity":2}'

# Check email at MailHog (QR codes contain HMAC-signed tokens)
open http://localhost:8025

# Validate a ticket — use the HMAC-signed token from the QR code (admin)
curl -X POST http://localhost:8081/validate \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -d '{"token":"<signed-token-from-qr>"}'
```

Alternatively, import `docs/EntradasQR.postman_collection.json` into Postman and set the `admin_token` / `user_token` collection variables.

---

## Environment Variables

Copy `.env.example` to `.env` and fill in your values. The real `.env` is gitignored.

### Shared (all services)

| Variable | Default | Description |
|---|---|---|
| `COGNITO_REGION` | `us-east-1` | AWS region of the Cognito User Pool |
| `COGNITO_USER_POOL_ID` | _(required)_ | Cognito User Pool ID, e.g. `us-east-1_AbCdEf` |
| `HMAC_SECRET` | `change-me-in-production` | Secret for HMAC-SHA256 QR token signing |
| `RABBITMQ_URL` | `amqp://guest:guest@localhost:5672/` | RabbitMQ connection string |

### Ticket API

| Variable | Default | Description |
|---|---|---|
| `PORT` | `8080` | HTTP listen port |
| `DB_HOST` | `localhost` | MySQL host |
| `DB_PORT` | `3306` | MySQL port |
| `DB_USER` | `root` | MySQL user |
| `DB_PASSWORD` | `root` | MySQL password |
| `DB_NAME` | `tickets_db` | Database name |

### Validator API

| Variable | Default | Description |
|---|---|---|
| `PORT` | `8081` | HTTP listen port |
| `REDIS_HOST` | `localhost` | Redis server hostname |
| `REDIS_PORT` | `6379` | Redis server port |
| `TICKET_SERVICE_URL` | `http://localhost:8080` | Ticket API fallback URL |

### QR Worker

| Variable | Default | Description |
|---|---|---|
| `SMTP_HOST` | `localhost` | SMTP server host |
| `SMTP_PORT` | `1025` | SMTP server port |
| `SMTP_FROM` | `tickets@entradasqr.local` | Sender email address |
| `SMTP_USER` | _(empty)_ | SMTP username (empty = no auth, e.g. MailHog) |
| `SMTP_PASSWORD` | _(empty)_ | SMTP password |
| `QR_SIZE` | `256` | QR code image size in pixels |

---

## Makefile Targets

| Target | Description |
|---|---|
| `make build` | Compile all three binaries |
| `make run-ticket` | Run Ticket API |
| `make run-validator` | Run Validator API |
| `make run-qr-worker` | Run QR Worker |
| `make test` | Run all tests with verbose output |
| `make test-cover` | Run tests with coverage report |
| `make lint` | Run golangci-lint (auto-installs if missing) |
| `make infra` | Start Docker infrastructure |
| `make infra-down` | Stop Docker infrastructure |
| `make tidy` | Run `go mod tidy` |

---

## Project Structure

```
entradasQR/
├── cmd/
│   ├── ticket-api/          # Ticket API entry point
│   ├── validator-api/       # Validator API entry point
│   └── qr-worker/           # QR Worker entry point
├── internal/
│   ├── ticket/              # Ticket bounded context
│   │   ├── adapter/         # RabbitMQ publisher, QR generator, email sender
│   │   ├── handler/         # HTTP handlers
│   │   └── storage/         # MySQL repositories
│   ├── validator/           # Validator bounded context
│   │   ├── adapter/         # RabbitMQ consumer, HTTP client
│   │   ├── handler/         # HTTP handlers
│   │   └── storage/         # Redis repository
│   └── platform/            # Shared infrastructure
│       ├── config/          # Environment configuration
│       ├── database/        # MySQL connection
│       ├── metrics/         # Prometheus metrics & middleware
│       ├── middleware/      # JWT auth + IP rate limiting middleware
│       └── rabbitmq/        # RabbitMQ connection & topology
├── migrations/              # SQL schema files
├── configs/                 # Prometheus, Loki, Grafana configs
├── docs/                    # OpenAPI specs + Postman collection
├── docker-compose.yml       # Infrastructure services
└── Makefile                 # Build automation
```
