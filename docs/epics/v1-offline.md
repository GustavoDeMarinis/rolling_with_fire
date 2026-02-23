# Epic: Tiradas Offline Personales

Status: Draft
Version: V1
Owner: TBD

---

## Product Goal

Permitir que un usuario descargue la app, se registre y pueda hacer tiradas de dados RPG de forma inmediata, con historial persistente y macros de configuración guardadas, ya sea con o sin conexión activa.

---

## Value Delivered

- El jugador tiene una herramienta de dados confiable en el bolsillo sin depender de bots de Discord o apps genéricas.
- Las tiradas se generan en servidor, garantizando aleatoriedad no manipulable.
- Las macros guardadas eliminan la fricción de configurar dados compuestos en cada sesión.
- El historial permite revisar tiradas pasadas incluso días después.

---

## Backend Scope

- **Auth:**
  - `POST /auth/register` — registro con email + contraseña.
  - `POST /auth/login` — devuelve access token (15min) + refresh token (7d).
  - `POST /auth/refresh` — rota el refresh token.
  - `POST /auth/logout` — invalida el refresh token actual.
- **Rolls:**
  - `POST /rolls` — recibe input de configuración, genera la tirada server-side con `crypto.randomInt`, persiste en DB, devuelve resultado con breakdown.
  - `GET /rolls?page=1&limit=20` — historial paginado del usuario autenticado. Sin `room_id`.
- **Roll Configs (Macros):**
  - `POST /roll-configs` — guarda una configuración de dados compuesta.
  - `GET /roll-configs` — lista macros del usuario.
  - `PUT /roll-configs/:id` — edita una macro.
  - `DELETE /roll-configs/:id` — elimina una macro.
- **Usuario:**
  - `GET /users/me` — perfil del usuario autenticado.
  - `PUT /users/me` — actualización de username o email.

**Modelo de datos involucrado:** `users`, `rolls` (sin `room_id`), `roll_configs`.

---

## Mobile Scope

- Pantalla de registro / login con validación básica.
- Pantalla principal: tirador de dados ad-hoc (selector de tipo de dado + cantidad + bonus manual).
- Pantalla de historial personal: lista paginada de tiradas previas con breakdown expandible.
- Pantalla de macros: CRUD de configuraciones compuestas.
- Persistencia offline: las tiradas fallidas por falta de red se encolan localmente y se sincronizan cuando hay conexión.
- Manejo de refresh token automático en cada request.

---

## UX Scope

- Flujo de onboarding: registro → pantalla principal (sin tutoriales verbosos).
- Componente visual del dado: animación de "tirando" mientras se espera respuesta del servidor.
- Indicador claro del desglose de tirada (ej: 2d6 [3, 5] + 4 = **12**).
- Estado offline visible: banner o indicador cuando no hay conexión.
- UI kit base seleccionado y aplicado (decisión pendiente: React Native Paper / NativeBase).

> ⚠ El diseñador UX/UI no está disponible en V1. Usar componentes de UI kit estándar y no invertir en polish visual hasta V2.

---

## Acceptance Criteria

- [ ] Un usuario nuevo puede registrarse y hacer una tirada en menos de 60 segundos desde la instalación.
- [ ] La tirada es generada en el servidor (nunca en el cliente).
- [ ] El historial muestra correctamente al menos las últimas 50 tiradas.
- [ ] Una macro de tirada compuesta (ej: 2d6 + 4 + 1d8) puede ser guardada y reutilizada.
- [ ] Con conexión perdida, la app no crashea y permite tiradas queued que se sincronizan al reconectar.
- [ ] El refresh token se rota correctamente sin que el usuario note la sesión expirada.
- [ ] La contraseña del usuario está hasheada con bcrypt (salt rounds ≥ 12).

---

## Future Expansion Hooks

- El endpoint `POST /rolls` acepta `room_id` opcional desde V1 (null en este punto) para no cambiar el contrato en V2.
- El schema `roll_configs.config` es JSONB — soportará expansión a bonuses condicionales, ventaja/desventaja sin cambios de esquema.
- La tabla `rolls` incluye `config_id` nullable desde V1 — vinculación a macro usada disponible sin migración en V2.
- El módulo de auth incluye campo `role` en `users` (default: 'user') para futura diferenciación admin/moderador.
