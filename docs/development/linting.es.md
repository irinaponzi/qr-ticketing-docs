# Linting y calidad de código

Configuración de análisis estático y estándares de código.

---

## golangci-lint

El proyecto usa [golangci-lint](https://golangci-lint.run/) v2 para análisis estático exhaustivo.

### Ejecutar el linter

```bash
make lint
```

Instala automáticamente `golangci-lint` si no está presente y ejecuta todos los linters configurados.

---

## Configuración

Definida en `.golangci.yml` en la raíz del proyecto.

### Linters habilitados

| Linter | Categoría | Propósito |
|---|---|---|
| `govet` | Corrección | Reporta construcciones sospechosas |
| `errcheck` | Corrección | Verifica errores no controlados |
| `staticcheck` | Corrección | Análisis estático avanzado |
| `unused` | Limpieza | Detecta código sin usar |
| `gosimple` | Estilo | Sugiere simplificaciones de código |
| `ineffassign` | Corrección | Detecta asignaciones inefectivas |
| `revive` | Estilo | Linter extensible para Go (reemplaza golint) |
| `gocritic` | Estilo | Patrones de código opinados |
| `gofmt` | Formato | Impone el formato estándar |
| `goimports` | Formato | Impone el orden de imports |

### Exclusiones

La configuración incluye exclusiones específicas para falsos positivos comunes:

| Patrón | Linter | Razón |
|---|---|---|
| `defer .*\.Close\(\)` | errcheck | Los errores de close diferido se ignoran intencionalmente |
| `msg\.Ack\(` / `msg\.Nack\(` | errcheck | ack/nack de RabbitMQ en loops de consumer |
| `stutters` | revive | Permite el patrón de naming `ticket.TicketService` |
| `package name` | revive | Permite `metrics.HTTPMetricsMiddleware` |
| `os.Exit` | gocritic | Permitido en funciones `main()` |

---

## Estándares de GoDoc

Todos los tipos, funciones, métodos y constantes exportados deben tener comentarios GoDoc.

### Formato

```go
// FunctionName does something specific.
//
// Parameters:
//   - param1: Description of the first parameter.
//   - param2: Description of the second parameter.
//
// Returns:
//   - *Type: Description of the return value.
//   - error: Conditions that cause an error.
func FunctionName(param1 string, param2 int) (*Type, error) {
```

### Reglas

- **Primera línea** debe comenzar con el nombre del identificador
- La sección **Parameters** documenta cada parámetro de entrada
- La sección **Returns** documenta cada valor de retorno
- Mantener las descripciones concisas pero informativas
- Documentar explícitamente las condiciones de error

---

## Checklist de calidad de código

Antes de enviar código, verificar:

- [ ] `make build` compila sin errores
- [ ] `make lint` no reporta ningún problema
- [ ] `make test` pasa todos los tests
- [ ] Todos los identificadores exportados tienen comentarios GoDoc
- [ ] Las entidades de dominio siguen el patrón de atributos privados
- [ ] Las interfaces de repositorio se mantienen mínimas
- [ ] Sin imports de infraestructura en paquetes de dominio
- [ ] El manejo de errores sigue las convenciones del dominio (`ErrNotFound`, `ErrBusinessRule`)

---

## Integración con CI

Para agregar linting a un pipeline de CI:

```yaml
# Ejemplo de step en GitHub Actions
- name: Lint
  uses: golangci/golangci-lint-action@v6
  with:
    version: latest
```

La configuración `.golangci.yml` es detectada automáticamente por la action.
