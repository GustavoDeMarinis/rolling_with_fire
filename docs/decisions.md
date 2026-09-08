# Decisiones — MVP

> Registro de decisiones tomadas. Es la fuente de verdad: si otro documento del
> repo lo contradice, gana este (ver "Estado de la documentación" al final).

---

## Tesis del producto

El **broadcast instantáneo de la tirada es el producto**, no una feature de la
tercera versión. Todo lo demás (historial, macros, permisos) existe para sostener
ese momento: cinco personas mirando el mismo dado caer al mismo tiempo.

Corolario: las salas live entran en el MVP. El roadmap previo las ponía en V3
(mes 7–9), después de dos versiones que habrían sido un dice roller genérico más
— exactamente el mercado saturado que anota el riesgo P1.

---

## D1 — Las tiradas se resuelven en el servidor

El cliente manda la intención, nunca el resultado. RNG con CSPRNG
(`:crypto.strong_rand_bytes` / `:rand` sembrado), persistido antes de emitir.

No es (solo) anti-trampa: es tener **un único resultado canónico**. No hay
conflictos que resolver ni orden que reconciliar.

## D2 — El reveal se sincroniza por plazo, no por confirmación

El servidor manda el resultado junto con un `reveal_at`. Cada cliente anima hasta
ese instante y recién ahí muestra el número.

Descartado: barrera de ACKs (esperar el "recibido" de todos y después mandar el
"go"). El mensaje de "go" viaja con la misma latencia variable que el resultado,
así que el desfase del reveal es idéntico — pagando dos hops extra. Peor: el
teléfono dormido de cualquier jugador haría que **toda** tirada espere el timeout.

El servidor dimensiona el plazo con el RTT que ya conoce por los heartbeats del
channel: `reveal_at = now + max(RTT de la sala) + margen`. Se obtiene la garantía
que buscaba la barrera ("nadie revela antes de que todos lo tengan") sin preguntar.

**Degradado:** si el mensaje llega con `reveal_at` ya vencido, la tirada entra al
log sin animación. El mismo campo cubre gratis el replay de reconexión — 12
tiradas perdidas se renderizan sin animar ninguna.

## D3 — Orden de operaciones: izquierda a derecha, estricto

Una tirada es una **lista ordenada de pasos**, cada uno aplicado al acumulado. Sin
precedencia matemática, sin paréntesis, sin parser.

`2d6 Espada + 1d6 Fuego + 3 Fuerza * 2 Crítico + 16 Rage` es ambiguo con
precedencia estándar (el `*2` multiplicaría solo al 3). Con pasos ordenados el
usuario decide moviendo el paso: si el crítico duplica los dados pero no los
modificadores (como D&D), pone el `*2` antes de Fuerza.

El acumulador arranca en 0; el primer paso es siempre aditivo.

## D4 — Las variables son del perfil (`profiles`)

Entidad `profiles` desde el MVP. `UNIQUE(profile_id, name)`.

Un **perfil** es un paquete con nombre de variables + templates que asumís en una
sala. No presupone jugador: sirve para un PJ, un NPC, un monstruo o un build. El
caso que lo obliga es el GM, que no tiene "personaje" pero sí quiere un perfil por
monstruo — con `characters` tendría que inventar personajes falsos.

> Se evaluó `actors`, que es la abstracción exacta (algo que actúa y tira). Se
> descarta por colisión de vocabulario: en el BEAM "actor" es el modelo de
> concurrencia, y un schema `Actor` al lado de procesos actores es ambigüedad
> gratuita en cada conversación sobre el código. El nombre del modelo tampoco ata a
> la UI, que puede etiquetarlo como quiera.

**El problema:** una persona juega varios personajes. Gustavo tiene a Jorge el
Guerrero (`FUERZA = 5`) los martes y a Mira la Maga (`FUERZA = 1`) los sábados, y
es un solo `users`. Con variables del usuario hay un solo `FUERZA`: para jugar los
dos hay que llamarlas `FUERZA_JORGE` y `FUERZA_MIRA` y arrastrar el prefijo a cada
paso de cada tirada — el usuario haciendo a mano lo que el modelo no hace.

**Por qué no `room_ids[]` en la variable:** el array no da alcance, da
visibilidad. Filtra en qué salas *aparece* la variable, pero no garantiza "en esta
sala FUERZA vale 5" — para eso harían falta dos filas llamadas `FUERZA` y una
restricción de no-solapamiento (`EXCLUDE USING gist` sobre arrays), que no es algo
para mantener en un MVP. Costaría un paso extra hoy sin resolver el problema, y la
migración a perfiles llegaría igual.

**Por qué ahora y no después:** hoy no hay usuarios ni código. Escribir `profiles`
una vez cuesta menos que escribir la versión user-scoped y migrar más tarde (dos
tablas re-parenteadas, backfill, y un concepto nuevo metido en una UI viva).

**Fricción de onboarding:** cero, con el mismo truco de D8 — al registrarse se crea
un perfil por defecto con el username. Invisible hasta que hace falta el segundo.

**Corolario — los pasos referencian la variable por nombre, no por id.** El template
resuelve `FUERZA` contra el perfil activo de esa sala
(`room_members.active_profile_id`), así "Daño Espada Larga" sirve para Jorge y para
Mira sin duplicarse. Si el perfil activo no tiene esa variable, la tirada falla con
un mensaje explícito; nunca asume 0.

Un perfil es entonces un paquete portable (variables + templates), que es
exactamente el "Jorge el Guerrero exportable entre salas" del pedido original.

## D5 — Tiradas ocultas: se declaran al tirar

La visibilidad se elige **antes** de tirar, no se esconde después.

| `visibility` | Ve el resultado | Quién puede usarla |
|---|---|---|
| `everyone` | Toda la sala | Todos |
| `admins` | El que tiró + los admins | Todos (tirada privada de jugador) |
| `self` | Solo el que tiró | Solo admins |

Los jugadores pueden ocultar a otros jugadores, pero **nunca a los admins** — como
en la mesa real, el GM ve tu tirada. Solo un admin puede tirar oculto a todos.

`leaves_trace` (booleano, por tirada) decide si los que no pueden ver el resultado
igual ven "Fulano hizo una tirada oculta". Es el ruido de los dados detrás del
biombo, y a veces se quiere y a veces no.

> **El filtrado va server-side, por destinatario.** Mandar la tirada completa a
> todos y esconderla en la UI la deja legible para cualquiera con devtools. En
> Phoenix esto es `intercept` + `handle_out/3`: cada socket decide, con sus propios
> assigns, si recibe la tirada completa, la versión redactada o nada.

## D6 — Ingreso a la sala por código de invitación

Un código por sala, rotable por cualquier admin. Quien lo tiene, entra.

**Consecuencia:** desaparece la bandeja de invitaciones pendientes con
aceptar/rechazar. No hay entidad `invitations`, ni pantalla, ni notificación. Es
una simplificación grande y además es lo único testeable el día 1 sin un
directorio de usuarios.

Descartado: contraseña de sala compartida (`rooms.password_hash` del doc previo) y
salas públicas con lobby.

**Futuro:** un `/invite` dirigido a un usuario, con bandeja de aceptar/rechazar,
convive con el código sin reemplazarlo. Fuera del MVP.

## D7 — Roles: solo `room_members.role`, sin dueño

No hay `rooms.owner_id`. El creador entra como `admin` y puede dejar de serlo como
cualquier otro.

**Invariante: toda sala tiene mínimo un admin.** No se puede degradar ni salir si
sos el último — hay que promover a alguien primero. El último admin que quiere
irse **borra la sala** (soft delete: `archived_at`, el historial se preserva).

## D8 — Toda tirada pertenece a una sala

`rolls.room_id` es `NOT NULL`. No existe la tirada personal suelta.

Para no perder el caso "quiero tirar solo": al registrarse, cada usuario recibe
una **sala personal** automática (`is_personal`, un solo miembro, sin código de
invitación). Un solo modelo, los dos usos.

Esa sala es un **playground**: no se puede invitar a nadie, no se puede rotar
código (no tiene), no se puede archivar. Sirve para probar una tirada sin público.

Descartado: el modo offline-first del doc previo. Era contradictorio con D1 — una
tirada encolada sin red se resuelve minutos después, cuando ya no significa nada.

## D9 — Al expulsar, el historial queda

Las tiradas de un miembro expulsado siguen en el log de la sala. Es el registro
histórico de la campaña, no le pertenece a él.

## D10 — Paginación por cursor, nunca por offset

El historial usa `seq` (contador monótono por sala), no `page`/`limit`.

Con tiradas entrando en vivo, los offsets se corren bajo los pies del scroll
infinito: filas duplicadas y filas salteadas. El mismo cursor sirve para el replay
al reconectar (`after_seq`) — un solo mecanismo para las dos cosas.

## D11 — `total` es columna, no un campo del JSONB

`rolls.total INT NOT NULL`, aparte del `breakdown JSONB`.

La razón fuerte no es evitar aritmética en el front (es trivial): una columna real
es **indexable y consultable** — estadísticas de campaña, "todos los 20 naturales",
promedios por jugador. En JSONB eso no sale sin índices GIN, que era un riesgo
anotado del diseño anterior.

## D12 — Stack

- **Backend: Elixir + Phoenix.** Channels/PubSub/Presence son literalmente esto.
  Un WebSocket ocioso en BEAM cuesta casi nada, que es lo que hace viable un
  producto gratis masivo financiado por donaciones.
- **Sin Redis.** PubSub distribuye sobre Erlang nativo. Elimina de un saque los dos
  riesgos que traía Socket.IO: el Redis Adapter para multi-instancia y la
  degradación con muchas salas concurrentes en un VPS chico.
- **Monetización: donaciones**, no skins ni Google Play Billing. Elimina el eject de
  Expo, que era el 🔴 más caro del registro de riesgos.
- **Mobile: Flutter** (ver D13).

## D13 — Mobile: Flutter

Retracta una recomendación previa de React Native, cuyo argumento central era "no
aprender dos lenguajes a la vez". La premisa era falsa: el autor **ya sabe Elixir**.
También se descarta el argumento de la Vibration API — es una limitación de Safari
en iOS (camino PWA), no de RN, y además la háptica es circunstancial para este
producto.

Rehecho el análisis contra los tres criterios reales — **liviandad, velocidad,
animación**:

**El trade-off real no es "bonito o no", es 2D contra 3D con física.**

| | Flutter | React Native |
|---|---|---|
| Animación 2D (giro, glow, partículas) | Terreno propio: Skia/Impeller + Rive | Muy bueno: Reanimated + Skia |
| 3D con física real | Punto flojo, inmaduro | `react-three-fiber` sobre Three.js, camino transitado |
| Liviandad | AOT a ARM, sin runtime de JS. ~15MB con split por ABI | Runtime JS (Hermes). ~25–40MB |
| Gama baja | Mejor | Anda, con más memoria |

Un dice roller necesita 2D. Y como "corre en cualquier teléfono" es el pitch
declarado del producto, se elige el que es efectivamente mejor en eso. La ventaja
de RN ("React ya lo sé") cubre menos superficie de la que parece: sin experiencia
mobile previa, navegación, ciclo de vida, builds nativos y tiendas hay que
aprenderlos igual.

**Riesgo asumido:** si alguna vez se quieren dados 3D con física real, es el único
punto donde se pelea con el framework. Mitigación: Rive produce dados de aspecto
tridimensional convincentes a una fracción del costo, que es lo que hacen casi
todas las apps de dados pulidas.

**Validación:** M0 *es* el experimento. Un APK que se conecta a Phoenix y anima un
dado. Si la animación pelea, se pierden días, no meses, y se sabe con las manos.

## D14 — La animación es coreografía, no física

**Nada de simulación física.** No es solo simplificar: la física es incompatible
con D1. El servidor ya decidió que salió 14, así que una simulación real tendría
que resolver hacia atrás qué impulso inicial aterriza en esa cara, o hacer un snap
final que se ve falso. Conociendo la respuesta, se anima *hacia* ella.

**Nada de 3D en runtime tampoco.** Durante el giro el dado va tan rápido que no se
lee ninguna cara — entonces no hay nada que rotar en 3D. El volumen es un problema
de *aspecto*, resuelto offline en Blender, no de motor.

**Assets:** por tipo de dado, un loop de giro borroneado (uno solo, agnóstico del
resultado) más una imagen por cara para el asentamiento. Seis loops y ~60 imágenes
en total, con motion blur calculado por el renderer. En el dispositivo solo hay un
sprite y su transformada: la implementación más liviana posible, que es el
diferenciador declarado del producto. Por eso esto **no reabre D13**.

### Las cuatro fases

1. Aparición y aceleración; el blur crece.
2. Giro sostenido y borroneado — **duración elástica**.
3. Frenado y asentamiento en la cara que salió, con *overshoot* (escala 1.15 → 1.0,
   easing elástico: eso es lo que da la sensación de impacto).
4. Pop del número y efectos; shake de dos frames en un crítico.

**La fase 2 es la implementación de D2.** El segmento de duración variable es donde
se esconde el viaje de red — el colchón de latencia no se agrega, ya está ahí.

**Y permite animación optimista:** como durante el blur no hay nada legible, el
cliente empieza a girar en el mismo frame del tap y el resultado recién importa en
el asentamiento.

### Secuencia y aceleración

Los dados se revelan de a uno, y cada uno más rápido que el anterior:

```
duración(i) = max(MIN_MS, BASE_MS * decay^i)
```

Con `BASE=900ms`, `decay=0.75`, `MIN=180ms`, diez dados terminan en ~3,7s. La
fórmula es **determinista**, así que todos los dispositivos que arrancan juntos en
`reveal_at` recorren la misma secuencia en sincronía sin un mensaje más.

### Dónde va el presupuesto de animación

En los modificadores, no en el dado. Lo que hace buena la tirada de Baldur's Gate 3
no son los polígonos: es que después de que el dado cae, los modificadores entran de
a uno —`+3 Fuerza`… `+2 Rage`— y el total se arma a la vista.

Eso ya está modelado: es `breakdown.steps[].acc`, paso por paso con su nombre y su
acumulado. El desglose que se diseñó para que el front no calcule nada resulta ser
el guion de la animación.

> BG3 usa 3D real y sí muestra las caras tumbando. Esto es una simplificación
> deliberada, no una réplica.

Y el sonido: un clac seco al asentar rinde más que cualquier efecto visual y cuesta
un archivo de 20kb.

## D15 — Auth: la forma se fija ahora, el mecanismo después

Auth **va sí o sí**. Lo que se difiere es el mecanismo (contraseña, Google, OAuth,
magic link), porque no hay todavía nada que proteger. Para que diferirlo no cueste
retrabajo, se fijan ahora cuatro cosas que ningún mecanismo cambia.

### 1. Identidad y credencial son cosas distintas

`users` guarda **quién sos** (id estable, username) y nada sobre cómo lo probás. La
credencial —hash de contraseña, `sub` de Google, lo que sea— cuelga aparte:

```
user_identities (user_id, provider, provider_uid, secret_hash)
--  provider: 'password' | 'google' | 'apple' | …
```

Con esta separación, sumar Google es **una fila nueva**, no una migración de
`users`. Sin ella, elegir el mecanismo tarde significa reestructurar la tabla de la
que cuelga todo el resto del sistema.

> La tabla se **escribe cuando llegue el mecanismo**, no antes. Lo que se decide hoy
> es la forma, no el código: `users` no lleva ni `email` ni `password_hash`, para no
> comprometerse con un mecanismo por omisión.

### 2. Toda la superficie de auth es una función

`UserSocket.connect/3` traduce token → `user_id`. De ahí en adelante todo lee
`socket.assigns.current_user_id` y **nadie sabe cómo llegó ahí**. Cambiar JWT por
OAuth toca esa función y nada más.

Es lo que permite que M0 y M1 corran con identidad de desarrollo (elegís username,
sin contraseña) sin que eso sea deuda: es la misma función, con otro cuerpo.

### 3. El contrato es un token, no una cookie

Una app mobile sobre WebSocket presenta un token en los params del socket. Todo
mecanismo termina igual: **acuñando un token**. Por eso el formato se puede fijar
hoy aunque el mecanismo no.

Se usa **`Phoenix.Token`**: viene en el framework, firma con el secreto de la app,
tiene `max_age`, cero dependencias. Lo acuña la identidad de desarrollo hoy y lo
acuña un login real de Google mañana, sin que el cliente note la diferencia.

### 4. La autorización NO se difiere

Diferir el mecanismo de autenticación no difiere los permisos. Membresía de sala,
roles y visibilidad de tiradas (D5, D7) ya están decididos y se verifican
server-side. Eso es lo que protege los datos, y no depende de cómo iniciás sesión.

### Cuándo se decide el mecanismo

**Antes de que el servidor salga de la LAN.** Mientras es una PC y un celular en la
misma wifi, la identidad de desarrollo alcanza. El primer usuario que no sea el
autor es la línea.

> Lean para ese momento, no decisión: **Google Sign-In**. En Android es un toque,
> y evita construir recuperación de contraseña, envío de mails y la
> responsabilidad de custodiar contraseñas. En una app gratuita, cada paso de
> registro es gente que no llega.

---

## Estado de la documentación

La planificación previa (roadmap V1–V4, epics, stack Node/Express/Prisma/Socket.IO)
fue **eliminada del repo**: contradecía estas decisiones o duplicaba `mvp.md`. Vive
en el historial de git, hasta el commit `046e4a8`.

Los documentos vigentes no se pisan entre sí:

| Documento | Qué fija |
|---|---|
| `decisions.md` | Este. D1–D15: qué se decidió y por qué |
| `mvp.md` | Alcance y orden de trabajo (M0–M4) |
| `data-model.md` | Schema, formato de `steps` y de `breakdown` |
| `realtime-contract.md` | Eventos del channel `room:{id}` |
| `architecture.md` | Stack, procesos, seguridad, despliegue |
| `product-vision.md` | Para quién es, cómo se diferencia, cómo se sostiene |
| `risks.md` | Registro de riesgos vivos |
| `metrics.md` | Qué se mide y con qué umbral de acción |
