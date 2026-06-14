---
name: wc-rest-api-auth
description: Use when writing or reviewing Python scripts that call the WooCommerce REST API — para verificar que usan consumer key/secret y no credenciales de login WordPress, y para auditar el blast radius cuando se cambia la configuración de permisos de la REST API en el sitio.
---

## Patrón

La WooCommerce REST API requiere autenticación con **consumer key / consumer secret** (`ck_...` / `cs_...`), no con el email y contraseña de la cuenta WordPress. Usar auth de WP en vez de consumer key "funciona" solo si el permission callback de la API está abierto (`return true`). Al cerrar ese hueco, todos los scripts con auth incorrecta fallan con HTTP 401, a menudo en silencio.

## Forma correcta

```python
import os, requests

WC_URL   = os.environ["WC_URL"]    # https://sitio.cl/wp-json/wc/v3
WC_AUTH  = (os.environ["WC_KEY"], os.environ["WC_SECRET"])  # ck_... / cs_...

resp = requests.get(f"{WC_URL}/orders", auth=WC_AUTH, params={"per_page": 100})
resp.raise_for_status()
```

Variables de entorno correctas: `WC_KEY` (empieza en `ck_`) y `WC_SECRET` (empieza en `cs_`).

## Auditoría rápida

```bash
# Detectar scripts que usan login WP en vez de consumer key
grep -rn "WC_USER\|WC_PASS" *.py

# Si aparece como auth= en requests → reemplazar por WC_KEY/WC_SECRET
```

## Señal de alerta

Si un cambio de seguridad en WP (plugin, filtro de permisos, actualización WC) coincide con que varios agentes empiezan a fallar o reportar datos vacíos: buscar primero si algo estaba abierto y ahora requiere credenciales reales de API.

## Blast radius típico en TAB (incidente 2026-06-11)

6 scripts afectados simultáneamente al cerrar el permission callback:
- Cierre diario → $0 en silencio
- Informe mensual → $0 en silencio  
- Actualizador de pedidos → no bajaba data
- Informe ventas web → 401 fuerte (tenía validación)
- CLM → 401 en wc_session()
- Enrich CSAT → 401

Fix: `auth=(WC_USER,WC_PASS)` → `auth=(WC_KEY,WC_SECRET)` en los 6 + actualizar GitHub Secrets.

## Por qué importa

Un permission callback abierto en WP es un agujero de seguridad real (cualquier usuario autenticado puede leer/escribir órdenes). Al corregirlo se rompen todas las integraciones que no usan la auth correcta. La corrección es sencilla una vez identificada, pero detectarla tarde cuesta datos incorrectos llegando a ejecutivos.
