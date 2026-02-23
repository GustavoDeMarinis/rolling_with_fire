# Roadmap — Rolling With Fire

## Principios de Priorización

1. **Valor entregable antes que perfección técnica.**
2. **Cada versión debe poder monetizarse o validar hipótesis antes de avanzar.**
3. **Infraestructura se escala cuando el problema es real, no anticipado.**
4. **Dependencias deben estar resueltas antes de comenzar la versión siguiente.**

---

## V1 — Offline Personal

**Objetivo:** Una app funcional donde un usuario puede hacer tiradas de dados RPG, guardarlas en historial y sincronizarlas con su cuenta en la nube.

**Validación:** ¿Los usuarios retienen la app más de 7 días? ¿Crean macros de tirada?

| Área | Alcance |
|------|---------|
| Mobile | Tiradas ad-hoc offline, historial local, macros guardadas localmente, auth básica |
| Backend | Auth (register/login/refresh), historial de tiradas en DB, sincronización de macros |
| Infra | VPS + Docker Compose + PostgreSQL + Nginx + SSL |

**Dependencias:** Ninguna (punto de partida).

**Exit Criteria:**
- App instalable vía APK o Play Store interno.
- Tiradas generadas en backend (no client-side).
- Historial visible y paginado.
- 1 macro de tirada compuesta guardada correctamente.

---

## V2 — Salas Asíncronas

**Objetivo:** Los usuarios pueden crear grupos de juego (salas), invitar a jugadores y compartir tiradas que persisten en la sala aunque no estén todos conectados.

**Validación:** ¿Los usuarios crean salas con otros? ¿Vuelven a consultar el historial de sala?

| Área | Alcance |
|------|---------|
| Mobile | Crear/unirse a sala, historial de sala, tiradas en contexto de sala |
| Backend | CRUD de salas, membresía, permisos (admin/member), tiradas vinculadas a sala |
| Infra | Mismo VPS — sin cambios de infraestructura |

**Dependencias:** V1 completo. Modelo de datos de `rooms` y `room_members` definido.

**Exit Criteria:**
- Un usuario puede crear una sala privada con contraseña.
- Otro usuario puede unirse con la contraseña.
- Las tiradas en sala aparecen en el historial para todos los miembros.
- El admin puede expulsar un miembro y eliminar la sala.

---

## V3 — Salas Live (Tiempo Real)

**Objetivo:** Los usuarios conectados a la misma sala ven las tiradas de los demás en tiempo real.

**Validación:** ¿Las sesiones de juego aumentan en duración? ¿Aumenta la retención semanal?

| Área | Alcance |
|------|---------|
| Mobile | Indicador de usuarios conectados, tiradas transmitidas en vivo, notificación de reconexión |
| Backend | Socket.IO, autenticación de WebSocket, eventos de sala, presencia de usuarios |
| Infra | Mismo VPS — evaluar si el CPU/RAM requiere upgrade en esta etapa |

**Dependencias:** V2 completo. Modelo de salas estable.

**Exit Criteria:**
- Al menos 3 usuarios en misma sala ven tiradas en tiempo real (< 500ms latencia).
- Reconexión automática funcional.
- JWT validado en handshake WebSocket.
- Admin puede cerrar sala desde la app y todos son desconectados.

---

## V4 — Monetización Completa

**Objetivo:** Activar revenue con skins, donaciones y Google Play Billing. Establecer controles anti-fraude de monetización.

**Validación:** ¿La tasa de conversión free→paid supera el 3%? ¿La retención de usuarios pagos es mayor?

| Área | Alcance |
|------|---------|
| Mobile | Tienda de skins, aplicación visual de skins, flow de compra Google Play Billing |
| Backend | Catálogo de skins, validación server-side de skins activas, integración Google Play Verify |
| Infra | Evaluar separar DB a VPS propio si carga lo justifica |

**Dependencias:** V3 completo. Base de usuarios reales. UX/UI Designer activo.

**Exit Criteria:**
- Un usuario puede comprar una skin vía Google Play.
- La skin es validada server-side en cada sesión.
- El historial muestra skins activas correctamente.
- No es posible activar una skin no comprada mediante manipulación del cliente.

---

## Línea de Tiempo Estimada (Bootstrap)

```
Mes 1-2  │███ V1 │
Mes 3-4  │       │███ V2 │
Mes 5-6  │               │ Validación + UX polish │
Mes 7-9  │                               │███ V3 │
Mes 10-12│                                       │███ V4 │
```

> Los tiempos asumen 1 backend developer + 1 mobile developer. Ajustar si el Mobile Developer se incorpora tarde.

---

## Open Items Pre-Roadmap

- [ ] Confirmar si iOS entra en V1 o V3+.
- [ ] Definir modelo de límites del tier gratuito (cantidad de salas, macros, historial).
- [ ] Decidir si V2 incluye notificaciones push o se deja para V3.
