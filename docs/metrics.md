# Metrics — Rolling With Fire

## Principios

> **Solo medir lo que se puede actuar.** Cada métrica debe tener un umbral de acción definido.

---

## Métricas de Negocio (Producto)

### Adquisición

| Métrica | Descripción | Umbral de Acción |
|---------|-------------|-----------------|
| Instalaciones totales | Acumuladas desde Play Store | — (informativo) |
| Instalaciones activas (MAU) | Usuarios con al menos 1 sesión en 30 días | < 100 MAU en M3 → revisar estrategia de adquisición |
| Costo por instalación (CPI) | Solo si hay campañas de pago | — (no aplicable en V1-V2 bootstrap) |

### Retención

| Métrica | Descripción | Objetivo |
|---------|-------------|----------|
| D1 Retention | % usuarios que vuelven al día siguiente | > 40% |
| D7 Retention | % usuarios que vuelven en 7 días | > 20% |
| D30 Retention | % usuarios que vuelven en 30 días | > 10% |
| Sesiones por usuario activo (semanal) | Indicador de engagement | > 2 sesiones/semana |

### Engagement

| Métrica | Descripción | Señal Positiva |
|---------|-------------|----------------|
| Tiradas por sesión | Promedio de tiradas por sesión de usuario | > 5 tiradas/sesión |
| Macros creadas por usuario | Uso de configuraciones guardadas | > 1 macro por usuario activo |
| Tasa de uso de salas | % usuarios activos que usan al menos 1 sala (V2+) | > 30% |
| Duración de sesión en sala live (V3) | Tiempo promedio en sesión de sala activa | > 10 min |

---

## Métricas de Monetización (V4)

| Métrica | Descripción | Objetivo |
|---------|-------------|----------|
| DAU Paying | Usuarios únicos que pagan en el día | Crecer mes a mes |
| ARPU (Average Revenue Per User) | Revenue total / MAU | > $0.50 USD/MAU |
| Tasa de conversión free→paid | % de MAU que realizan al menos 1 compra | > 3% |
| Revenue por skin | Revenue por cada SKU de skin | Identifica ítems populares |
| Donaciones recibidas (si se implementa) | Total y promedio | — (informativo) |
| Churn de usuarios pagos | % usuarios pagos que no repiten compra en 90 días | < 40% |

---

## Métricas Técnicas

### Performance

| Métrica | Descripción | SLA |
|---------|-------------|-----|
| Latencia P95 de `/rolls` | 95% de peticiones de tirada resueltas en X ms | < 250ms |
| Latencia P95 de WebSocket (V3) | Tiempo desde roll en servidor hasta evento recibido | < 500ms |
| Uptime del servicio | Disponibilidad mensual | > 99.5% |
| Tiempo de respuesta de Auth | Login / refresh token | < 300ms |

### Infraestructura

| Métrica | Descripción | Umbral de Acción |
|---------|-------------|-----------------|
| CPU promedio del VPS | Media de 1h | > 70% sostenido → upgrade |
| RAM usada | Free memory del VPS | < 200MB libre → upgrade |
| Tamaño de tabla `rolls` | Filas acumuladas | > 1M → planificar particionamiento |
| Conexiones WebSocket activas | Pico concurrente | > 500 → evaluar Redis Adapter |
| Tiempo de query de historial | Queries `SELECT` sobre `rolls` paginadas | > 500ms → revisar índices |

### Errores

| Métrica | Descripción | Umbral de Acción |
|---------|-------------|-----------------|
| Tasa de errores 5xx | % requests con error de servidor | > 1% → alerta inmediata |
| Errores de autenticación | 401/403 por endpoint | Pico anormal → revisar posible ataque |
| Fallos de WebSocket | Reconnections inesperadas | > 5% de sesiones → revisar backend |

---

## Métricas de Producto por Versión

| Versión | Pregunta clave a responder |
|---------|---------------------------|
| V1 | ¿Los usuarios usan el historial y crean macros? (retención D7 > 20%) |
| V2 | ¿Los usuarios crean salas con otros jugadores? (tasa uso salas > 30%) |
| V3 | ¿Las sesiones live son más largas que las async? (duración sesión > 10 min) |
| V4 | ¿La conversión free→paid supera el 3%? |

---

## Herramientas de Tracking

| Herramienta | Propósito | Costo |
|-------------|-----------|-------|
| Firebase Analytics (gratuito) | Eventos de producto mobile | Gratis |
| Sentry (tier free) | Errores mobile + backend | Gratis (hasta 5k errores/mes) |
| Logs estructurados (pino en Node) | Debugging e historial de errores | $0 (autohosted) |
| Grafana + Prometheus (V3+) | Métricas de infraestructura | $0 (autohosted en VPS) |
| Google Play Console | Instalaciones, crashes, reviews | Incluido en developer account |
