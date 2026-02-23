# Epic: Salas Asíncronas

Status: Draft
Version: V2
Owner: TBD

---

## Product Goal

Permitir que grupos de jugadores creen salas de juego persistentes donde puedan hacer tiradas compartidas y consultar el historial completo de la sala, sin necesidad de estar conectados al mismo tiempo.

---

## Value Delivered

- Los grupos de rol tienen un espacio de partida persistente, no efímero.
- El historial de tiradas de toda la campaña queda guardado y es consultable por cualquier miembro.
- El admin tiene control real sobre quién pertenece a la sala.
- Las salas privadas con contraseña permiten grupos cerrados sin exposición.

---

## Backend Scope

- **Rooms:**
  - `POST /rooms` — crea una sala (nombre, descripción, pública/privada, contraseña opcional).
  - `GET /rooms` — lista salas del usuario autenticado (donde es miembro).
  - `GET /rooms/:id` — detalle de sala + miembros.
  - `PUT /rooms/:id` — editar sala (solo admin).
  - `DELETE /rooms/:id` — eliminar sala con soft delete (solo admin).
- **Membresía:**
  - `POST /rooms/:id/join` — unirse a sala (con contraseña si es privada).
  - `DELETE /rooms/:id/leave` — salir de sala.
  - `DELETE /rooms/:id/members/:userId` — expulsar miembro (solo admin).
  - `PUT /rooms/:id/members/:userId/role` — cambiar rol de miembro (solo admin).
- **Rolls en sala:**
  - `POST /rolls` — el mismo endpoint de V1, ahora acepta `room_id` en el body.
  - `GET /rolls?room_id=:id&page=1&limit=20` — historial de tiradas de sala, paginado, para cualquier miembro.

**Seguridad:**
- Verificar que el usuario es miembro de la sala antes de permitir tirada o consulta de historial.
- Verificar que el usuario es admin antes de permitir operaciones de admin.
- Contraseña de sala hasheada con bcrypt.

**Modelo de datos involucrado:** `rooms`, `room_members` (nuevo en V2), `rolls` (campo `room_id` activado).

---

## Mobile Scope

- Pantalla de listado de salas del usuario.
- Pantalla de creación de sala (nombre, descripción, público/privado, contraseña).
- Pantalla de detalle de sala: historial de tiradas + lista de miembros.
- Flujo de unirse a sala (por ID o código + contraseña si aplica).
- Pantalla de gestión de sala para el admin: expulsar miembro, cambiar roles, eliminar sala.
- Tiradas desde el contexto de sala: el dado sabe en qué sala está y envía `room_id`.

---

## UX Scope

- Navegación clara entre "Mis tiradas personales" y "Salas".
- Identificación visual del rol en sala (badge de admin).
- Confirmación de acción destructiva antes de expulsar miembro o eliminar sala.
- Historial de sala con avatar/username de quien tiró + breakdown + timestamp.
- Indicador de sala pública vs privada.

> El socio UX/UI idealmente entra en V2. Si no está disponible, continuar con UI kit y hacer polish en el siguiente ciclo.

---

## Acceptance Criteria

- [ ] Un usuario puede crear una sala privada con contraseña y otro usuario puede unirse con ella.
- [ ] Las tiradas realizadas dentro de la sala aparecen en el historial de todos los miembros.
- [ ] Un miembro no puede ver las tiradas de una sala a la que no pertenece.
- [ ] El admin puede expulsar a un miembro y ese miembro pierde acceso al historial de sala inmediatamente.
- [ ] El admin puede eliminar la sala (soft delete); el historial se preserva para auditoría interna pero la sala no es visible.
- [ ] El modelo de contraseña de sala usa bcrypt (no se almacena en texto plano).
- [ ] El endpoint de tiradas no requiere cambio de contrato respecto a V1 (solo se activa `room_id`).

---

## Future Expansion Hooks

- La tabla `room_members` incluye `role` como TEXT, no ENUM, para agregar roles futuros (ej: 'spectator', 'gm') sin migración.
- El modelo de sala incluye campo `settings` JSONB nullable, reservado para configuraciones futuras (límite de miembros, sistemas de dado habilitados, etc.).
- El campo `archived_at` en `rooms` permite consultas de "salas archivadas" para una feature futura de recuperar campaña.
- El historial de tiradas por sala es la base de datos para el módulo de "estadísticas de campaña" en versiones futuras.
