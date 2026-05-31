---
name: Brevo API clicks tracking — messageId debe incluir brackets <>
type: incident
description: La API de estadísticas de Brevo requiere el messageId CON los brackets <> para devolver resultados. strip('<>') antes del lookup causó 0% clickeo en todo el historial del CLM.
project: tab-agente
date: 2026-05-31
commits:
  - a45f07d
related: pattern_2026-05-21_brevo-lookup-por-message-id.md
---

## Qué pasó

`actualizar_aperturas_log()` aplicaba `strip('<>')` al `message_id` antes de enviarlo como parámetro `messageId` a la API de estadísticas de Brevo. Resultado: 0 eventos devueltos para todas las búsquedas de clics, haciendo que la columna clickeo% del Log_Comunicaciones estuviera en 0% durante semanas.

La corrección del bug anterior (ver pattern relacionado) introdujo el uso de `message_id` como clave primaria, pero asumió por error que la API recibía el ID sin brackets — siguiendo la convención SMTP donde el header `Message-ID` se normaliza sin `<>` al almacenar.

**Validado:** `messageId=<uuid@host>` retorna eventos correctamente; `messageId=uuid@host` (sin brackets) retorna 0 eventos.

## Corrección aplicada

```python
# ANTES (incorrecto — codificado en el skill BREVO-EMAIL-LOOKUP original):
mid_clean = message_id.strip('<>')
params = {"messageId": mid_clean, "event": event, "days": days}

# DESPUÉS (correcto):
params = {"messageId": message_id, "event": event, "days": days}
```

No normalizar el messageId antes de pasarlo a la API. Enviarlo tal como viene del header SMTP.

## Regla derivada

La API de Brevo (endpoint `/smtp/statistics/events`) espera el `messageId` tal como aparece en el header SMTP, incluyendo los brackets `<>`. Normalizar antes de la llamada rompe el lookup silenciosamente — la API responde 200 con lista vacía, no con error.

El skill propuesto BREVO-EMAIL-LOOKUP fue corregido en la misma fecha para reflejar esto.
