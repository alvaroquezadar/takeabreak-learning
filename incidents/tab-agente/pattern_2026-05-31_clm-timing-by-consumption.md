---
name: CLM timing — calibrar días de disparo contra ventana real de consumo del pack
type: project
description: Los emails de recompra y win-back del CLM deben dispararse dentro de la ventana de decisión activa del cliente, que depende del ritmo real de consumo del pack, no de una estimación genérica.
project: tab-agente
date: 2026-05-31
commits:
  - a310956
  - 0800d79
---

## Qué pasó

Dos emails del CLM llegaban fuera de la ventana de decisión del cliente:

**Pack14 — email de recompra (antes día 21, corregido a día 12):**
- Análisis de datos: el pico de recompra Pack14 ocurre en días 0-11 post-compra.
- El email llegaba al día 21 → el cliente ya había decidido (recomprar o no) al menos 10 días antes.
- Fix: mover a día 12 ± 2 días de tolerancia → ventana efectiva días 10-14.
- Baseline pre-cambio: retención 30d Pack14 orgánico = 36.8%.

**Pack5 y Pack7 — email at_risk / win-back (antes día 45, corregidos a días 28/32):**
- Pack5 y Pack7 se consumen en 5-14 días. El hábito se rompe alrededor del día 20.
- El email llegaba al día 45 → el hábito llevaba ~35 días roto, ventana de recuperación cerrada.
- Fix: Pack5 → día 28, Pack7 → día 32.

## Raíz del problema

Los días de disparo fueron definidos con estimaciones conservadoras al crear el CLM, sin datos reales de ritmo de consumo por pack. Una vez con datos de cohortes, los días óptimos resultan significativamente menores a los estimados inicialmente.

## Regla derivada

Para cada email del CLM, documentar tres valores antes de fijar el día de disparo:

| Campo | Descripción |
|-------|-------------|
| `días_consumo_promedio` | Cuántos días tarda el pack en terminarse (basado en datos reales) |
| `ventana_decisión` | Rango de días en que el cliente está activamente decidiendo qué sigue |
| `punto_no_retorno` | Día estimado a partir del cual el hábito está definitivamente roto |

El email debe dispararse dentro de `ventana_decisión` y antes de `punto_no_retorno`. Revisar estos valores cada vez que haya datos de retención por cohorte actualizados.
