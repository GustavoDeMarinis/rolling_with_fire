# Riesgos — Rolling With Fire

Registro vivo. Los riesgos que el stack elegido en D12/D13 eliminó no figuran acá;
al final se anota cuáles fueron y por qué, para no volver a discutirlos.

- 🔴 Alto impacto / alta probabilidad
- 🟡 Alto impacto / baja probabilidad
- 🟢 Bajo impacto / cualquier probabilidad

---

## Técnicos

| ID | Riesgo | Cat. | Impacto | Mitigación |
|----|--------|------|---------|------------|
| T1 | **La animación pelea con Flutter.** Es el riesgo que justifica el orden de `mvp.md` | 🟡 | Reevaluar D13 | M0 *es* el experimento: un APK que anima un dado contra Phoenix. Se sabe en días |
| T2 | **`reveal_at` mal dimensionado.** Un cliente con RTT atípico revela tarde, o todos esperan de más por uno | 🔴 | Se rompe la sincronía, que es el producto | Piso = duración de animación (~3s), que absorbe casi todo el RTT real. Medir el desfase efectivo entre dispositivos en M0 |
| T3 | **Reloj del cliente desincronizado.** El offset se estima mal y `reveal_at` cae en el pasado o muy lejos | 🟡 | Tiradas que no animan, o animan de más | 3–5 vueltas de `ping` al entrar, quedarse con el RTT mínimo. Degradado: `reveal_in_ms` relativo a la llegada |
| T4 | **Idempotencia insuficiente en reconexión.** Un reintento por timeout duplica la tirada | 🔴 | Historial corrupto, `seq` inflado | `client_roll_id` como clave de idempotencia, verificada antes de tirar. Test explícito de reconexión en M3 |
| T5 | **Fuga por visibilidad.** Una tirada oculta llega completa a quien no debe verla | 🔴 | Se rompe la confianza del GM | Filtrado en `handle_out/3`, nunca en la UI (D5). Verificado con dos clientes conectados en M4 |
| T6 | **Sin rate limiting, el evento `roll` se abusa** | 🟡 | CPU y crecimiento de tabla | Límite por usuario en el channel, antes de salir de la LAN |
| T7 | **Migraciones de Ecto con datos reales** | 🟡 | Downtime o pérdida | Backup obligatorio antes de cada deploy. Mientras haya un solo VPS, snapshot previo |
| T8 | **Tabla `rolls` crece sin límite.** Una campaña larga son decenas de miles de filas | 🟡 | Historial lento | Índice `(room_id, seq DESC)` desde el día uno. Particionar por `rolled_at` recién si aparece el problema |
| T9 | **Sin experiencia mobile previa.** Navegación, ciclo de vida, builds y firma de APK son territorio nuevo | 🟢 | Fricción en M0 | Es costo de aprendizaje, no incertidumbre. Las cuatro trampas conocidas de M0 ya están anotadas en `mvp.md` |

---

## Producto

| ID | Riesgo | Cat. | Impacto | Mitigación |
|----|--------|------|---------|------------|
| P1 | **El mercado de apps de dados está saturado** | 🟡 | Crecimiento orgánico nulo | El diferencial es el broadcast sincronizado, no el dado. Si el MVP se siente igual que la competencia, la tesis falló. Validar con 50 usuarios reales |
| P2 | **La sincronía no se percibe.** El usuario no nota la diferencia con "todos ven el resultado enseguida" | 🔴 | El diferencial central no vende | Es lo primero a probar con jugadores reales, no lo último. Basta una sesión con cuatro personas |
| P3 | **Sin diseñador, la primera impresión sufre** | 🟡 | Retención inicial baja | Concentrar el presupuesto visual en la animación y el desglose; el resto sobrio y consistente |
| P4 | **Un solo desarrollador.** Todo el proyecto es una persona | 🔴 | El calendario depende de una agenda | El orden por miedo existe para esto: si el proyecto muere, muere sabiendo si la tesis servía |

---

## Financieros y de escala

| ID | Riesgo | Cat. | Impacto | Mitigación |
|----|--------|------|---------|------------|
| F1 | **Las donaciones no aparecen** | 🟡 | No hay revenue | El costo base es < $20/mes. Aguanta indefinidamente; lo que se pierde es la capacidad de crecer, no el servicio |
| F2 | **Crecimiento antes de tener con qué pagarlo** | 🟢 | Sube el costo operativo | Un WebSocket ocioso en BEAM cuesta casi nada. El techo del VPS único está muy por encima de donde estará el problema real |
| F3 | **VPS único = single point of failure** | 🟡 | Downtime sin HA | Snapshot diario. Para una app de dados, un downtime corto no es existencial |
| F4 | **Cuenta de desarrollador de Play Store** | 🟢 | Costo fijo de entrada | Es un pago único de USD 25. Contemplado |

---

## Decisiones abiertas de riesgo

- [ ] **T2 / T3:** medir en M0 el desfase real de reveal entre dos dispositivos, y
      ajustar el margen con ese número en vez de estimarlo.
- [ ] **T6:** fijar el límite concreto del evento `roll` antes de exponer el servidor
      fuera de la LAN.
- [ ] **D15:** elegir el mecanismo de autenticación. Misma línea: antes de salir de
      la LAN. Lean actual, Google Sign-In.

---

## Riesgos eliminados por decisión

No reabrir sin revertir la decisión que los eliminó.

| Riesgo previo | Eliminado por |
|---|---|
| Expo requiere eject para Google Play Billing | D12 + D13: no hay Billing, y no hay Expo |
| Socket.IO no escala multi-instancia sin Redis Adapter | D12: PubSub sobre Erlang nativo |
| Socket.IO degrada con muchos rooms concurrentes en VPS chico | D12: BEAM |
| JWT refresh token sin revocación | D15: `Phoenix.Token` con `max_age`, y el mecanismo aún no se eligió. Vuelve a evaluarse cuando se elija |
| JSONB en `result` no indexable | D11: `total` es columna real |
| Skins cosméticas no incentivan el pago | D12: no hay skins |
| Socio UX/UI negocia equity | No hay socio |
