---
name: Brevo tracking — lookup por email genera falsos positivos cruzados
type: incident
description: Consultar eventos Brevo filtrando por email del cliente confunde registros cuando el mismo cliente recibió varios emails en la ventana. Fix — usar message_id como clave única.
project: tab-agente
date: 2026-05-21
commits:
  - 314768b
  - cf64a4b
  - d696de7
---

## Qué pasó

`actualizar_aperturas_log()` en `agente_clm_tab.py` consultaba la API de Brevo filtrando por `email` del cliente para detectar aperturas y clics. Si el cliente recibió 2 emails en la misma semana (p.ej. CSAT intento 1 y intento 2), ambas filas del log mostraban el mismo resultado: el estado del email más reciente se propagaba hacia atrás.

**Resultado:** falsos positivos de apertura/clic en filas anteriores del log; métricas de engagement del CLM infladas.

Además, en `cruzar_nps_csat()`:
- El índice de columna del score estaba hardcodeado en `col[7]` pero el formato real de `recuperar_csat_brevo` pone el score en `col[3]`.
- La deduplicación usaba solo `email`, no `email+order_id` — clientes recurrentes solo recibían CSAT por su primera compra.

## Raíz del problema

Usar `email` como proxy del evento de email es ambiguo cuando hay múltiples emails enviados al mismo destinatario en la ventana de lookback.

## Fix aplicado

1. Guardar `message_id` en el Log_Comunicaciones al momento del envío.
2. En el lookup de apertura/clic: usar `message_id` (stripping `<>` del header SMTP) como clave primaria. Fallback a email solo si `message_id` está vacío.
3. Deduplicación CSAT cambiada a `email+order_id`. Fallback a solo email para filas históricas sin `order_id`.
4. Ventana de lookback ampliada 7 → 14 días para cubrir el ciclo completo CSAT (hasta intento 3).

```python
raw_mid = row[idx_message_id].strip() if idx_message_id >= 0 and idx_message_id < len(row) else ''
message_id = raw_mid.strip('<>') if raw_mid else None

if message_id:
    ev_open  = verificar_aperturas_brevo(message_id=message_id, days=dias, event="opened")
    ev_click = verificar_aperturas_brevo(message_id=message_id, days=dias, event="clicks")
else:
    ev_open  = verificar_aperturas_brevo(email=email_row, days=dias, event="opened")
    ev_click = verificar_aperturas_brevo(email=email_row, days=dias, event="clicks")
```

## Regla derivada

Cada sistema de log de emails que necesite rastrear apertura/clic por registro individual debe guardar el `message_id` al momento del envío y usarlo como clave de lookup, no el email del destinatario.

Ver skill propuesto: `BREVO-EMAIL-LOOKUP`
