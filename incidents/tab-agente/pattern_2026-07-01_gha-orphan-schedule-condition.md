---
name: gha-orphan-schedule-condition
type: feedback
description: El job email_avance_18 tiene condición `github.event.schedule == '0 21 * * *'` pero ese cron nunca se agregó a `on.schedule`. Lleva 3 auditorías consecutivas sin fix. El job solo dispara con workflow_dispatch manual.
---

## Anti-patrón

```yaml
# .github/workflows/actualizar_tab.yml

on:
  schedule:
    - cron: '0 10 * * *'
    - cron: '0 16 * * *'
    - cron: '5 3 * * *'
    # ⚠️ falta: '0 21 * * *'
  workflow_dispatch:

jobs:
  email_avance_18:
    if: github.event.schedule == '0 21 * * *'  # nunca se cumple
```

El job `email_avance_18` solo dispara cuando GHA inyecta `github.event.schedule = '0 21 * * *'`. Eso requiere que esa expresión esté en `on.schedule`. Si no está → el condicional nunca es `true` → el job nunca corre automáticamente.

GHA no lanza ningún error. El workflow reporta "skipped" o simplemente no corre ese job sin explicación.

## Check preventivo al tocar workflows con condicionales de schedule

```bash
# Extraer todos los valores de on.schedule
grep -A1 "cron:" .github/workflows/actualizar_tab.yml | grep "'"

# Extraer todos los condicionales que referencian schedule
grep "event.schedule ==" .github/workflows/actualizar_tab.yml
```

Verificar que cada valor en los condicionales aparece también en el bloque `on.schedule`.

## Fix

Agregar `- cron: '0 21 * * *'` al bloque `on.schedule` de `actualizar_tab.yml`. (Chile continental = UTC-3 año redondo desde 2019 — confirmar offset antes del merge.)

## Estado al 2026-07-05

Hallazgo de 3 auditorías consecutivas (May / Jun / Jul 2026). El fix es 1 línea, no se ha aplicado. El avance de 18:00 (el más útil operativamente — tendencia del día antes del cierre) lleva 2 meses solo disponible via `workflow_dispatch` manual.
