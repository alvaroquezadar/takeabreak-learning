---
name: gha-monitor-vs-cloud-native
type: feedback
description: El workflow de monitor de uptime en GHA se desactivó porque Google Cloud Monitoring (externo, multi-región, gratis, cada 5min) lo reemplaza mejor. Un monitor GHA: solo fines de semana, cada 2h, single-region, consume minutos pagos.
---

## Decisión

Reemplazar el monitor custom `agente_monitor_tab.py` corriendo en GHA por **Google Cloud Monitoring uptime checks**.

| Criterio          | GHA monitor                     | GCM uptime checks               |
|------------------|---------------------------------|---------------------------------|
| Frecuencia        | Cada 2h (solo fines de semana)  | Cada 5 min, todos los días      |
| Cobertura         | Single-region (us-east-1)       | Multi-región, externo           |
| Costo             | Minutos de GHA (pagos)          | Gratis (nivel free tier)        |
| Falsos positivos  | Sí (single-region, flaps)       | Raro (multi-región confirma)    |
| Alertas           | Email vía SMTP propio            | Email/SMS/webhook nativos       |

## Por qué importa

Un monitor de uptime que corre **desde adentro del mismo ecosistema** (GHA → GitHub → mismo CDN/red) puede fallar en detectar caídas que sí ven los usuarios externos, o reportar falsos DOWN por fallas de red propias del runner. Un check externo y multi-región confirma con mayor fidelidad lo que experimenta el cliente.

## Estado

El workflow `monitor_tab.yml` quedó con `workflow_dispatch` solo (desactivado el cron). Se mantiene como herramienta manual si se necesita un check puntual desde el entorno Python (con más contexto que el HTTP check básico de GCM).

## Regla para futuros proyectos

Antes de crear un monitor de uptime en GHA: verificar si el proveedor de infraestructura ya ofrece uptime checks externos. Si sí, usarlos. El monitor GHA agrega valor solo cuando necesita lógica de negocio específica (autenticación con variables de entorno del repo, verificación de payload JSON, etc.) que GCM no puede expresar.
