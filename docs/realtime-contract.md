# Contrato de tiempo real — `room:{room_id}`

Un solo canal por sala. **La tirada va por el channel, no por HTTP.** El doc previo
la mandaba por `POST /rolls` y emitía el evento aparte; eso le entrega el resultado
al que tiró antes que a nadie, que es justo lo que D2 evita.

---

## Join

**Cliente → Servidor:** `join("room:{id}", %{})`
El usuario sale del token del socket (`connect/3`), no del payload.

**Respuesta:**
```json
{
  "room":        { "id": "…", "name": "…", "invite_code": "…" },
  "me":          { "user_id": "…", "role": "admin",
                   "profile": { "id": "…", "name": "Jorge el Guerrero" } },
  "server_time": 1725740000000,
  "latest_seq":  412
}
```

`server_time` arranca la estimación de offset de reloj. `latest_seq` le dice al
cliente si quedó atrás y tiene que pedir `resume`.

`Phoenix.Presence` entra acá gratis: `presence_state` / `presence_diff` resuelven
"quién está en la mesa" sin escribir nada.

---

## Sincronización de reloj

**Cliente → Servidor:** `ping` `{ "client_time": … }`
**Servidor → Cliente:** `{ "client_time": …, "server_time": … }`

3–5 vueltas al entrar. Se descarta todo menos la muestra de **RTT mínimo** (lo que
hace NTP) y se estima:

```
offset = server_time - (client_time_recv - rtt/2)
```

Si no se logra estimar, el cliente ignora `reveal_at` y usa `reveal_in_ms` relativo
a la llegada. Degrada, no rompe.

---

## Tirar

**Cliente → Servidor:** `roll`
```json
{
  "client_roll_id": "uuid-v4",
  "template_id":    "…",
  "steps":          [ … ],
  "visibility":     "everyone",
  "leaves_trace":   true
}
```

- `template_id` **o** `steps`, no ambos. Sin `template_id` es una tirada ad-hoc.
- Las variables de los pasos se resuelven contra el perfil activo del que tira en
  esa sala (D4). Si le falta una, el reply es un error explícito y no se tira.
- `client_roll_id` es la **clave de idempotencia**: si el cliente reintenta por
  timeout, el servidor devuelve la tirada ya creada en vez de tirar de nuevo. Sin
  esto, cada reconexión duplica tiradas.
- El servidor valida que `visibility: "self"` solo la pueda pedir un admin (D5).

**Reply:** `{ "roll_id": "…", "seq": 413 }` — solo el acuse. El resultado llega
por el broadcast, igual que a todos.

---

## Recibir

**Servidor → Cliente:** `roll:new`
```json
{
  "id": "…", "seq": 413,
  "user":  { "id": "…", "username": "tika" },
  "template_name": "Daño Espada Larga de Fuego",
  "total": 43,
  "breakdown": { "steps": [ … ] },
  "reveal_at":    1725740003200,
  "reveal_in_ms": 3000,
  "rolled_at":    1725740000200
}
```

El cliente anima hasta `reveal_at` (o `reveal_in_ms` si no tiene offset). Si ya
venció: al log, sin animación.

`reveal_at = now + max(RTT observado en la sala) + margen`, con un piso que es la
duración deseada de la animación (~3s).

### Versión redactada

Para quien no puede ver el resultado pero la tirada tiene `leaves_trace: true`:

```json
{ "id": "…", "seq": 413, "user": {…}, "hidden": true, "rolled_at": … }
```

Sin `total`, sin `breakdown`, sin `template_name`. Con `leaves_trace: false` no se
emite nada a ese destinatario.

> **Implementación:** `intercept ["roll:new"]` + `handle_out/3`. Cada socket decide
> con sus propios assigns (`user_id`, `role`) si le corresponde la versión completa,
> la redactada o ninguna. El filtrado nunca es del cliente (D5).

---

## Historial y reconexión

| Evento | Dirección | Payload | Para qué |
|---|---|---|---|
| `history` | C→S | `{ before_seq, limit }` | Scroll infinito hacia atrás |
| `resume`  | C→S | `{ after_seq }` | Recuperar lo perdido mientras estaba desconectado |

Las dos devuelven `{ rolls: [...], has_more: bool }`, ya filtradas por
visibilidad para ese usuario. Todas vienen con `reveal_at` vencido, así que
ninguna anima.

---

## Administración

| Evento | Dirección | Notas |
|---|---|---|
| `member:kick` | C→S | Solo admin |
| `member:set_role` | C→S | Solo admin. Rechaza degradar al último admin (D7) |
| `room:rotate_code` | C→S | Solo admin. No aplica a la sala personal |
| `member:set_profile` | C→S | Cambiar el perfil activo propio en esta sala |
| `member:joined` / `member:left` | S→C | |
| `member:kicked` | S→C | El expulsado recibe el evento y el servidor cierra su socket |
| `member:role_changed` | S→C | |
| `room:archived` | S→C | Desconecta a todos |

---

## Ciclo de vida de la sala

Un GenServer por sala, arrancado al primer join y terminado al último leave (con
un hibernate de gracia para no rebotar en reconexiones).

**El proceso es coordinación en memoria; Postgres es la verdad.** Una sala sin
nadie conectado deja de existir como proceso y no pierde absolutamente nada.
