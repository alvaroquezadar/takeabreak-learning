---
name: gha-cron-audit
description: Use when adding, removing, or modifying cron schedules in GitHub Actions workflows — especialmente antes de reducir frecuencia para bajar consumo de minutos del plan Free (2.000 min/mes), o cuando el repo tiene varios workflows con crons.
---

## Patrón

Editar crons de GHA sin inventario previo genera dos tipos de errores opuestos:
1. **Duplicado**: dos entradas con el mismo schedule en `on.schedule` → double-run, doble envío de emails a clientes.
2. **Over-cutting**: al recortar frecuencia para bajar minutos, se elimina por error un cron funcional → job deja de ejecutarse sin error visible.

Ambos son silenciosos: GHA no avisa del duplicado, y el job eliminado simplemente deja de correr.

## Inventario antes de tocar

```bash
# Ver todos los crons del repo en una pasada
grep -rn "cron:" .github/workflows/ | sort

# Detectar duplicados dentro de un archivo específico
grep -n "cron:" .github/workflows/<workflow>.yml
```

Construir la tabla antes de hacer cambios:

| Workflow | Cron | Función | Min/mes estimados |
|----------|------|---------|------------------|
| monitor_tab.yml | `*/5 * * * *` | Uptime check | ~2.700 |
| recuperar_csat.yml | `0 */2 * * *` | Sweep CSAT | ~360 |

## Cálculo de consumo

```python
runs_mes = (60 * 24 * 30) / intervalo_minutos
min_mes  = runs_mes * duracion_promedio_run_min
```

Budget orientativo con plan Free (2.000 min/mes) y duración promedio 3 min/run:
- Máx ~22 runs/día totales entre todos los workflows.
- Un monitor cada 5 min (288 runs/día × 3 min) gasta el plan solo.

## Al reducir frecuencia

Hacer una sola pasada con todos los cambios. Antes de eliminar un cron:
1. Verificar qué función cumple en el código (`grep` del nombre del job en el script).
2. Confirmar que no hay otro job que dependa de su output.
3. Si el job debe seguir corriendo pero menos seguido, cambiar el interval — no eliminar el cron entero.

## Al agregar un nuevo cron

```bash
# Buscar duplicados antes del commit
grep -n "cron:" .github/workflows/<nuevo_workflow>.yml
```

Si el schedule ya existe en el mismo archivo → duplicado. GitHub ejecuta el workflow una vez por entrada de `on.schedule`.

## Por qué importa

Implementado en tab-agente semanas 2026-05-20 y 2026-05-31: el plan Free fue agotado por un monitor con cron de 5 min, y en la corrección se introdujo un duplicado y un over-cut en la misma semana. Ver patterns `pattern_2026-05-21_github-actions-cron-quota.md` y `pattern_2026-05-31_gha-cron-duplicate.md`.
