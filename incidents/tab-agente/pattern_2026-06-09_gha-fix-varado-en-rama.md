---
name: Fix varado en rama sin mergear — main sigue roto aunque el arreglo exista
type: incident
description: AG-26 Meta Ads fallaba a diario en CI. El fix ya estaba escrito en la rama feature/calidad-trafico pero no había sido pusheado ni mergeado. Main siguió enviando emails de falla durante días.
project: tab-agente
date: 2026-06-09
commits:
  - b568f71
  - fix original (commit 3051937 en feature/calidad-trafico)
---

## Qué pasó

El workflow `ag_26_meta_ads.yml` fallaba todos los días con:

```
Invalid pattern '../tab-intelligence/data/db/tab_retail.duckdb'.
Relative pathing '.' and '..' is not allowed.
```

El arreglo ya estaba escrito — commit `3051937` en `feature/calidad-trafico` — pero esa rama nunca fue pusheada ni mergeada. `origin/main` seguía con el workflow roto y lo ejecutaba cada día según su `schedule`.

## Causa secundaria: workflow local-first con schedule

El workflow AG-26 estaba diseñado para correr en local, donde `tab-agente/` y `tab-intelligence/` son carpetas hermanas. Tenía:
1. Un `schedule: cron '30 10 * * *'` — lo ejecutaba diariamente en GHA
2. Un step `upload-artifact` con `path: ../tab-intelligence/...` — `..` no está permitido en `upload-artifact` y el path no existe en el runner hosted

Un workflow pensado para uso local no debe tener `schedule` en GHA.

## Fix

Cherry-pick del commit existente a `main` vía git worktree aislado (sin contaminar `feature/calidad-trafico`):
- Sacar el `schedule` → solo `workflow_dispatch` (manual)
- Sacar el step de upload-artifact con path relativo

## Reglas derivadas

**Antes de diagnosticar un fallo recurrente de CI, verificar si ya existe el fix:**

```bash
git log --all --oneline -- .github/workflows/<workflow>.yml
```

Si el fix ya está en otra rama, llevarlo a main con cherry-pick en worktree aislado.

**Workflows con paths a repos hermanos (`../otro-repo/`) no deben tener `schedule`** — ese path solo existe en local. Dejarlos como `workflow_dispatch` o moverlos a `tab-intelligence`.

**Un fix no sirve si está en una rama sin pushear.** Al terminar de escribir un fix para un fallo activo de producción, el paso siguiente es: pushear la rama y mergear (o cherry-pick a main si la rama tiene trabajo a medio terminar).
