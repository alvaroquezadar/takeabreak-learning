---
name: GHA workflows — cron duplicado y over-cutting de frecuencia en la misma semana
type: incident
description: En la misma semana ocurrieron dos problemas opuestos de gestión de crons en GHA — un cron duplicado que causó double-run, y una reducción agresiva de frecuencia que eliminó por error un cron funcional.
project: tab-agente
date: 2026-05-31
commits:
  - 5873d35
  - b25691f
  - 33166c4
  - 47c85a5
related: pattern_2026-05-21_github-actions-cron-quota.md
---

## Qué pasó

**1. Cron duplicado → double-run (commit 5873d35):**

Un workflow tenía dos entradas `cron: '6 3 * * *'` en el mismo bloque `on.schedule`. GitHub ejecuta el workflow una vez por entrada → double-run, doble envío de email a clientes.

**2. Over-cutting → cron eliminado por error (commits 33166c4 → b25691f):**

Al reducir workflows para quedar bajo 1.500 min/mes se eliminó el cron `03:06 UTC` del `email_mensual` junto con otros crons que sí debían recortarse. El job seguía siendo necesario y se tuvo que restaurar en el commit inmediato siguiente.

**3. Tres rondas de ajuste en tres commits (33166c4, 47c85a5, b25691f):**

La reducción de frecuencia requirió tres pasadas para quedar estable: primera reducción agresiva → restaurar lo que se rompió → ajustes finos de preferencias. Señal de que no hubo inventario previo de qué crons existían y para qué servían.

## Raíz del problema

Editar múltiples secciones `on.schedule` en YAML a mano sin inventario previo es propenso a:
- Introducir duplicados por copy-paste.
- Perder el mapeo función↔cron cuando hay varios workflows.
- Cortar demasiado o muy poco en una sola pasada.

## Regla derivada

**Antes de tocar crons en GHA:**

```bash
# Inventario rápido de todos los crons del repo
grep -rn "cron:" .github/workflows/ | sort
```

Construir una tabla: `workflow | job | cron | función | min/mes estimados` antes de hacer cambios.

**Para detectar duplicados antes de commitear:**

```bash
grep -n "cron:" .github/workflows/<workflow>.yml
```

Si aparece el mismo schedule dos veces en el mismo archivo → duplicado.

**Al reducir frecuencia por quota:**
- Hacer una sola pasada con todos los cambios documentados.
- Verificar que cada cron eliminado no tiene un job dependiente sin otra fuente de trigger.
- Budget orientativo con plan Free (2.000 min/mes, ~5 min/run): máx ~80 runs/día totales entre todos los workflows.
