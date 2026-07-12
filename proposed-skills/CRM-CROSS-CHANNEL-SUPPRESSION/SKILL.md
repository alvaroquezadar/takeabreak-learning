---
name: CRM-CROSS-CHANNEL-SUPPRESSION
description: Use when building or launching a new email campaign carril (journey, one-shot, winback) alongside existing ones — to design global suppression between carrils so a contact doesn't receive emails from multiple campaigns within the same N-day window.
---

# CRM — Supresión global entre carriles de email

## Patrón

Cuando corren múltiples carriles de email en paralelo (primer_ciclo, recompra, winback, one-shot),
sin supresión global un mismo contacto puede recibir emails de dos carriles distintos el mismo día
o dentro de una ventana corta. Esto genera ruido para el cliente y contamina la atribución.

## Implementación (Python + Brevo + log en Sheets)

```python
SUPPRESSION_DAYS = 7  # ventana entre carriles

def ya_recibio_reciente(email: str, log_rows: list[list], dias: int = SUPPRESSION_DAYS) -> bool:
    """True si el contacto recibió cualquier email de carril en los últimos N días."""
    hoy = date.today()
    for row in log_rows:
        if row[COL_EMAIL].lower() == email.lower():
            try:
                enviado = date.fromisoformat(row[COL_FECHA_ENVIO])
                if (hoy - enviado).days < dias:
                    return True
            except (ValueError, IndexError):
                continue
    return False
```

El log de carriles es la SSOT de supresión: cada carril escribe en el mismo Sheet
(o tabla) al enviar → todos los demás lo leen antes de enviar.

## Excepciones válidas

- **Re-serve / reenvío no-abridores**: re-sirve el mismo email a quien no lo abrió.
  No cuenta como un envío nuevo para efectos de la supresión.
- **Transaccionales** (confirmación de pedido, cupón de canje): siempre pasan, sin supresión.

Documentar las excepciones explícitamente en el código con un comentario que explique el motivo.

## Regla

Al agregar un nuevo carril de email:
1. ¿Qué otros carriles corren en paralelo? Listarlos.
2. Definir la ventana de supresión (7d es el default TAB).
3. El nuevo carril lee el log compartido antes de cada envío.
4. El nuevo carril escribe en el log compartido después de cada envío.
5. Los reenvíos (no-abridores del mismo mensaje) se marcan con un flag distinto en el log
   para no contaminar la ventana de supresión del carril siguiente.
