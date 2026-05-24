---
name: GitHub Actions cron alta frecuencia agota plan Free (2.000 min/mes)
type: incident
description: monitor_tab.yml con cron cada 5 min generaba ~8.900 runs/mes. El plan Free de GitHub solo incluye 2.000 min/mes → workflows bloqueados a mitad de mes.
project: tab-agente
date: 2026-05-20
commits:
  - 562a7e8
  - 541126a
  - e9ab741
---

## Qué pasó

Tres workflows con crons frecuentes agotaban los minutos del plan Free de GitHub Actions:

| Workflow             | Frecuencia anterior | Runs/mes aprox | Min/mes (estimado) |
|----------------------|---------------------|----------------|--------------------|
| monitor_tab.yml      | cada 5 min          | 8.928          | ~2.680             |
| recuperar_csat.yml   | cada 30 min         | 1.488          | ~744               |
| informe_csat.yml     | cada 30 min         | 1.488          | ~744               |

Total estimado: > 4.000 min/mes → superaba el límite de 2.000 min.

## Fix aplicado

- monitor_tab.yml: cada 5 min → cada 60 min (~744 runs/mes)
- recuperar_csat.yml y informe_csat.yml: cada 30 min → cada 2h (~360 runs/mes cada uno)

Total nuevo: ~1.464 min/mes (dentro del límite Free).

## Regla derivada

Antes de deployar un cron en GitHub Actions, calcular el consumo mensual:
```
runs/mes = (60 min/h × 24 h × 30 días) / intervalo_minutos
min/mes  = runs/mes × duración_promedio_run
```

Con el plan Free (2.000 min/mes) y 5 workflows, el budget por workflow es ~400 min/mes.

Un monitor que corra cada 5 min y tarde ~18 seg gasta ~2.700 min/mes por sí solo.

## Recomendación

- Monitores de uptime: cada 30-60 min es suficiente para detectar outages reales.
- Scripts de recovery (recuperar_csat): cada 2-4h — la data no es time-critical al minuto.
- Informes: 1-2 veces al día.
- Revisar consumo real en GitHub → Settings → Billing → Actions al mes de haber subido los crons.
