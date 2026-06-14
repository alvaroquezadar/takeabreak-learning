---
name: Guard anti-silencio — 401/403 debe levantar error, no devolver vacío
type: feedback
description: Scripts que llaman APIs externas pueden recibir HTTP 401/403 y devolver lista vacía en vez de fallar. El workflow de CI reporta "success" y el script emite un informe con datos en cero. El error es invisible hasta que alguien nota que los números no tienen sentido.
project: tab-agente
date: 2026-06-11
commits:
  - 54e1dc5
---

## Qué pasó

El cierre diario del 10/06 reportó $0 / 0 pedidos a ejecutivos. GHA marcó el run como "success". La causa fue que `get_ventas()` llamaba a WC REST API, recibía HTTP 401, y devolvía `[]` sin levantar ningún error. El script tomó eso como "hoy no hubo pedidos".

El día NO fue cero: con auth correcta, 35 pedidos / $1.095.052 s/IVA.

## Regla

Toda función que lea de una API externa y use su resultado como input de un informe/email debe levantar excepción ante 4xx:

```python
def get_ventas(fecha: str) -> list[dict]:
    resp = requests.get(f"{WC_URL}/orders", auth=WC_AUTH, params={"date": fecha})
    if resp.status_code in (401, 403):
        raise RuntimeError(
            f"WC REST API auth fallida ({resp.status_code}) — verificar WC_KEY/WC_SECRET"
        )
    resp.raise_for_status()
    return resp.json()
```

Combinado con el decorator `@con_alerta`, el 401 dispara un email de alerta en vez de un informe en ceros.

## Por qué "success" en GHA no es suficiente

GHA marca el job como "success" si el proceso Python termina con exit code 0. Un script que recibe 401, swallows the error, y llama `sys.exit(0)` (o simplemente termina) pasa como green aunque no haya hecho su trabajo.

Las alertas de degrade silencioso no se detectan con CI — se detectan con guards en los propios datos.

## Pattern de guard genérico

```python
def fetch_api(url, auth, **kwargs):
    resp = requests.get(url, auth=auth, **kwargs)
    if resp.status_code in (401, 403):
        raise RuntimeError(f"Auth fallida en {url} ({resp.status_code})")
    resp.raise_for_status()
    return resp.json()
```

Un día genuinamente sin datos devuelve 200 con `[]` → se permite. Solo los errores de auth/permiso deben bloquear.
