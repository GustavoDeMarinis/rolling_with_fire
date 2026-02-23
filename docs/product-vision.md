# Product Vision — Rolling With Fire

## Vision Statement

Ser la herramienta de dados para rol más confiable, rápida y social del mercado Android hispanohablante, con un modelo de negocio sostenible desde el día uno basado en utilidad real, no en suscripciones forzadas.

---

## Horizonte: 12 meses

| Milestone | Objetivo |
|-----------|----------|
| M1 (mes 1-2) | App funcional offline, tiradas personales, historial local |
| M2 (mes 3-4) | Backend estable, autenticación, historial en nube |
| M3 (mes 5-6) | Salas asíncronas, grupos de juego persistentes |
| M4 (mes 7-9) | Salas live en tiempo real, presencia de usuarios |
| M5 (mes 10-12) | Monetización activa: skins, donaciones, Google Play Billing |

---

## Posicionamiento

**Para quién:** Jugadores de rol de mesa (D&D, Pathfinder, CoC, etc.) que juegan de forma remota o híbrida.

**Qué resolvemos:** El dolor de usar bots de Discord, apps genéricas de dados sin contexto de sala, o grupos de WhatsApp para coordinar tiradas.

**Cómo nos diferenciamos:**

- Tiradas generadas en **servidor** (anti-trampa verificable).
- Historial persistente de partida por sala.
- Configuraciones de dados compuestas guardadas (macros de tirada).
- UX pensada para rol de mesa, no para casino.
- Modelo freemium con valor real en el tier gratuito.

---

## Diferencial Competitivo

| Factor | Competidores típicos | Rolling With Fire |
|--------|----------------------|-------------------|
| Generación de dados | Client-side | Server-side (verificable) |
| Historial de sala | No o efímero | Persistente en DB |
| Macros de tirada | Raro | First-class feature |
| Anti-fraude | Inexistente | Arquitectura base |
| Monetización | Ads agresivos o paywall | Freemium ético |
| Mercado | Global genérico | Hispanohablante + global |

---

## Estrategia Bootstrap

### Principios de costos

1. **VPS único al inicio** — Un solo servidor cubre V1 y V2 cómodamente.
2. **PostgreSQL self-hosted** — Sin costos de DB administrada hasta escalar.
3. **Sin CDN hasta necesitarlo** — Assets estáticos servidos desde el VPS inicial.
4. **Expo (React Native)** — Un solo codebase mobile cubre Android; iOS es optativo futuro.
5. **Monetización en V4** — No invertir en infraestructura de pagos antes de tener usuarios reales.

### Umbral de rentabilidad estimado

> **Validar antes de escalar.** El objetivo para M6 no es ganancia, es retención y NPS positivo.

- Break-even objetivo: 500 usuarios activos mensuales con conversión del 5% a skins/donaciones.
- Costo operativo estimado V1-V2: **< $20 USD/mes** (Hetzner CX21 o DigitalOcean Basic).

---

## Riesgos de Producto a Vigilar

- Nicho pequeño si no se expanden sistemas de rol soportados.
- Dependencia de que el Mobile Developer soporte Expo sin fricción mayor.
- El socio UX/UI es futuro — V1 puede sufrir en diseño; priorizar funcionalidad y mejorar visual en V2.

---

## Open Strategic Questions

> Estas preguntas deben responderse antes de V2:

1. ¿Se apunta únicamente a Android o se incluye iOS desde el inicio con Expo?
2. ¿El modelo de salas es gratuito ilimitado o hay un límite de salas por usuario free?
3. ¿Se parte con idioma español first o inglés first para el mercado?
4. ¿Las skins son puramente cosméticas o hay elementos de gameplay asociados?
5. ¿Se contemplan salas públicas en un "lobby" tipo directorio, o solo salas privadas por invitación?
