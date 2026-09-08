# Métricas — Rolling With Fire

> **Solo medir lo que se puede actuar.** Cada métrica lleva un umbral de acción.

El hito está en `mvp.md` (M0–M4). Acá está qué mirar en cada uno y con qué se
instrumenta. Nada de esto aplica antes de M1: M0 corre en la LAN de casa y se valida
mirando dos pantallas.

---

## La métrica que define el producto

**Desfase de reveal entre dispositivos.** Milisegundos entre el primer cliente que
muestra el número y el último, para la misma tirada.

Es la única métrica que mide la tesis directamente (D2). Si es alta, no hay producto:
hay un dice roller con historial.

| Umbral | Acción |
|---|---|
| < 100ms | Se percibe como simultáneo. Objetivo |
| 100–300ms | Aceptable. Vigilar el margen de `reveal_at` |
| > 300ms | Revisar la estimación de offset de reloj y el `max(RTT)` de la sala |

Se mide desde M0, con dos dispositivos y cronómetro si hace falta. Después, con
telemetría del cliente: `reveal_at` planificado contra el instante real de reveal.

---

## Producto

### Adquisición y retención

| Métrica | Umbral de acción |
|---|---|
| Instalaciones activas (MAU) | Informativo hasta tener 50 usuarios reales |
| Retención D1 | > 40% |
| Retención D7 | > 20% |
| Retención D30 | > 10% |

Un jugador de rol juega una vez por semana. **La retención semanal importa más que
la diaria**, y una D1 baja no es necesariamente mala señal en este producto.

### Engagement

| Métrica | Señal positiva |
|---|---|
| Tiradas por sesión | > 5. Una sesión de rol real son 50–200 |
| Usuarios activos que tiran en una sala compartida (no la personal) | **> 60%.** La métrica que dice si la tesis se cumple |
| Templates creados por usuario activo | > 1 |
| Miembros conectados en simultáneo por sala activa | > 2. Si es 1, el producto se está usando como dice roller solo |
| Duración de sesión en sala con 2+ conectados | > 20 min |

> Si la mayoría de las tiradas ocurre en salas personales, el producto se está usando
> como un dice roller con historial — y eso es la señal de que la tesis falló, no de
> que falte una feature.

### Por hito

| Hito | Pregunta que responde |
|---|---|
| M0 | ¿Dos dispositivos revelan junto? ¿La animación pelea con Flutter? |
| M1 | ¿La gente invita a alguien con el código, o se queda sola? |
| M2 | ¿Se crean templates, o todos tiran ad-hoc? |
| M3 | ¿Se consulta el historial después de la sesión? |
| M4 | ¿Se usan las tiradas ocultas? ¿Quién: GMs o jugadores? |

---

## Técnicas

### Performance

| Métrica | SLA |
|---|---|
| Desfase de reveal (arriba) | < 100ms |
| Latencia del evento `roll`: tap → broadcast emitido | < 150ms P95 |
| RTT del channel por miembro | Se mide siempre: es el insumo de `reveal_at` |
| Tiempo de `join` a una sala (incluye `latest_seq`) | < 400ms P95 |
| Query de historial por cursor | < 200ms P95 |
| Uptime | > 99% |

### Infraestructura

| Métrica | Umbral de acción |
|---|---|
| CPU del VPS, media de 1h | > 70% sostenido → upgrade vertical |
| RAM libre | < 200MB → upgrade |
| WebSockets concurrentes | Informativo. En BEAM el límite práctico está mucho más arriba de donde estará el problema |
| Procesos de sala vivos | Informativo. Debe seguir a las salas con gente adentro; si no baja, hay GenServers que no terminan |
| Filas en `rolls` | > 1M → evaluar particionar por `rolled_at` |

### Errores

| Métrica | Umbral de acción |
|---|---|
| Crashes de channel / GenServer de sala | Cualquiera → investigar. En BEAM reinicia solo, y por eso pasa desapercibido |
| Tiradas rechazadas por variable faltante | Pico → es un problema de UX de perfiles (D4), no un bug |
| Reintentos con `client_roll_id` repetido | Informativo, pero si sube hay un problema de red o de timeouts |
| Tiradas duplicadas en el historial | **Cero.** Cualquiera es un bug de idempotencia |
| Reconexiones de WebSocket | > 5% de las sesiones → revisar |
| Crashes de la app | > 1% de sesiones → prioridad |

---

## Herramientas

| Herramienta | Para qué | Costo |
|---|---|---|
| `:telemetry` + `Phoenix.LiveDashboard` | Métricas del backend, incluido en Phoenix | $0 |
| Logger de Elixir, estructurado | Debugging | $0 |
| Sentry (tier free) | Errores de Flutter y de Elixir | $0 hasta 5k/mes |
| Firebase Analytics | Eventos de producto en mobile | $0 |
| Google Play Console | Instalaciones, crashes, reviews | Incluido |

Nada de Grafana/Prometheus hasta que LiveDashboard no alcance.
