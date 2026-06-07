---
name: gha-concurrency-multi-cron
description: Use when a GitHub Actions workflow has two or more cron schedules (on.schedule) that could run close together — specifically when setting or debugging the concurrency.group key to avoid one cron blocking or canceling another.
---

## Patrón

Un workflow con múltiples entradas en `on.schedule` y un grupo de concurrencia basado solo en el nombre del workflow o en el mode del dispatch puede bloquear sus propios crons.

**Ejemplo del bug (tab-agente, 2026-06-06):**

`agente_clm_tab.yml` tenía dos crons: `reenvios_pm` (22:50 UTC, ~4h de duración) y `clm_full` (09:00 UTC). Ambos caían en el mismo grupo porque la clave era:

```yaml
concurrency:
  group: ${{ github.workflow }}-${{ github.event.inputs.mode || 'cron' }}
```

Cuando no hay `inputs.mode` (trigger por cron), ambos producen el mismo string → el segundo cron queda en cola hasta que el primero termina, o si `cancel-in-progress: true`, se cancela.

## Fix

Incluir `github.event.schedule` en la clave del grupo:

```yaml
concurrency:
  group: ${{ github.workflow }}-${{ github.event.inputs.mode || github.event.schedule || 'default' }}
  cancel-in-progress: false
```

Cada cron genera un valor distinto para `github.event.schedule` (ej: `'0 9 * * *'` vs `'50 22 * * *'`), creando grupos separados. Los manual dispatches con `inputs.mode` siguen usando ese valor como discriminador.

## Por qué `cancel-in-progress: false`

Con `false`, si dos crons coinciden en el mismo grupo (improbable pero posible), el segundo espera al primero en lugar de cancelarse. Para workflows de envío de emails a clientes, cancelar-en-progreso puede dejar journeys a medias.

## Cuándo aplica

- Workflow con 2+ entradas en `on.schedule`.
- Workflow con `on.schedule` + `on.workflow_dispatch` que comparten el mismo grupo de concurrencia.
- Bug silencioso: el cron más largo bloquea al más corto sin error visible en GHA. El job aparece como "queued" indefinidamente.

## Diagnóstico

```bash
# Ver qué grupo genera cada trigger
# En el log de GHA Actions → el job muestra "Waiting for a pending job to finish"
# si está en el mismo grupo
grep -n "concurrency:" .github/workflows/<workflow>.yml
```

Si ves "Waiting for a pending job to finish" en un job de cron que debería correr solo → revisar el grupo de concurrencia.

## Implementado en

`tab-agente/.github/workflows/agente_clm_tab.yml`, fix commit `2a123f9` (2026-06-06).
