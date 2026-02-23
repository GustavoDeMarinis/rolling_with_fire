# Architecture — Rolling With Fire

## Filosofía de Diseño

> **Evolutionary Architecture:** Cada decisión técnica debe soportar V1 sin comprometer V4.
> No se construye para el futuro inmediato, se construye para no romper el futuro.

---

## Vista General por Versión

```
V1: Mobile ──────────────────────────────> Backend REST
         (offline-first + sync opcional)   (Auth + Historial)

V2: Mobile ─────────────────────────────> Backend REST
                                           (Salas + Async Rolls)

V3: Mobile ─────────────────────────────> Backend REST + WebSocket
                                           (Live Rooms + Events)

V4: Mobile ─────────────────────────────> Backend REST + WebSocket + Billing
                                           (Skins + Payments + Anti-fraude)
```

---

## Stack Técnico

### Mobile

| Tecnología | Decisión | Alternativa Considerada |
|------------|----------|------------------------|
| **React Native + Expo** | ✅ Aprobado | Kotlin puro (mayor costo inicial) |
| Expo Go (V1) | Para desarrollo rápido | EAS Build desde V2 |
| EAS Build (V2+) | Para distribución real | Play Store manual |
| AsyncStorage | Historial offline V1 | SQLite (si historial es complejo) |
| Socket.IO Client | WebSockets V3 | WS nativo |

> ⚠ **Riesgo Expo:** Algunas integraciones nativas (Google Play Billing, biometría) pueden requerir un custom dev client. Planificar el eject parcial antes de V4.

### Backend

| Tecnología | Decisión | Alternativa Considerada |
|------------|----------|------------------------|
| **Node.js + Express** | ✅ Aprobado | Fastify (mayor perf, menor ecosistema) |
| **Prisma ORM** | ✅ Aprobado | TypeORM (más verbose), Drizzle (más moderno pero menos maduro) |
| **PostgreSQL** | ✅ Aprobado | MySQL (menos features JSON), MongoDB (no relacional, pérdida de integridad) |
| **Socket.IO** | ✅ para V3 | WS puro (más control, más código) |
| JWT (Access + Refresh) | Auth stateless | Sessions (más complejo con WebSockets) |

### Infraestructura

| Componente | V1-V2 | V3-V4 |
|------------|-------|-------|
| Servidor | 1 VPS (Hetzner CX21 ~€4/mes) | Mismo VPS o upgrade vertical |
| Base de datos | PostgreSQL en mismo VPS | Separar VPS de DB si > 10k usuarios |
| Contenedores | Docker Compose | Docker Compose (mantener hasta necesitar K8s) |
| Reverse Proxy | Nginx | Nginx (agregar rate limiting en V3) |
| SSL | Let's Encrypt (Certbot) | Mismo |
| CI/CD | GitHub Actions básico | GitHub Actions + rollback strategy |

---

## Modelo de Datos — Diseño Evolutivo

### Core Entities

```sql
-- USERS
users (
  id          UUID PRIMARY KEY,
  username    TEXT UNIQUE NOT NULL,
  email       TEXT UNIQUE NOT NULL,
  password_hash TEXT NOT NULL,
  created_at  TIMESTAMPTZ DEFAULT NOW(),
  updated_at  TIMESTAMPTZ DEFAULT NOW()
)

-- ROOMS
rooms (
  id          UUID PRIMARY KEY,
  name        TEXT NOT NULL,
  description TEXT,
  is_public   BOOLEAN DEFAULT FALSE,
  password_hash TEXT,                    -- NULL si pública
  owner_id    UUID REFERENCES users(id),
  created_at  TIMESTAMPTZ DEFAULT NOW(),
  archived_at TIMESTAMPTZ               -- soft delete
)

-- ROOM MEMBERS
room_members (
  room_id     UUID REFERENCES rooms(id),
  user_id     UUID REFERENCES users(id),
  role        TEXT DEFAULT 'member',    -- 'admin' | 'member'
  joined_at   TIMESTAMPTZ DEFAULT NOW(),
  PRIMARY KEY (room_id, user_id)
)

-- ROLL CONFIGURATIONS (Macros)
roll_configs (
  id          UUID PRIMARY KEY,
  user_id     UUID REFERENCES users(id),
  name        TEXT NOT NULL,
  config      JSONB NOT NULL,           -- { components: [{dice: 'd6', count: 2}, {bonus: 4}] }
  created_at  TIMESTAMPTZ,
  updated_at  TIMESTAMPTZ
)

-- ROLLS (Event Store)
rolls (
  id          UUID PRIMARY KEY,
  user_id     UUID REFERENCES users(id),
  room_id     UUID REFERENCES rooms(id), -- NULL si personal
  config_id   UUID REFERENCES roll_configs(id), -- NULL si ad-hoc
  input       JSONB NOT NULL,           -- configuración usada
  result      JSONB NOT NULL,           -- { total, breakdown: [...] }
  rolled_at   TIMESTAMPTZ DEFAULT NOW()
)

-- SKINS (V4)
skins (
  id          UUID PRIMARY KEY,
  name        TEXT,
  type        TEXT,                     -- 'dice' | 'table' | 'theme'
  metadata    JSONB,
  price_cents INT
)

-- USER SKINS (V4)
user_skins (
  user_id     UUID REFERENCES users(id),
  skin_id     UUID REFERENCES skins(id),
  acquired_at TIMESTAMPTZ,
  PRIMARY KEY (user_id, skin_id)
)
```

> **Decisión de diseño:** `rolls.result` como JSONB permite almacenar el desglose completo sin normalizar ahora. Es extensible sin migración agresiva.

> **Decisión de diseño:** `roll_configs.config` como JSONB permite soportar configuraciones compuestas arbitrarias (2d6 + 4 + 1d8 + bonuses custom) sin esquema rígido. Se valida en servicio.

---

## Arquitectura de Tiradas — Event-Based

Las tiradas **siempre** se procesan en backend. El cliente solo envía la intención.

```
CLIENT                          SERVER
  │                               │
  │── POST /rolls (input: {...}) ──>│
  │                               │ valida input
  │                               │ genera RNG seguro (crypto.randomInt)
  │                               │ persiste en DB
  │                               │ (V3: emite evento WebSocket a sala)
  │<── 200 { result, breakdown } ──│
```

### Generación RNG

- V1-V2: `crypto.randomInt()` de Node.js (CSPRNG).
- V3+: Mismo CSPRNG, resultado transmitido vía WebSocket a todos los miembros de sala.
- **No se expone seed ni estado interno.**

---

## Arquitectura de Salas — Live (V3)

```
CLIENT A ──────────────────────── WS ──> SERVER (Socket.IO)
CLIENT B ──────────────────────── WS ──>   │
CLIENT C ──────────────────────── WS ──>   │── Room Channel: room:{id}
                                            │
                                        Events emitidos:
                                        - roll:created { userId, result, breakdown }
                                        - user:joined { userId, username }
                                        - user:kicked { userId }
```

### Estrategia de Autenticación en WebSockets

- El cliente envía el JWT en el handshake (`auth.token`).
- El servidor valida el token antes de permitir la conexión.
- Si el token expira durante la sesión: reconexión con refresh token.

---

## Seguridad

| Capa | Mecanismo |
|------|-----------|
| Autenticación | JWT (Access 15min + Refresh 7d) |
| Contraseñas | bcrypt (salt rounds ≥ 12) |
| Salas privadas | bcrypt sobre password de sala |
| Rate limiting | express-rate-limit (desde V1 en auth endpoints) |
| Generación de dados | Server-side con `crypto.randomInt` |
| WebSockets | JWT validation en handshake |
| Skins | Validación server-side en cada request que involucre skin activa |
| SQL Injection | Prisma con prepared statements |
| CORS | Whitelist estricta desde V1 |

---

## Decisiones Técnicas con Impacto Futuro

### 1. UUIDs vs Auto-increment IDs
**Decisión:** UUID para todas las entidades.
**Por qué:** Permite merge de datos, multi-región futura, y no expone contadores de recursos.

### 2. Soft Deletes
**Decisión:** `archived_at` en `rooms`, `deleted_at` en `users` (V2+).
**Por qué:** Preserva integridad referencial del historial de tiradas.

### 3. JSONB para Roll Config y Result
**Decisión:** JSONB en lugar de tablas normalizadas.
**Por qué:** La variabilidad de configuraciones de dados RPG es alta. Normalizar ahora es YAGNI.

### 4. Socket.IO vs WS puro
**Decisión:** Socket.IO.
**Por qué:** Manejo automático de reconexión, rooms nativas, fallback HTTP polling. El overhead es aceptable en V3.

### 5. Monolito vs Microservicios
**Decisión:** Monolito modular hasta V4 al menos.
**Por qué:** El equipo es pequeño. La complejidad operacional de microservicios en bootstrap es un riesgo mayor que el acoplamiento controlado.

---

## Módulos del Backend (estructura interna)

```
src/
  modules/
    auth/         # register, login, refresh, logout
    users/        # CRUD de perfil
    rooms/        # CRUD + membresía + permisos
    rolls/        # generación, validación, historial
    skins/        # catálogo, asignación, validación (V4)
  events/         # event emitters para WebSocket (V3)
  middleware/     # auth, rate-limit, error handler
  config/         # env, db, jwt config
  prisma/         # schema, migrations
```
