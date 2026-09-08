# MVP — Alcance y orden de trabajo

El orden no es por capas, es **por miedo**: primero lo que más incomoda, porque es
lo que se posterga hasta que nunca se hace.

---

## M0 — El esqueleto que camina (APK + servidor local)

**Objetivo:** un APK instalado en el celular, hablándole a Phoenix corriendo en la
PC, por wifi de casa. Cero features.

- Phoenix con un `RoomChannel` de mentira: una sala hardcodeada, sin auth real.
- App **Flutter** (D13) con **un botón**: tira 1d20.
- El servidor resuelve, persiste, broadcastea con `reveal_at`.
- La animación corre y revela en el plazo.

**Criterio de salida:** dos dispositivos (o un celular y el navegador) ven la misma
tirada animar y revelar juntos, con el servidor en la LAN.

M0 es además el **experimento que valida D13**: si la animación pelea con Flutter,
se descubre acá, en días y no en meses.

### Las cuatro trampas de M0

Esto falla siempre por lo mismo, y el error nunca dice qué pasó:

1. **Phoenix escucha en `127.0.0.1`.** En `config/dev.exs`:
   `http: [ip: {0, 0, 0, 0}, port: 4000]`. Sin esto el celular no lo ve.
2. **`check_origin` rechaza el WebSocket** desde un origin no configurado.
   `check_origin: false` en dev.
3. **Android 9+ bloquea HTTP en claro por defecto.** Hace falta un
   `network_security_config.xml` que permita la IP de la LAN, o el socket falla sin
   mensaje útil.
4. **Usar la IP de la LAN, no `localhost`.** Celular y PC en la misma red.

---

## M1 — Usuarios y salas

- Auth (definición pendiente). Hasta que exista: identidad de desarrollo — elegís
  un username y te da un token, sin password. Aislada detrás de `connect/3` para
  que la auth real entre por un solo lugar.
- Crear sala. El creador entra como `admin`.
- Sala personal automática al registrarse: playground, sin invitados (D8).
- Código de invitación: unirse, rotar.
- Roles: promover, degradar, expulsar, con el invariante de mínimo un admin (D7).
- Presence: quién está conectado.

## M2 — Tiradas de verdad

- Perfiles (CRUD). Uno por defecto al registrarse, con el username.
- Selector de perfil activo por sala.
- Variables del perfil (CRUD).
- Templates con pasos ordenados (CRUD), evaluación izquierda a derecha (D3).
- Tirada ad-hoc sin template.
- Desglose completo en el feed: cada paso con su nombre, su valor y su acumulado.

## M3 — Historial

- Scroll infinito por cursor `seq` (D10).
- `resume` al reconectar.
- Idempotencia por `client_roll_id`.

## M4 — Tiradas ocultas

- `visibility` + `leaves_trace` al tirar (D5).
- Filtrado en `handle_out/3`, verificado con dos clientes conectados.

---

## Fuera del MVP

- Copiar/exportar un perfil entre salas de un toque. La entidad `profiles` entra en
  el MVP (D4), pero el import/export explícito no.
- `/invite` dirigido a un usuario con bandeja de aceptar/rechazar. Convivirá con el
  código de invitación, no lo reemplaza (D6).
- Visibilidad "oculta a todos salvo X jugador". Requiere lista de destinatarios
  además del enum de D5. Confirmada como feature futura, fuera del MVP.
- Push notifications. La vibración en foreground no necesita permisos; el push sí,
  y una sesión de rol tiene entre 50 y 200 tiradas — notificar cada una es la forma
  más rápida de que el usuario apague las notificaciones para siempre. Cuando
  entre, es para lo dirigido a esa persona: "el GM te pide una tirada", "es tu
  turno", "te invitaron a una sala".
- Donaciones. Nada de infraestructura de pagos antes de tener usuarios.
- Ventaja/desventaja, dados explosivos, "quedarse con los N más altos".
- División en los pasos (el redondeo es decisión del sistema de rol, no de la app).
- Salas públicas, lobby, directorio.
- iOS.

---

## Estado

**Sin preguntas abiertas.** Las quince decisiones están en `decisions.md`.

Lo único diferido es el **mecanismo** de autenticación (D15): la forma, el contrato
del token y la superficie de código ya están fijados, y M0/M1 corren con identidad
de desarrollo. La línea para elegirlo es antes de que el servidor salga de la LAN.
