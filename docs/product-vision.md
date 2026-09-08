# Visión de producto — Rolling With Fire

## Tesis

**El broadcast instantáneo de la tirada es el producto.** Cinco personas mirando el
mismo dado caer al mismo tiempo, desde cinco lugares distintos. Todo lo demás
—historial, macros, permisos, perfiles— existe para sostener ese momento.

No es una feature de la tercera versión. Sin eso, esto es un dice roller genérico
más, y ese mercado ya está saturado.

---

## Posicionamiento

**Para quién:** jugadores de rol de mesa (D&D, Pathfinder, CoC) que juegan remoto o
híbrido, en Android, empezando por el mercado hispanohablante.

**Qué resolvemos:** la tirada hoy vive en un bot de Discord que escupe texto, en una
app de dados sin contexto de sala, o en un grupo de WhatsApp. En los tres casos se
pierde lo mismo: el momento compartido.

**Cómo nos diferenciamos:**

| Factor | Apps típicas | Rolling With Fire |
|---|---|---|
| Tirada | Client-side | Server-side, resultado único canónico (D1) |
| Sincronía | El que tira ve primero | Reveal sincronizado por plazo (D2) |
| Historial de sala | No, o efímero | Persistente, paginado por cursor (D10) |
| Tiradas compuestas | Raras, o con parser de fórmulas | Pasos ordenados, editables, con nombre (D3) |
| Personajes | Variables globales del usuario | Perfiles portables: PJ, NPC, monstruo (D4) |
| Tiradas ocultas | Inexistentes | Declaradas al tirar, filtradas server-side (D5) |
| Peso | 25–40MB | ~15MB, pensada para gama baja (D13) |
| Monetización | Ads agresivos o paywall | Gratis. Donaciones (D12) |

El desglose es parte del diferencial, no un detalle: cada paso entra a la vista con
su nombre y su acumulado (`+3 Fuerza`… `+2 Rage`) y el total se arma delante del
jugador. Eso es lo que hace buena la tirada de Baldur's Gate 3, y no son los
polígonos.

---

## Modelo de sostenimiento

**La app es gratis y no tiene tier pago.** Se sostiene con donaciones (D12).

Eso no es altruismo, es la consecuencia de dos elecciones: BEAM hace que un usuario
conectado y ocioso cueste casi nada, y no hay infraestructura de pagos que mantener.
Descartados explícitamente: skins cosméticas, Google Play Billing, suscripciones.

Costo operativo estimado hasta miles de usuarios: **< $20 USD/mes** en un VPS único.

> Donaciones entran cuando haya usuarios. Nada de infraestructura de pagos antes.

---

## Cómo se llega

El orden de trabajo está en `mvp.md`, y no es por capas: es **por miedo**. Primero
un APK hablándole a Phoenix con un solo botón (M0), porque es lo que valida el
riesgo real —que la animación pelee con Flutter— en días en vez de meses.

---

## Riesgos de producto a vigilar

- El nicho es chico si no se soportan varios sistemas de rol. El modelo de pasos
  ordenados (D3) es agnóstico a propósito por eso.
- No hay diseñador. El presupuesto visual va donde se nota: la animación del dado y
  el desglose. El resto puede ser sobrio.
- Una app gratis sin conversión depende de que las donaciones existan. Si no
  aparecen, el costo es bajo igual — pero el crecimiento tiene techo.

---

## Preguntas abiertas

Las de producto están cerradas: salas públicas, límites del tier gratuito y modelo
de monetización quedaron resueltas por D6, D12 y el alcance de `mvp.md`.

Queda una sola, y es de mercado, no de producto:

1. **¿Español first o inglés first?** No bloquea el MVP — bloquea la publicación en
   Play Store.

Lo único diferido del lado técnico es el **mecanismo** de autenticación (D15), con
fecha límite clara: antes de que el servidor salga de la LAN.
