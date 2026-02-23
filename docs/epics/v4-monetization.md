# Epic: Monetización Completa

Status: Draft
Version: V4
Owner: TBD

---

## Product Goal

Activar un flujo de revenue sostenible mediante la venta de skins cosméticas y soporte a donaciones, con validación de compras en servidor para garantizar integridad y prevenir fraude, integrando Google Play Billing como canal principal.

---

## Value Delivered

- Los usuarios que quieren personalizar su experiencia pueden hacerlo con compras únicas (no suscripción).
- La app genera revenue real sin degradar la experiencia gratuita.
- El anti-fraude de skins garantiza que solo usuarios que pagaron obtienen contenido exclusivo.
- La monetización no depende de publicidad intrusiva, preservando la UX.

---

## Backend Scope

- **Catálogo de Skins:**
  - `GET /skins` — lista de skins disponibles con precio y tipo (`dice`, `table`, `theme`).
  - `GET /skins/:id` — detalle de skin.
- **Skins de Usuario:**
  - `GET /users/me/skins` — skins adquiridas por el usuario autenticado.
  - `POST /users/me/skins` — registrar adquisición de skin (con validación de compra de Google Play).
  - `PUT /users/me/skins/active` — seleccionar skins activas (por tipo).
- **Validación de Compras (Google Play):**
  - Al recibir `POST /users/me/skins`, el backend llama a Google Play Developer API para verificar que el `purchaseToken` es válido y no fue ya utilizado.
  - Si la verificación falla: HTTP 402 / 409, la skin NO se asigna.
  - Si es válida: se persiste en `user_skins`, se marca el token como consumido.
- **Donaciones:**
  - Integración con plataforma de donaciones (Stripe, Ko-fi, o equivalente) — alcance a definir.
  - El backend puede recibir webhooks de donaciones para otorgar un badge o skin especial a donadores.
- **Anti-fraude de Skins en sesión:**
  - Cada endpoint que devuelve información que incluye skins activas (perfil, sala, rolls) valida que la skin activa pertenece realmente al usuario via `user_skins`.
  - El cliente **no puede** declarar una skin activa que no esté validada server-side.

**Modelo de datos involucrado:** `skins` (nuevo), `user_skins` (nuevo), extension de `users` con campos de skin activa.

**Dependencia externa:** Google Play Developer API (requiere cuenta de desarrollador activa).

---

## Mobile Scope

- **Tienda In-App:**
  - Pantalla de tienda con catálogo de skins por categoría.
  - Preview visual de cada skin antes de comprar.
  - Flujo de compra vía Google Play Billing (nativo).
- **Gestión de Skins:**
  - Pantalla "Mi colección" con skins adquiridas.
  - Selector de skin activa por tipo (dado, tema, etc.).
- **Aplicación visual:**
  - La animación del dado respeta la skin activa del usuario.
  - El historial y feed live muestran la skin del dado del usuario que tiró.
- **Donaciones:**
  - Botón o sección "Apoyar al proyecto" con enlace a plataforma de donación.

> ⚠ Google Play Billing requiere un custom dev client en Expo. Planificar eject parcial o migración a bare workflow antes de iniciar V4 mobile.

---

## UX Scope

- La tienda debe sentirse premium y confiable, no genérica.
- El diseñador UX/UI debe estar activo y liderar el diseño de tienda y preview de skins.
- El flujo de compra debe ser < 3 taps desde "ver skin" hasta "comprar".
- Confirmación clara de compra exitosa con animación de la skin desbloqueada.
- Las skins activas deben verse reflejadas inmediatamente en toda la app (sin restart).
- Diferenciar claramente skins disponibles, compradas y activas.

---

## Acceptance Criteria

- [ ] Un usuario puede comprar una skin a través de Google Play Billing.
- [ ] El backend valida el `purchaseToken` con Google Play API antes de asignar la skin.
- [ ] Un token de compra ya consumido no puede ser reutilizado para obtener otra skin.
- [ ] Un usuario no puede activar una skin que no haya comprado, ni siquiera manipulando el cliente.
- [ ] Las skins activas se reflejan en el feed live de la sala para todos los miembros de la sala.
- [ ] La skin activa persiste entre sesiones (se guarda server-side, no solo local).
- [ ] El backend valida correctamente el estado de la skin en cada request que la incluya.
- [ ] Existe al menos 1 skin gratuita y 3 skins de pago en el catálogo inicial.

---

## Future Expansion Hooks

- El campo `type` en `skins` permite agregar nuevas categorías (ej: efectos de sonido, temas de sala) sin cambio de schema.
- El campo `metadata` JSONB en `skins` soporta propiedades específicas por tipo sin normalización adicional.
- El sistema de donaciones puede evolucionar a un modelo de **membresía / tier** (Patreon-style) sin cambiar el core de monetización.
- La tabla `user_skins` con `acquired_at` soporta campañas futuras de "skin de temporada" o "edición limitada".
- La integración con Google Play está preparada para extenderse a **suscripciones** si el modelo de negocio lo requiere.
- La validación server-side de skins es la base para un posible **marketplace** entre usuarios en versiones futuras.

---

## Open Strategic Questions (V4)

> Deben responderse antes de comenzar diseño de V4:

1. ¿Las skins son compras únicas o hay también suscripciones?
2. ¿Se incluye iOS en V4 o se mantiene Android-only?
3. ¿Hay una skin gratuita por default para todos, o la app base no tiene skin?
4. ¿El modelo de donación es externo (Ko-fi) o integrado en la app?
5. ¿Existe un programa de afiliados o referidos como canal de adquisición en V4?
