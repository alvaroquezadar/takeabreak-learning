---
name: silent-failure-guard
description: Use when a Python script reads from an external API (WooCommerce, Brevo, Google Ads, GA4, etc.) and uses that data to generate a report or send an email — para verificar que errores de auth (401/403) levantan excepción en vez de devolver lista vacía, y que GitHub Actions "success" no puede enmascarar output incorrecto.
---

## Patrón

Scripts que leen de APIs externas pueden recibir HTTP 401/403 y devolver `[]` o `{}` en vez de fallar. El workflow de CI termina con exit code 0 ("success") y el script emite un informe con datos en cero. El error es invisible hasta que alguien nota que los números no tienen sentido.

### Caso real (tab-agente, 2026-06-11)

WC REST API retornó 401 por auth incorrecta. `get_ventas()` devolvió `[]`. El script calculó $0 / 0 pedidos y envió el informe a ejecutivos. GHA marcó el job como "success". La falla estuvo activa ~24h antes de ser detectada visualmente.

## Guard estándar

```python
def get_ventas(fecha: str) -> list[dict]:
    resp = requests.get(
        f"{WC_URL}/orders",
        auth=(os.environ["WC_KEY"], os.environ["WC_SECRET"]),
        params={"date": fecha},
    )
    if resp.status_code in (401, 403):
        raise RuntimeError(
            f"API auth fallida ({resp.status_code}) en {resp.url} — "
            "verificar credenciales"
        )
    resp.raise_for_status()
    return resp.json()
```

## Lógica de distinción: vacío legítimo vs. error

| Caso | HTTP status | Acción correcta |
|------|------------|----------------|
| Día sin pedidos | 200 con `[]` | Permitir — emitir informe con ceros |
| Auth incorrecta | 401 / 403 | `raise RuntimeError` — nunca emitir |
| Error del servidor | 500 | `raise_for_status()` — nunca emitir |
| Rate limit | 429 | `raise RuntimeError` con indicación de retry |

## Integración con alertas

Si el script usa un decorator `@con_alerta` o similar, el `RuntimeError` dispara un email de alerta a ops en vez de un informe en ceros a clientes/ejecutivos.

```python
@con_alerta  # email a alvaro@ si hay excepción no capturada
def main():
    ventas = get_ventas(hoy)
    enviar_informe(ventas)
```

## Por qué GHA "success" no es suficiente

GHA marca "success" cuando el proceso termina con exit code 0. Un script que captura silenciosamente el 401 y llama `sys.exit(0)` pasa como green aunque el informe sea incorrecto. Los guards deben estar en el código, no en el CI.

## Auditoría rápida

```bash
# Buscar funciones que hacen requests y podrían swallow errores
grep -n "\.json()" *.py | grep -v "raise_for_status\|raise Runtime"
```

Cualquier llamada `.json()` sin un check de status previo es candidata a degrade silencioso.
