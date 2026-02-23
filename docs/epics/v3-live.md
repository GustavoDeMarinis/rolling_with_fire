# Epic: Salas Live (Tiempo Real)

Status: Draft
Version: V3
Owner: TBD

---

## Product Goal

Transformar las salas de juego asíncronas en sesiones de juego en tiempo real, donde todos los miembros conectados ven las tiradas instantáneamente, tal como en una sesión de rol presencial pero desde cualquier lugar.

---

## Value Delivered

- La experiencia de sesión de rol remoto se vuelve inmersiva: ver el dado caer al mismo tiempo que el grupo.
- Se elimina el lag de "¿tiraste?" → "sí, mira el historial" propio del modo asíncrono.
- El director de juego puede ver en tiempo real quién está conectado y activo.
- La presencia en sala genera el sentido de "mesa virtual" que diferencia la app de un simple historial de dados.

---

## Backend Scope

- **WebSocket con Socket.IO:**
  - Autenticación en handshake: el cliente envía JWT en `socket.handshake.auth.token`.
  - Si el token es inválido o el usuario no es miembro de la sala, la conexión es rechazada.
  - Rooms de Socket.IO mapeadas 1:1 a rooms de la DB (`room:{uuid}`).
- **Eventos emitidos por el servidor:**
  - `roll:created` — cuando un miembro hace una tirada en sala: `{ userId, username, input, result, rolledAt }`.
  - `user:joined` — cuando un miembro se conecta a la sala: `{ userId, username }`.
  - `user:left` — cuando un miembro se desconecta o abandona: `{ userId }`.
  - `user:kicked` — cuando el admin expulsa a un miembro durante la sesión: `{ userId }`.
  - `room:closed` — cuando el admin elimina la sala durante una sesión activa.
- **Flujo de tirada live:**
  1. Cliente envía `POST /rolls` con `room_id` (HTTP REST, no WebSocket).
  2. El servidor genera la tirada, la persiste en DB.
  3. El servidor emite `roll:created` a todos los conectados en `room:{id}`.
  4. El cliente recibe la confirmación HTTP Y el evento WebSocket simultáneamente.
- **Presencia:**
  - El servidor mantiene en memoria (Map/Set) los usuarios conectados por sala.
  - `GET /rooms/:id/presence` — endpoint REST para obtener lista de usuarios conectados (útil al entrar a sala).
- **Infraestructura WebSocket:**
  - Socket.IO sobre el mismo proceso Node.js del monolito.
  - Si se añade un segundo servidor en el futuro, se requiere Redis Adapter (documentado en riesgos).

**Modelo de datos:** Sin cambios de schema. Solo activación de la capa de eventos sobre la DB existente.

---

## Mobile Scope

- Conexión WebSocket al entrar a una sala (y desconexión al salir).
- Stream de tiradas en tiempo real en la pantalla de sala: las tiradas nuevas aparecen animadas en la parte superior del feed.
- Indicador de presencia: avatares o lista de usernames conectados actualmente.
- Reconexión automática con exponential backoff si se pierde la conexión WebSocket.
- Manejo de token expirado durante sesión: reconectar automáticamente con refresh token sin interrumpir al usuario.
- Notificación local (in-app) si el admin cierra la sala durante la sesión.

---

## UX Scope

- Feed de tiradas en vivo visualmente distinto al historial asíncrono (animación de entrada, color diferente para tiradas propias vs ajenas).
- Indicador de estado de conexión WebSocket (conectado / reconectando / desconectado).
- Avatar / inicial del usuario junto a cada tirada en el feed live.
- Animación de dado personalizable (base de skins visuales para V4).
- Transición suave entre modo "solo historial" (sin conexión) y modo "live" (con WebSocket conectado).

---

## Acceptance Criteria

- [ ] Dos usuarios en la misma sala reciben la tirada de cualquiera de ellos en < 500ms (en condiciones de red normal).
- [ ] El JWT es validado al conectar el WebSocket; una conexión sin token válido es rechazada.
- [ ] Si el token expira durante la sesión, el cliente reconecta automáticamente con el refresh token.
- [ ] Al expulsar un usuario, el evento `user:kicked` lo desconecta del canal de sala en < 1 segundo.
- [ ] La pérdida y reconexión de WebSocket no genera tiradas duplicadas en el historial.
- [ ] El endpoint REST `/rolls` y el evento WebSocket son las dos únicas formas en que una tirada llega al cliente (no se procesa lógica de dado en el cliente).
- [ ] El servidor soporta al menos 50 conexiones WebSocket concurrentes sin degradación visible en el mismo VPS.

---

## Future Expansion Hooks

- La arquitectura de eventos está preparada para agregar **Redis Adapter** sin cambios en la lógica de negocio: solo se cambia la configuración de Socket.IO.
- El campo `metadata` JSONB en `rolls` permite adjuntar información de contexto live (ej: nombre de personaje activo, condición de ventaja) sin migración.
- Los eventos `user:joined` y `user:left` son la base para un sistema de **notificaciones push** O2O (V4+).
- El sistema de presencia es reutilizable para un futuro "lobby público" donde se vean salas con jugadores activos.
- La animación de dado en el feed live es el hook visual principal para las **skins de dados** de V4.
