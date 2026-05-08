---
name: brevo-event-types
description: Use when writing Python code that consume eventos de Brevo (Sendinblue) vía webhook o API de estadísticas. La API devuelve variantes del mismo tipo de evento y Gmail genera aperturas sintéticas.
---

## Patrón

Brevo no tiene un esquema de event types estable: el mismo evento puede aparecer con nombres distintos dependiendo del endpoint (webhook vs. stats API vs. transaccional).

### Variantes documentadas (mayo 2026)

| Evento           | Variantes observadas                                              |
|------------------|-------------------------------------------------------------------|
| Enviado          | `request`, `requests`                                             |
| Entregado        | `delivered`                                                       |
| Apertura         | `opened`, `opens`, `open`, `first_opening`, `unique_opened`, `loadedbyproxy`, `loaded_by_proxy` |
| Clic             | `clicks`, `click`, `unique_clicked`                               |
| Soft bounce      | `soft_bounce`, `soft_bounces`, `softbounce`, `softbounces`        |
| Hard bounce      | `hard_bounce`, `hard_bounces`, `hardbounce`, `hardbounces`        |
| Spam             | `spam`                                                            |
| Bloqueado        | `blocked`                                                         |

**Nota importante sobre `loadedbyproxy`:** Gmail y otros clientes corporativos precargan el pixel de tracking a través de un proxy de seguridad. Esto genera eventos `loadedbyproxy` o `loaded_by_proxy` que son aperturas reales del usuario (o muy cercanas). Deben contarse como `opens`.

### Campo del evento

La API a veces devuelve el tipo en `event`, a veces en `message`. Normalizar:

```python
ev = (e.get("event") or e.get("message") or "").lower()
```

## Por qué importa

Si el agregador solo reconoce un subconjunto de variantes, las métricas de apertura quedan subcontadas. Esto afecta decisiones de SAC y evaluación de campañas. El problema apareció en producción en tab-agente el 6 mayo 2026 (commits b2ae426 y fa34224).

## Implementación recomendada

```python
from collections import Counter

OPEN_EVENTS   = {"opened", "opens", "open", "first_opening", "unique_opened",
                  "loadedbyproxy", "loaded_by_proxy"}
CLICK_EVENTS  = {"clicks", "click", "unique_clicked"}
S_BOUNCE      = {"soft_bounce", "soft_bounces", "softbounce", "softbounces"}
H_BOUNCE      = {"hard_bounce", "hard_bounces", "hardbounce", "hardbounces"}
REQUEST_EVENTS = {"request", "requests"}

def aggregate(events):
    counts = {
        "requests": 0, "delivered": 0,
        "opens": 0, "unique_opens": set(),
        "clicks": 0, "unique_clicks": set(),
        "soft_bounces": 0, "hard_bounces": 0, "spam": 0, "blocked": 0,
    }
    unknown = Counter()
    for e in events:
        ev = (e.get("event") or e.get("message") or "").lower()
        em = e.get("email", "")
        if ev in REQUEST_EVENTS:        counts["requests"] += 1
        elif ev == "delivered":          counts["delivered"] += 1
        elif ev in OPEN_EVENTS:
            counts["opens"] += 1
            if em: counts["unique_opens"].add(em)
        elif ev in CLICK_EVENTS:
            counts["clicks"] += 1
            if em: counts["unique_clicks"].add(em)
        elif ev in S_BOUNCE:             counts["soft_bounces"] += 1
        elif ev in H_BOUNCE:             counts["hard_bounces"] += 1
        elif ev == "spam":               counts["spam"] += 1
        elif ev == "blocked":            counts["blocked"] += 1
        else:
            unknown[ev or "(empty)"] += 1
    counts["unique_opens"]  = len(counts["unique_opens"])
    counts["unique_clicks"] = len(counts["unique_clicks"])
    if unknown:
        print(f"  ⚠ Event types desconocidos (no contados): {dict(unknown)}")
    return counts
```

Loggear los `unknown` permite detectar nuevas variantes antes de que afecten las métricas.
