---
name: monitor-auth-false-positive
type: feedback
description: El monitor de uptime marcaba DOWN la API REST de WC (401) tras el cierre de PII del 11-jun que exigió auth. Fix: lista extra_alive por URL que acepta códigos 4xx específicos como "vivo".
---

## Patrón

Un monitor de HTTP que trata cualquier código `>= 400` como caída generará **falsos positivos** si un endpoint protegido devuelve 401/403 de forma esperada (por ejemplo, porque se activó auth donde antes no había).

### Caso real (tab-agente, 2026-06-21)

Tras el fix del 11-jun que cerró PII (`/wp-json/wc/v3/system_status` pasó a exigir credenciales), el monitor marcaba "CAÍDO" cada ~2-3h porque la API devolvía 401 aunque el stack estuviera completamente operativo (homepage en 200).

**Síntoma:** emails de alerta de caída con el sitio funcionando normalmente.

## Fix aplicado

4º campo `extra_alive: frozenset` por URL. Códigos en ese set cuentan como UP aunque sean 4xx:

```python
URLS = [
    ("Homepage",  "https://takeabreak.cl/",                    None,   frozenset()),
    ("API REST",  "https://takeabreak.cl/wp-json/wc/v3/...",  auth,   frozenset({401, 403})),
    ("Checkout",  "https://takeabreak.cl/finalizar-compra/",   None,   frozenset()),
]

def check_url(name, url, auth, extra_alive=frozenset()):
    ...
    if status_code in extra_alive:
        return "UP", status_code, None, latency
    if status_code >= 400:
        return "DOWN", status_code, None, latency
```

## Distinción clave

| Respuesta        | Significado                           | Estado correcto |
|-----------------|---------------------------------------|-----------------|
| 401/403 en API  | Stack responde, auth requerida        | UP              |
| 5xx en cualquier URL | Error del servidor               | DOWN            |
| Timeout         | Box no responde                       | DOWN            |
| 200 con latencia > umbral | Lento pero vivo            | SLOW            |

Un 401/403 en un endpoint auth-gated **prueba que el servidor está vivo y la seguridad funciona**. No es caída.

## Trigger de alerta

Este falso positivo se activó porque el cierre de PII (un cambio de seguridad) no incluyó actualizar la configuración del monitor. Checklist post-cambio de auth en cualquier endpoint monitorizado: revisar si el monitor necesita `extra_alive`.
