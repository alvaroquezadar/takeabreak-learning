---
name: email-unsubscribe-link
description: Use when writing or reviewing HTML email templates for marketing journeys (re-engagement, lifecycle, promotions) — para verificar que el mecanismo de opt-out es funcional, no un `href="#"` falso, y para evitar degradación de entregabilidad del dominio.
---

## Patrón

Los emails de marketing deben incluir un link de "Dejar de recibir estos correos" que realmente funcione. Un `href="#"` no lleva a ningún lado — el cliente hace clic, no pasa nada, y en vez de darse de baja reporta spam. Gmail y Outlook penalizan dominios con unsubscribes rotos aumentando la tasa de clasificación como spam, lo que degrada la entregabilidad de **todos** los correos del dominio, incluidos los transaccionales.

### Caso real (tab-agente, detectado audit Jul 2026)

`agente_clm_tab.py`, función `_org_shell_v2` (línea ~1365) y 7 plantillas legacy incluyen:
```html
<a href="#">Dejar de recibir estos correos</a>
```

Los emails del journey orgánico (Recompra / Pre-At Risk / At Risk) están en vivo enviando a cientos de clientes. Ningún click produce nada. El Sheet de clientes tiene columna `excluir_journeys` pero no hay ningún punto de entrada desde el email.

## Opciones de implementación (por esfuerzo)

### Opción A — mailto (mínimo viable, bajo esfuerzo)

```html
<a href="mailto:alvaro@takeabreak.cl?subject=Dar de baja&body=Por favor, no me envíes más correos de recetas.">
  Dejar de recibir estos correos
</a>
```

Simple, sin infraestructura. Requiere procesamiento manual. Útil mientras el volumen es bajo.

### Opción B — Google Form (sin infra propia)

Un Form que recibe el email del cliente y escribe en un Sheet. Un script de Apps Script lee ese Sheet y actualiza `excluir_journeys=true` en el Sheet de clientes. No requiere servidor propio.

### Opción C — Endpoint propio (robusto)

Apps Script o Cloudflare Worker que recibe `?email=X&token=HMAC` y marca la exclusión directamente en Sheets. El precedente en el ecosistema: `feedback_worker` de berlín. Token = HMAC-SHA256 del email con una clave en `.env` — evita que un tercero dé de baja a otro usuario.

```python
import hmac, hashlib, os

def generar_token_unsub(email: str) -> str:
    key = os.environ["UNSUB_SECRET"].encode()
    return hmac.new(key, email.encode(), hashlib.sha256).hexdigest()[:16]
```

## Verificación rápida

```bash
# Buscar href="#" en plantillas de email del repo
grep -rn 'href="#"' agente_clm_tab.py | grep -i "recibir\|baja\|unsub"
```

## Regla

En cualquier email de cadencia de marketing (journey de lifecycle, reactivación, promociones), el link de opt-out debe ser funcional antes de activar el envío a producción. Verificarlo junto con el DRY_RUN.
