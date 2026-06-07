---
name: CLM Journey — feature flag como gate de copy
type: feedback
description: Patrón de la semana 2026-06-06 — journey Plan Mensual D-5 implementado completo pero pausado con flag booleano hasta que el copy esté aprobado. Mismo patrón aplicado a lifecycle journeys (recompra / pre_at_risk / at_risk).
---

**Qué pasó:**

En la misma sesión del 2026-06-06 se hicieron dos commits consecutivos sobre `agente_clm_tab.py`:

1. `feat(clm): journey Plan Mensual D-5 renovación` — 295 líneas de código nuevo listo para producción.
2. `chore(clm): deshabilitar journey PM hasta aprobación de copy` — `PM_JOURNEY_ENABLED = False`.
3. `chore(clm): pausar journeys lifecycle hasta rediseño de copy` — `ORGANICO_DRY_RUN = True`.

El código funciona, pero el copy no fue aprobado antes de llegar a producción. El flag evita que los emails salgan mientras el journey permanece deployado.

**Por qué es buen patrón:**

- El código queda versionado y revisable sin crear una rama larga.
- El flag tiene nombre descriptivo que explica qué falta (`PM_JOURNEY_ENABLED`, `ORGANICO_DRY_RUN`).
- El commit de pausa documenta exactamente qué falta y cómo reactivar.
- CSAT sigue activo porque no tiene copy de venta — siempre discriminar qué puede correr y qué no.

**Cómo reactivar:**

```python
PM_JOURNEY_ENABLED = True   # journey PM — copy aprobado
ORGANICO_DRY_RUN = False    # lifecycle journeys — copy aprobado
```

**Regla derivada:**

Antes de hacer `feat(clm)` en producción, verificar si el copy fue aprobado. Si no: deployar con flag `_ENABLED = False` o `_DRY_RUN = True` desde el commit inicial — no requiere un segundo commit de pausa.
