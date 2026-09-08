# Modelo de datos — MVP

Postgres vía Ecto. Es el schema canónico: `architecture.md` no lo repite.

---

## Tablas

```sql
users (
  id            UUID PRIMARY KEY,
  username      TEXT UNIQUE NOT NULL,
  -- Sin email ni password_hash: la credencial cuelga de user_identities,
  -- que se escribe cuando se elija el mecanismo (D15).
  inserted_at   TIMESTAMPTZ NOT NULL,
  updated_at    TIMESTAMPTZ NOT NULL
)

rooms (
  id            UUID PRIMARY KEY,
  name          TEXT NOT NULL,
  description   TEXT,
  invite_code   TEXT UNIQUE,          -- rotable por admin; NULL = sala cerrada
  is_personal   BOOLEAN NOT NULL DEFAULT FALSE,
  next_seq      BIGINT NOT NULL DEFAULT 1,
  archived_at   TIMESTAMPTZ,          -- soft delete
  inserted_at   TIMESTAMPTZ NOT NULL,
  updated_at    TIMESTAMPTZ NOT NULL
)
-- Sin owner_id: el rol vive solo en room_members (D7).
-- is_personal: playground de un solo miembro. Sin invite_code, no admite
-- invitados, no se archiva (D8).

profiles (
  id            UUID PRIMARY KEY,
  user_id       UUID NOT NULL REFERENCES users(id),
  name          TEXT NOT NULL,        -- "Jorge el Guerrero", "Dragón Rojo"
  inserted_at   TIMESTAMPTZ NOT NULL,
  updated_at    TIMESTAMPTZ NOT NULL,
  UNIQUE (user_id, name)
)
-- Un perfil: paquete de variables + templates. Sirve para PJ, NPC o monstruo.
-- Se crea una por defecto al registrarse, con el username (D4).

room_members (
  room_id       UUID NOT NULL REFERENCES rooms(id),
  user_id       UUID NOT NULL REFERENCES users(id),
  role          TEXT NOT NULL DEFAULT 'member',   -- 'admin' | 'member'
  active_profile_id UUID REFERENCES profiles(id) ON DELETE SET NULL,
  joined_at     TIMESTAMPTZ NOT NULL,
  PRIMARY KEY (room_id, user_id)
)
-- Invariante: toda sala no archivada tiene >= 1 admin.
-- Se verifica en la transacción de demote / leave / kick.
-- active_profile_id: qué perfil usás en esta sala. NULL cae al perfil por defecto.

variables (
  id            UUID PRIMARY KEY,
  profile_id      UUID NOT NULL REFERENCES profiles(id) ON DELETE CASCADE,
  name          TEXT NOT NULL,        -- "FUERZA"
  value         INT NOT NULL,
  inserted_at   TIMESTAMPTZ NOT NULL,
  updated_at    TIMESTAMPTZ NOT NULL,
  UNIQUE (profile_id, name)
)

roll_templates (
  id            UUID PRIMARY KEY,
  profile_id      UUID NOT NULL REFERENCES profiles(id) ON DELETE CASCADE,
  name          TEXT NOT NULL,        -- "Daño Espada Larga de Fuego"
  steps         JSONB NOT NULL,
  inserted_at   TIMESTAMPTZ NOT NULL,
  updated_at    TIMESTAMPTZ NOT NULL
)

rolls (
  id            UUID PRIMARY KEY,
  room_id       UUID NOT NULL REFERENCES rooms(id),
  user_id       UUID NOT NULL REFERENCES users(id),
  seq           BIGINT NOT NULL,      -- monótono por sala: cursor + replay
  template_id   UUID REFERENCES roll_templates(id) ON DELETE SET NULL,
  template_name TEXT,                 -- snapshot: sobrevive al borrado del template
  profile_id      UUID REFERENCES profiles(id) ON DELETE SET NULL,
  profile_name    TEXT NOT NULL,        -- snapshot: quién tiró, en ficción
  total         INT NOT NULL,         -- columna real, indexable (D11)
  breakdown     JSONB NOT NULL,
  visibility    TEXT NOT NULL DEFAULT 'everyone',  -- 'everyone'|'admins'|'self'
  leaves_trace  BOOLEAN NOT NULL DEFAULT TRUE,
  reveal_at     TIMESTAMPTZ NOT NULL,
  rolled_at     TIMESTAMPTZ NOT NULL,
  UNIQUE (room_id, seq)
)

CREATE INDEX ON rolls (room_id, seq DESC);
CREATE INDEX ON rolls (room_id, user_id);
```

### Sobre `seq`

`rooms.next_seq` se incrementa en la misma transacción que inserta la tirada.

En Phoenix hay un atajo natural: **todas las tiradas de una sala pasan por el
GenServer de esa sala**, un solo proceso, así que la asignación de `seq` está
serializada por construcción y no hay contención. La columna existe igual, para
que el número sobreviva a la muerte del proceso.

---

## `steps` — la definición de la tirada

Lista ordenada. Se evalúa de izquierda a derecha sobre un acumulador que arranca
en 0 (D3). El primer paso es siempre aditivo.

```json
[
  { "op": "+", "kind": "dice",     "count": 2, "sides": 6, "label": "Espada Larga" },
  { "op": "+", "kind": "dice",     "count": 1, "sides": 6, "label": "Fuego" },
  { "op": "*", "kind": "literal",  "value": 2,             "label": "Crítico" },
  { "op": "+", "kind": "variable", "variable": "FUERZA",  "label": "Fuerza" },
  { "op": "+", "kind": "literal",  "value": 16,            "label": "Rage" }
]
```

- `op`: `+` `-` `*` (división queda fuera del MVP: redondeo es una decisión de
  sistema de rol, no de la app).
- `kind`: `dice` | `literal` | `variable`.
- Las variables se referencian **por nombre**, no por id: se resuelven contra el
  `active_profile_id` de la sala al momento de tirar (D4). Así un template sirve para
  varios perfiles sin duplicarse. Si el perfil activo no tiene esa variable, la tirada
  **falla con un error explícito**; nunca asume 0.
- `label`: obligatorio. Es lo que hace legible el desglose; sin él la tirada es
  una cuenta anónima.

---

## `breakdown` — el resultado

Snapshot completo: **nombres y valores**, resueltos. El front no recalcula nada,
ni siquiera los acumulados parciales.

```json
{
  "steps": [
    { "op":"+", "kind":"dice",     "label":"Espada Larga", "sides":6, "rolls":[3,5], "subtotal":8,  "acc":8  },
    { "op":"+", "kind":"dice",     "label":"Fuego",        "sides":6, "rolls":[4],   "subtotal":4,  "acc":12 },
    { "op":"*", "kind":"literal",  "label":"Crítico",      "value":2,                "acc":24 },
    { "op":"+", "kind":"variable", "label":"Fuerza",       "value":3,                "acc":27 },
    { "op":"+", "kind":"literal",  "label":"Rage",         "value":16,               "acc":43 }
  ]
}
```

**El valor de la variable se copia, no se referencia.** Si mañana FUERZA sube a 5,
la tirada de hace tres sesiones tiene que seguir diciendo 3. Un log que se
reescribe solo no es un log.

Por la misma razón `template_name` se copia a la tirada: borrar la macro no puede
dejar huérfano el historial. Lo mismo con el perfil — `rolls` guarda `profile_name`
como snapshot.
