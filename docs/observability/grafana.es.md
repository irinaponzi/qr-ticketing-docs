# Dashboards de Grafana

Instancia de Grafana preconfigurada con datasources y dashboards auto-provisionados.

---

## Acceso

| Propiedad | Valor |
|---|---|
| **URL** | [http://localhost:3000](http://localhost:3000) |
| **Usuario** | `admin` |
| **Contraseña** | `admin` |
| **Acceso anónimo** | Habilitado (rol Viewer) |

---

## Datasources (Auto-provisionados)

| Nombre | Tipo | URL |
|---|---|---|
| Prometheus | Prometheus | `http://prometheus:9090` |
| Loki | Loki | `http://loki:3100` |

Configurados en `configs/grafana/provisioning/datasources/datasources.yml`.

---

## Dashboard EntradasQR

Un dashboard pre-construido se carga automáticamente desde `configs/grafana/dashboards/entradas-qr.json`.

### Paneles

| Panel | Tipo | Métrica | Descripción |
|---|---|---|---|
| **Request Rate** | Time series | `http_requests_total` | Requests/seg por servicio |
| **P95 Latency** | Time series | `http_request_duration_seconds` | Latencia percentil 95 |
| **Error Rate** | Stat | `http_requests_total{status=~"5.."}` | Porcentaje de errores 5xx |
| **Events Created** | Stat | `events_created_total` | Total de eventos creados |
| **Tickets Purchased** | Stat | `tickets_purchased_total` | Total de tickets comprados |
| **Validation Results** | Pie chart | `tickets_validated_total` | Ratio válidos vs inválidos |
| **RabbitMQ Published** | Time series | `rabbitmq_events_published_total` | Eventos publicados/seg |
| **RabbitMQ Consumed** | Time series | `rabbitmq_events_consumed_total` | Eventos consumidos/seg |

### Layout

```
┌─────────────────┬─────────────────┬─────────────────┐
│  Request Rate   │   P95 Latency   │   Error Rate    │
├─────────────────┼─────────────────┼─────────────────┤
│ Events Created  │ Tickets Bought  │ Validation Rate │
├─────────────────┴─────────────────┴─────────────────┤
│              RabbitMQ Published / Consumed           │
└─────────────────────────────────────────────────────┘
```

---

## Estructura de Provisionamiento

```
configs/grafana/
├── dashboards/
│   └── entradas-qr.json           # Definición del dashboard
└── provisioning/
    ├── dashboards/
    │   └── dashboards.yml          # Configuración del proveedor de dashboards
    └── datasources/
        └── datasources.yml         # Fuentes de datos Prometheus + Loki
```

### Proveedor de Dashboards

```yaml
# configs/grafana/provisioning/dashboards/dashboards.yml
apiVersion: 1
providers:
  - name: 'default'
    folder: ''
    type: file
    options:
      path: /var/lib/grafana/dashboards
```

---

## Agregar Nuevos Paneles

1. Abrir Grafana en `http://localhost:3000`
2. Editar el dashboard existente o crear uno nuevo
3. Exportar el JSON del dashboard vía **Share → Export → Save to file**
4. Reemplazar `configs/grafana/dashboards/entradas-qr.json` con el archivo exportado
5. Reiniciar Grafana: `docker compose restart grafana`
