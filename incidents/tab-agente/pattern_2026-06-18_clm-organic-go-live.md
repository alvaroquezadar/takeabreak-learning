---
name: clm-organic-go-live
type: feedback
description: Go-live del journey orgánico v2 (Jun 18). Lecciones de timing de envío, diseño de asunto por etapa, y recomendados diferenciados. También: el bloque TEST_MODE quedó apuntando a funciones v1 en vez de v2 — precedente de bug de preview desfasado.
---

## Patrón

El journey orgánico (Recompra / Pre-At Risk / At Risk) se envió en vivo por primera vez el 18-jun a ~20:00. Tres aprendizajes operacionales:

### 1. Hora de envío

~20:00 no es óptimo para emails de re-engagement. Para reactivar clientes que perdieron el hábito rinde mejor mañana/mediodía (cron 6am → entregado ~9am Chile). Regla: los envíos ad-hoc de journeys deben respetar la hora que el cron hubiera disparado, no la hora a la que el operador ejecuta la sesión.

### 2. Asunto como pregunta cuando se infiere el estado del cliente

Para etapas donde no se sabe con certeza el estado del cliente (Recompra, Pre-At Risk), el asunto fue reformulado como pregunta ("¿ya se te acaba el pack?", "¿todo bien por tu cocina?") — no como afirmación. At Risk sí usó asunto directo porque es una oferta concreta, no un diagnóstico.

Regla: aserto en el asunto solo cuando el sistema tiene evidencia directa del estado. Si se infiere → pregunta.

### 3. Recomendados diferenciados por etapa

No repetir los mismos productos en cada etapa del journey. El cliente en Pre-At Risk quizás no compró más porque algo no le gustó — re-empujar sus mismos platos puede irritar.

| Etapa       | Recomendados               | Lógica                              |
|-------------|---------------------------|-------------------------------------|
| Recompra    | Sus platos comprados       | Confirmar lo que ya le gustó        |
| Pre-At Risk | Más vendidos (favoritos)   | Introducir algo distinto             |
| At Risk     | Sin recomendados           | Solo oferta + cupón, sin distracciones |

### 4. TEST_MODE debe apuntar a las funciones activas (bug detectado en audit Jul 1)

Al lanzar v2 se agregaron `_html_recompra_v2`, `_html_pre_at_risk_v2`, `_html_at_risk_v2`, y el flujo real (`ejecutar_journey_organico`) usa exclusivamente las v2. Pero el bloque `TEST_MODE=True` seguía llamando a las v1. Si alguien corre el preview para revisar At Risk antes de reactivarlo, ve el diseño que ya no se envía a nadie.

Regla: al agregar funciones `_vN`, actualizar TAMBIÉN los bloques de preview/test. Borrar las versiones anteriores si ya no tienen llamadores.

## Contexto

- Envío manual 18-jun: 147 emails reales (132 Recompra + 15 Pre-At Risk). Sin errores.
- At Risk en pausa: `ORGANICO_ETAPAS_LIVE = {"recompra","pre_at_risk"}`.
- A/B no se montó — no hay línea base histórica (primer go-live del journey).
