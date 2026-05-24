---
name: brevo-email-lookup
description: Use when writing Python code that tracks email engagement (apertura, clic) consultando la API de Brevo por registro individual de un log de envíos. Cubre el patrón de falsos positivos por lookup via email y la solución con message_id.
---

## Patrón

Consultar la API de estadísticas de Brevo filtrando por `email` del destinatario es ambiguo cuando ese destinatario recibió múltiples emails en la ventana de lookback. La API devuelve eventos de todos los emails enviados a ese address → el estado del más reciente se confunde con registros anteriores.

**Síntoma:** filas del log de comunicaciones muestran "Sí" de apertura para emails que nunca se abrieron, porque otro email del mismo cliente sí fue abierto en la misma ventana.

**Cuándo aparece:** sistemas CLM/CRM con journeys de múltiples intentos (CSAT intento 1, 2, 3) o cualquier campaña con más de 1 envío por cliente en el lookback period.

## Fix: usar message_id como clave primaria

Guardar el `message_id` (del header SMTP) al momento del envío en el log. Usarlo como clave de lookup en lugar del email.

### Al enviar

```python
import smtplib
from email.message import EmailMessage

with smtplib.SMTP_SSL(SMTP_HOST, SMTP_PORT) as smtp:
    smtp.login(SMTP_FROM, SMTP_PASS)
    smtp.send_message(msg)
    # El message_id está en el header después del envío
    message_id = msg.get("Message-ID", "")
    # Guardar en el log: log_row["message_id"] = message_id
```

Si usas la API transaccional de Brevo, el response incluye el `messageId`:
```python
response = api_instance.send_transac_email(body)
message_id = response.message_id  # formato: <uuid@smtp-relay.sendinblue.com>
```

### Al consultar aperturas/clics

```python
def verificar_evento_brevo(message_id: str | None, email: str, days: int, event: str) -> bool:
    """Lookup exacto por message_id; fallback a email para filas históricas."""
    if message_id:
        # Brevo espera el ID sin los <> del header SMTP
        mid_clean = message_id.strip('<>')
        # Usar endpoint de estadísticas con messageId filter
        params = {"messageId": mid_clean, "event": event, "days": days}
    else:
        params = {"email": email, "event": event, "days": days}
    
    resp = requests.get(BREVO_STATS_URL, headers=BREVO_HEADERS, params=params)
    data = resp.json()
    return bool(data.get("events"))
```

### En el log (gspread)

Agregar columna `message_id` al Log_Comunicaciones:

```python
HEADERS = ["fecha", "email", "tipo", "abierto", "clickeo", "message_id", ...]

# Al leer para actualizar:
idx_message_id = headers.index('message_id') if 'message_id' in headers else -1
raw_mid = row[idx_message_id].strip() if idx_message_id >= 0 and idx_message_id < len(row) else ''
message_id = raw_mid.strip('<>') if raw_mid else None
```

## Ventana de lookback

Para journeys con múltiples intentos, usar 14 días (no 7). Un ciclo CSAT de 3 intentos puede durar hasta 12 días; con 7 días se pierden aperturas del intento 1 cuando se procesa el intento 3.

## Deduplicación de respuestas

Si el sistema recibe respuestas (p.ej. Typeform), deduplicar por `email+order_id`, no solo por `email`. Un cliente recurrente puede tener múltiples pedidos y debe recibir CSAT por cada uno.

```python
seen = set()
for row in all_values[1:]:
    key = (row[idx_email].lower().strip(), row[idx_order_id].strip())
    if key in seen:
        continue
    seen.add(key)
    # procesar row
```

## Por qué importa

Implementado en tab-agente (commits 314768b, cf64a4b, d696de7 — mayo 2026). Los falsos positivos de engagement inflaban métricas del CLM y excluían incorrectamente clientes de journeys por supuesta reactivación.
