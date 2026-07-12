# Estrategia de testing

Enfoque de testing integral que cubre entidades de dominio, servicios, handlers y repositorios.

---

## Distribución de tests

| Capa | Paquete | Tests | Enfoque |
|---|---|---|---|
| **Entidades** | `internal/ticket/` | 20+ | Tests unitarios puros, sin mocks |
| **Entidades** | `internal/validator/` | 10+ | Tests unitarios puros, sin mocks |
| **Servicios** | `internal/ticket/` | 10+ | Mocks manuales para todos los puertos |
| **Servicios** | `internal/validator/` | 12+ | Mocks manuales para todos los puertos |
| **Handlers** | `internal/ticket/handler/` | 11 | `httptest` + mocks manuales |
| **Handlers** | `internal/validator/handler/` | 6 | `httptest` + mocks manuales |
| **Repositorios** | `internal/ticket/storage/` | 11 | `sqlmock` para MySQL |
| **Repositorios** | `internal/validator/storage/` | 8 | `sqlmock` para MySQL |

**Total: 65+ tests**

---

## Ejecutar los tests

```bash
# Todos los tests con salida detallada
make test

# Con reporte de cobertura
make test-cover

# Paquete específico
go test ./internal/ticket/... -v -count=1

# Test individual
go test ./internal/ticket/ -run TestPurchase_Success -v
```

---

## Patrones de testing

### 1. Tests de entidades (dominio puro)

Sin dependencias, sin mocks. Se testean invariantes, transiciones de estado y validaciones.

```go
func TestNewEvent_InvalidCapacity(t *testing.T) {
    _, err := NewEvent(1, "Concert", "Venue", futureDate, 0)
    if err == nil {
        t.Error("expected error for zero capacity")
    }
}

func TestTicket_Cancel_AlreadyUsed(t *testing.T) {
    tk, _ := NewTicket(1, 10, 100)
    _ = tk.MarkAsUsed()

    err := tk.Cancel()
    if err == nil {
        t.Error("expected error cancelling used ticket")
    }
}
```

### 2. Tests de servicios (con mocks manuales)

Cada puerto (repositorio, publisher, etc.) tiene un mock manual que implementa la interfaz:

```go
type mockEventRepository struct {
    getFunc    func(ctx context.Context, id int) (*Event, error)
    addFunc    func(ctx context.Context, event *Event) error
    updateFunc func(ctx context.Context, event *Event) error
}

func (m *mockEventRepository) Get(ctx context.Context, id int) (*Event, error) {
    return m.getFunc(ctx, id)
}
```

Los tests inyectan comportamiento específico por caso de prueba:

```go
func TestPurchase_EventNotFound(t *testing.T) {
    eventRepo := &mockEventRepository{
        getFunc: func(ctx context.Context, id int) (*Event, error) {
            return nil, nil  // not found
        },
    }
    svc := NewTicketService(eventRepo, ...)
    _, err := svc.Purchase(ctx, input)
    // assert ErrNotFound
}
```

#### Simular IDs asignados por la base de datos

Los métodos `Add` de los repositorios asignan el ID generado por la base de datos mediante `SetID()`. Los mocks deben replicar este comportamiento para verificar que el caller recibe el ID correcto:

```go
eventRepo := &mockEventRepo{
    addFunc: func(ctx context.Context, event *ticket.Event) error {
        event.SetID(42) // simula auto-increment de la DB
        return nil
    },
}
```

Este patrón aplica a las entidades `Event`, `Ticket` y `Purchase`.

### 3. Tests de handlers (httptest)

Se usa `net/http/httptest` para testear el ciclo request/response HTTP:

```go
func TestCreateEvent_InvalidBody(t *testing.T) {
    handler := NewTicketHandler(nil, nil, testLogger())
    req := httptest.NewRequest(http.MethodPost, "/events",
        bytes.NewBufferString("not json"))
    rr := httptest.NewRecorder()

    handler.CreateEvent(rr, req)

    if rr.Code != http.StatusBadRequest {
        t.Errorf("expected 400, got %d", rr.Code)
    }
}
```

Para endpoints que usan parámetros de URL de chi, se inyecta un contexto de ruta:

```go
rctx := chi.NewRouteContext()
rctx.URLParams.Add("code", "abc-123")
req = req.WithContext(context.WithValue(req.Context(), chi.RouteCtxKey, rctx))
```

### 4. Tests de repositorios (sqlmock)

Se usa `github.com/DATA-DOG/go-sqlmock` para testear queries SQL sin una base de datos real:

```go
func TestMySQLEventRepository_Get_Success(t *testing.T) {
    db, mock, err := sqlmock.New()
    // ...
    rows := sqlmock.NewRows(columns).AddRow(values...)
    mock.ExpectQuery("SELECT .+ FROM events").
        WithArgs(1).
        WillReturnRows(rows)

    repo := NewMySQLEventRepository(db)
    event, err := repo.Get(ctx, 1)
    // assert event fields
    // assert mock expectations met
}
```

---

## Cobertura de tests

```bash
make test-cover
```

Genera `coverage.out` con cobertura por función. Ver reporte HTML:

```bash
go tool cover -html=coverage.out
```

### Objetivos de cobertura

| Paquete | Objetivo |
|---|---|
| Entidades de dominio | > 90% |
| Servicios de dominio | > 80% |
| Handlers | > 70% |
| Repositorios | > 70% |

---

## ¿Por qué mocks manuales?

Este proyecto usa **structs de mock manuales** en lugar de herramientas de generación de código (ej. gomock, mockery) porque:

1. **Simplicidad** — Sin herramientas extra ni archivos generados
2. **Legibilidad** — Los mocks están en el mismo archivo de test, fáciles de entender
3. **Flexibilidad** — Cada test define el comportamiento exacto del mock mediante campos de función
4. **Superficie reducida** — Las interfaces tienen como máximo 2-4 métodos, los mocks manuales son triviales

Para proyectos más grandes, considerar `go generate` + mockery para generación automática de mocks.
