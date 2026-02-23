# Risks — Rolling With Fire

## Categorías

- 🔴 Alto impacto / alta probabilidad
- 🟡 Alto impacto / baja probabilidad
- 🟢 Bajo impacto / cualquier probabilidad

---

## Riesgos Técnicos

| ID | Riesgo | Categoría | Impacto | Mitigación |
|----|--------|-----------|---------|------------|
| T1 | Expo requiere eject para Google Play Billing (V4) | 🔴 | Retrasos de 2-4 semanas | Planificar custom dev client desde V3; no usar Expo Go en producción |
| T2 | Socket.IO no escala en VPS pequeño con muchos rooms concurrentes | 🟡 | Degradación en V3 | Monitorear conexiones activas; upgrade vertical de VPS antes del problema |
| T3 | PostgreSQL en mismo servidor que API genera contención de recursos | 🟡 | Latencia alta en picos | Separar DB a VPS propio cuando DB > 5GB o CPU > 70% sostenido |
| T4 | JSONB en `rolls.result` no indexable sin diseño explícito | 🟢 | Queries lentas en historial grande | Agregar índices GIN cuando el historial supere 100k filas |
| T5 | JWT Refresh Token sin revocación puede ser explotado | 🔴 | Sesiones comprometidas | Implementar refresh token rotation y tabla de tokens revocados desde V1 |
| T6 | Rate limiting insuficiente en endpoints de roll puede ser abusado | 🔴 | Costo de servidor inflado | Aplicar rate limiting por usuario/IP desde V1 en `/rolls` |
| T7 | Migración de esquema Prisma con datos reales en producción | 🟡 | Downtime o pérdida de datos | Estrategia de migraciones con backup obligatorio antes de cada deploy |

---

## Riesgos Financieros

| ID | Riesgo | Categoría | Impacto | Mitigación |
|----|--------|-----------|---------|------------|
| F1 | Crecimiento de usuarios supera capacidad del VPS inicial sin revenue | 🟡 | Costo operativo sube antes de monetizar | Definir umbrales: si > 5k usuarios activos antes de V4, evaluar upgrade |
| F2 | Google Play pide cuenta de desarrollador y comisión 15-30% | 🟢 | Reducción de margen en V4 | Contemplar en modelo de precios; considerar donaciones directas como alternativa inicial |
| F3 | Socio UX/UI negocia equity en vez de sueldo — impacto en estructura | 🟡 | Complejidad legal/societaria | Definir acuerdo claro antes de que el socio empiece a trabajar |
| F4 | La app no logra retención suficiente para justificar V3/V4 | 🔴 | Pivot o cierre | Establecer métricas de validación por versión (ver metrics.md); no avanzar sin evidencia |

---

## Riesgos de Escalabilidad

| ID | Riesgo | Categoría | Impacto | Mitigación |
|----|--------|-----------|---------|------------|
| E1 | Socket.IO no soporta multi-instancia sin Redis Adapter | 🔴 | Live rooms rotas si se escala horizontalmente | Integrar Socket.IO Redis Adapter antes de agregar segundo nodo de backend |
| E2 | Historial de tiradas crece sin límite — tabla `rolls` se vuelve lenta | 🟡 | Queries lentas en historial largo | Particionamiento por `rolled_at` o límite de historial por sala configurable |
| E3 | Single VPS es single point of failure | 🟡 | Downtime sin HA | Snapshot diario del VPS; DNS failover manual como primera línea |
| E4 | Monolito difícil de extraer si hay que separar WebSocket server | 🟢 | Refactor costoso | Diseño modular interno desde V1 permite extraer el módulo de events sin romper el resto |

---

## Riesgos de Producto

| ID | Riesgo | Categoría | Impacto | Mitigación |
|----|--------|-----------|---------|------------|
| P1 | Mercado de apps de dados RPG es pequeño o saturado | 🟡 | Bajo crecimiento orgánico | Validar con 50 usuarios reales antes de V3; buscar nichos (streamer RPG, comunidades) |
| P2 | El Mobile Developer llega tarde o trabaja medio tiempo | 🔴 | V1 se retrasa significativamente | Documentar contratos de API antes de que empiece; el backend puede avanzar en paralelo |
| P3 | Usuarios esperan tiradas client-side por velocidad | 🟢 | UX percibida lenta | Mostrar animación de "tirando..." mientras se espera la respuesta del servidor (< 200ms en LAN) |
| P4 | Skins cosméticas no son suficiente incentivo de pago | 🟡 | Baja conversión en V4 | Investigar qué valoran los jugadores de rol antes de diseñar el catálogo de skins |
| P5 | UX sin diseñador en V1 daña la primera impresión | 🔴 | Mala retención inicial | Usar UI kit existente (React Native Paper o NativeBase) en V1; rediseñar con socio en V2 |

---

## Registro de Decisiones Abiertas de Riesgo

- [ ] **T5:** Implementar refresh token rotation en V1 o diferir a V2.
- [ ] **E1:** Decidir si Redis Adapter se agrega en V3 o se posterga hasta escalar horizontalmente.
- [ ] **P5:** Seleccionar UI kit para V1 antes de que el Mobile Developer empiece.
