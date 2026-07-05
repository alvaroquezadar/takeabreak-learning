---
name: whatsapp-channel-monitoring
type: feedback
description: El canal de WhatsApp de pedidos estuvo caído (app Meta desactivada, Jun 19) y se detectó por reclamo de clienta, no por alerta. No existe hoy ningún monitor que verifique el estado del token/canal. Blindspot idéntico al de uptime del sitio web antes del GCM check.
---

## Patrón

El token de WhatsApp (`WA_TOKEN_PROD` en `.env`) puede morir sin que ningún script lo detecte:
- La app de Meta se desactiva (por registro de desarrollador incompleto, como ocurrió el 19-jun).
- El token del usuario del sistema vence o pierde permisos.
- Meta invalida el token por inactividad o cambio de política.

En todos esos casos el snippet PHP `PacksMensuales` (WPCode ID 81637) sigue corriendo, recibe HTTP 200 del hook de WooCommerce, pero Graph API devuelve un error de OAuth → el WhatsApp de confirmación de pedido y el del cupón de plan **no llegan al cliente** en silencio.

La caída del 19-jun estuvo activa un tiempo no determinado antes de que Patricia Carrasco reclamara.

## Check mínimo

```python
import os, requests

WA_TOKEN = os.environ["WA_TOKEN_PROD"]
PHONE_ID = "443539018839522"  # número de producción

resp = requests.get(
    f"https://graph.facebook.com/v21.0/{PHONE_ID}",
    params={"access_token": WA_TOKEN},
)
data = resp.json()

if "error" in data:
    raise RuntimeError(f"WhatsApp token/canal ERROR: {data['error']}")

# Verificar que el canal esté conectado
status = data.get("verified_name", {})  # respuesta varía por endpoint
# Alternativa: debug_token endpoint
resp2 = requests.get(
    "https://graph.facebook.com/debug_token",
    params={"input_token": WA_TOKEN, "access_token": WA_TOKEN},
)
token_data = resp2.json().get("data", {})
if not token_data.get("is_valid"):
    raise RuntimeError(f"WA_TOKEN inválido: {token_data}")
```

## Propuesta de fix (audit Jul 1, C.1)

Nuevo script `agente_monitor_whatsapp_tab.py` con cron diario — mismo patrón que `agente_monitor_tab.py`. Usa `@con_alerta` para enviar email a alvaro@ si el token está muerto o el canal no está CONNECTED/GREEN.

## Regla general

Todo canal de comunicación con clientes necesita un healthcheck periódico independiente del flujo de envío. El flujo de envío no detecta el error si el hook de WooCommerce llama al PHP y el PHP recibe HTTP 200 del servidor antes de llegar a la API de Meta.
