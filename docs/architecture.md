# Arquitectura — Rolling With Fire

Stack, procesos y despliegue. **No repite** lo que ya está fijado en otro lado:

- El schema vive en `data-model.md`.
- Los eventos del channel viven en `realtime-contract.md`.
- El *por qué* de cada elección vive en `decisions.md` (D1–D15).

---

## Forma general

Un monolito Phoenix y una app Flutter. Una sola conexión: el WebSocket.

```
Flutter (Android)  ──── WebSocket ────>  Phoenix
                                          ├── UserSocket.connect/3   token → user_id (D15)
                                          ├── RoomChannel            un canal por sala
                                          ├── Room GenServer         uno por sala activa
                                          └── Ecto ──> PostgreSQL
```

No hay API REST. La tirada viaja por el channel, no por `POST /rolls`: mandarla por
HTTP le entrega el resultado al que tiró antes que a nadie, que es exactamente lo
que D2 evita.

---

## Stack

### Backend — Elixir + Phoenix (D12)

Channels, PubSub y Presence son literalmente el producto, no una librería que hay
que elegir. Un WebSocket ocioso en BEAM cuesta casi nada, y eso es lo que hace
viable una app gratis sostenida por donaciones.

| Componente | Elección |
|---|---|
| Runtime | Elixir sobre BEAM |
| Web / tiempo real | Phoenix Channels + PubSub + Presence |
| Persistencia | PostgreSQL vía Ecto |
| RNG | `:crypto.strong_rand_bytes` (CSPRNG), server-side siempre (D1) |
| Tokens | `Phoenix.Token` — sin dependencias (D15) |
| Pub/Sub distribuido | Erlang nativo. **Sin Redis** |

### Mobile — Flutter (D13)

AOT a ARM, sin runtime de JS: ~15MB con split por ABI, y mejor comportamiento en
gama baja. "Corre en cualquier teléfono" es el pitch del producto, así que se elige
el que efectivamente es mejor en eso.

La animación es coreografía 2D, no física ni 3D en runtime (D14): un sprite y su
transformada. Assets producidos offline — un loop de giro borroneado por tipo de
dado, más una imagen por cara.

Android primero. iOS está fuera del MVP.

### Infraestructura

Un VPS, Docker, Postgres al lado, Nginx con TLS de Let's Encrypt terminando el
WebSocket. Es suficiente y sigue siéndolo por mucho más tiempo del que suele
asumirse.

Antes de eso, M0 corre en la LAN de casa: Phoenix en la PC, APK en el celular.

---

## Procesos: un GenServer por sala

Arranca en el primer join, termina en el último leave (con hibernate de gracia para
no rebotar en reconexiones).

Lo que gana estando serializado en un solo proceso:

- **`seq` sin contención.** Todas las tiradas de la sala pasan por el mismo proceso,
  así que el contador monótono se asigna en orden por construcción. La columna
  `rooms.next_seq` existe igual, para que el número sobreviva a la muerte del
  proceso.
- **RTT de la sala.** Los heartbeats del channel dan la latencia observada de cada
  miembro; el proceso la tiene junta y de ahí sale
  `reveal_at = now + max(RTT) + margen` (D2).

**El proceso es coordinación en memoria; Postgres es la verdad.** Una sala sin nadie
conectado deja de existir como proceso y no pierde nada.

---

## Camino de una tirada

1. El cliente manda `roll` por el channel con un `client_roll_id` (idempotencia).
2. El servidor resuelve las variables contra el perfil activo del que tira en esa
   sala (D4). Si falta una, error explícito: no se tira.
3. Evalúa los pasos de izquierda a derecha (D3), con CSPRNG.
4. Persiste: `total` como columna, `breakdown` como JSONB, `seq` asignado (D10, D11).
5. Calcula `reveal_at` y broadcastea.
6. `handle_out/3` decide, por socket, si va la tirada completa, la redactada o nada
   (D5).
7. Cada cliente anima hasta `reveal_at` y revela ahí.

El cliente puede arrancar el giro en el frame del tap: durante el blur no hay nada
legible, así que el resultado recién importa en el asentamiento (D14).

---

## Seguridad

| Capa | Mecanismo |
|---|---|
| Identidad | Token en los params del socket, resuelto en `connect/3` (D15) |
| Credencial | Fuera de `users`, en `user_identities`. Mecanismo diferido (D15) |
| Autorización | **No diferida.** Membresía, rol y visibilidad se verifican server-side |
| Ingreso a sala | Código de invitación rotable, no contraseña compartida (D6) |
| Tiradas ocultas | Filtrado por destinatario en `handle_out/3`, nunca en la UI (D5) |
| Generación de dados | CSPRNG server-side. No se expone seed ni estado (D1) |
| Rate limiting | Por usuario en el evento `roll` del channel |
| SQL injection | Ecto con queries parametrizadas |
| Origins | `check_origin` estricto en producción (en dev, `false` — ver M0) |

Diferir el mecanismo de autenticación **no** difiere los permisos: eso es lo que
protege los datos, y no depende de cómo iniciás sesión.

---

## Decisiones técnicas con impacto futuro

### 1. UUIDs, no auto-increment
Permite merge de datos y no expone contadores de recursos. La excepción deliberada
es `rolls.seq`, que es un contador **por sala** y existe justamente para ser
ordenable (D10).

### 2. Soft delete
`rooms.archived_at`. Preserva la integridad referencial del historial de tiradas: la
campaña es el registro, y borrarla en duro se lleva puesto el log (D7, D9).

### 3. JSONB para `steps` y `breakdown`, columna para `total`
La variabilidad de configuraciones de dados es alta y normalizarla ahora es YAGNI.
Pero `total` va como columna real: es lo que hace consultables las estadísticas de
campaña sin índices GIN (D11).

### 4. Snapshots en `rolls`
`template_name`, `profile_name` y el valor de cada variable se **copian** a la tirada.
Borrar una macro o subir FUERZA no puede reescribir el historial. Un log que se
reescribe solo no es un log.

### 5. Monolito modular
El equipo es una persona. La complejidad operacional de separar servicios es un
riesgo mayor que el acoplamiento controlado. Si alguna vez hay que escalar
horizontalmente, PubSub sobre Erlang distribuido ya cubre el caso sin Redis.

---

## Estructura del backend

```
lib/
  rolling_with_fire/          # dominio, sin Phoenix
    accounts/                 # users, identidades, tokens
    profiles/                 # perfiles, variables, templates
    rooms/                    # salas, membresía, roles, invite codes
    rolls/                    # evaluación de pasos, RNG, persistencia
  rolling_with_fire_web/
    channels/
      user_socket.ex          # connect/3: toda la superficie de auth (D15)
      room_channel.ex         # join, roll, history, resume, admin
    presence.ex
  room_server.ex              # GenServer por sala: seq, RTT, reveal_at
```

La evaluación de una tirada es una función pura sobre `steps` + variables resueltas.
El RNG entra como parámetro, así que se testea con valores fijos.
