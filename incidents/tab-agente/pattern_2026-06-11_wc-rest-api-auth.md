---
name: WC REST API — autenticar con consumer key/secret, nunca con login WordPress
type: project
description: 6 scripts usaban auth=(WC_USER, WC_PASS) —credencial de login WP— para llamar la WC REST API. Funcionaban solo porque el permission callback estaba abierto (return true). Al cerrar ese hueco, los 6 fallaron en silencio con HTTP 401.
project: tab-agente
date: 2026-06-11
commits:
  - 54e1dc5
---

## Qué pasó

El 10/06 se corrigió una vulnerabilidad en el sitio WP (`my_woocommerce_rest_check_permissions()`: `return true` → `return $permission`). Al día siguiente, el cierre diario reportó $0 / 0 pedidos. Causa: 6 scripts autenticaban la WC REST API con `auth=(WC_USER, WC_PASS)` (= email + contraseña de la cuenta WP), no con consumer key/secret. Esa combinación no es una credencial válida de API — solo pasaba porque la API estaba abierta para todos.

## Blast radius

| Script | Modo de falla |
|--------|--------------|
| `agente_email_tab.py` | Reportó $0 en silencio |
| `agente_email_mensual_tab.py` | Ídem |
| `agente_actualizar_tab.py` | No bajaba pedidos |
| `agente_informe_ventas_web.py` | Fallaba fuerte (tenía validación) |
| `agente_clm_tab.py` | 401 en `wc_session()` |
| `recuperar_csat_brevo.py` | 401 en enrich de pedidos |

## Regla

```python
# MAL — login de WordPress, no credencial de API
auth = (os.environ["WC_USER"], os.environ["WC_PASS"])

# BIEN — consumer key / secret (ck_... / cs_...)
auth = (os.environ["WC_KEY"], os.environ["WC_SECRET"])
```

Las vars de entorno que corresponden: `WC_KEY` (empieza en `ck_`) y `WC_SECRET` (empieza en `cs_`).

## Prevención

Al agregar o revisar cualquier script que llame a la WC REST API:

```bash
grep -rn "WC_USER\|WC_PASS" *.py
```

Si aparece en una llamada a `requests.get/post` como `auth=`, reemplazar por `WC_KEY`/`WC_SECRET`.

## Señal de alerta

Si un cambio de seguridad en el sitio WP (permisos, plugin, filtro) coincide con que varios agentes empiezan a fallar o reportar datos vacíos — buscar primero si algo que antes estaba abierto ahora requiere credenciales reales.
