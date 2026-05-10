---
name: ga4-attribution-meta-ai
description: Use when building a GA4 attribution mapping in Python that classifies (source, medium) pairs into high-level marketing channels. Meta family ads and AI search engines require special handling that the standard medium-based mapping misses.
---

## Patrón

El mapping estándar de GA4 (clasificar por `medium`) falla en dos fuentes comunes:

### 1. Meta family usa `medium=paid`, no `medium=social`

Las campañas de Facebook, Instagram, Threads y Audience Network llegan a GA4 con `medium=paid` (o en algunos casos sin medium), **no** con `medium=social` ni `medium=paid_social`.

Si el mapping clasifica `paid` → `paid_search` o lo ignora, el tráfico de Meta cae a "otros". En TAB (mayo 2026), esto causó que el 68.8% del tráfico quedara en "otros" — el canal más grande era ruido de clasificación.

**Fuentes de Meta family:**
```python
SOURCES_META_FAMILY = (
    "facebook", "fb", "instagram", "ig", "an", "th",
    "fb-sitelink",
)
```

Regla: si `source in SOURCES_META_FAMILY` **y** `"paid" in medium` → `paid_social`.

### 2. AI search engines llegan como `referral` con sources no mapeados

ChatGPT, Copilot, Perplexity y similares mandan tráfico con sus dominios como source y `medium=referral`.
Sin mapeo explícito, aparecen en el bucket "otros" o mezclados con referral orgánico.

```python
SOURCES_AI_REFERRAL = ("chatgpt.com", "copilot.com", "perplexity")
```

Si se quiere separar (canal "ai_search"), agregar a la lógica. Si no, al menos distinguirlos del referral orgánico en comentarios.

### 3. `(not set)` y `(data not available)` son tracking incompleto, no canal desconocido

Meter `(not set)` en "otros" diluye la categoría. Tiene su propio bucket `not_set` para distinguir:
- `otros` = canal presente pero no reconocido
- `not_set` = GA4 no pudo determinar el canal (consent, dark social, etc.)

## Implementación recomendada (Python)

```python
SOURCES_META_FAMILY = ("facebook", "fb", "instagram", "ig", "an", "th", "fb-sitelink")
SOURCES_AI_REFERRAL = ("chatgpt.com", "copilot.com", "perplexity")

CANAL_POR_MEDIUM = {
    "cpc":          "paid_search",
    "ppc":          "paid_search",
    "paidsearch":   "paid_search",
    "paid":         "paid_social",   # Meta usa medium=paid
    "social":       "paid_social",
    "paid_social":  "paid_social",
    "(none)":       "direct",
    "referral":     "referral",
    "referral_sm":  "referral",
    "email":        "email",
    "organic":      "organic",
    "not set":      "not_set",       # separado de "otros"
    "(not set)":    "not_set",
    "(data not available)": "not_set",
}

def clasificar_canal(source: str, medium: str) -> str:
    src = (source or "").lower().strip()
    med = (medium or "").lower().strip()

    # (not set) tiene bucket propio
    if src in ("(not set)", "(data not available)") or med in ("(not set)", "(data not available)"):
        return "not_set"

    # Meta family: source primero, medium contiene "paid"
    if src in SOURCES_META_FAMILY and "paid" in med:
        return "paid_social"

    # AI search como referral propio
    if src in SOURCES_AI_REFERRAL:
        return "referral"   # o "ai_search" si se quiere separar

    # Mapping estándar por medium
    canal = CANAL_POR_MEDIUM.get(med)
    if canal:
        return canal

    # Fallback conservador: medium contiene "paid" → paid_social
    if "paid" in med:
        return "paid_social"

    return "otros"
```

## Cómo ampliar el mapping

Cuando aparece tráfico inesperado en "otros", loggear los `(source, medium)` para auditar:

```python
from collections import defaultdict

otros_acum = defaultdict(int)
for row in rows_ga4:
    if clasificar_canal(row["source"], row["medium"]) == "otros":
        otros_acum[f"{row['source']}/{row['medium']}"] += int(row.get("newUsers", 0))

top_otros = sorted(otros_acum.items(), key=lambda kv: kv[1], reverse=True)[:20]
for k, v in top_otros:
    print(f"  {v:>8}  {k}")
```

Esto permite expandir `SOURCES_META_FAMILY`, `SOURCES_AI_REFERRAL` o `CANAL_POR_MEDIUM` con datos reales.

## Por qué importa

Si "otros" es el canal más grande en el reporte, el dato no es útil para decisiones de inversión en medios. El mapping incorrecto hace que Meta — frecuentemente el canal de mayor gasto en e-commerce chileno — sea invisible en el análisis de adquisición.

Documentado en tab-agente (commits f24a974 y ad7a8aa, mayo 2026), donde el 68.8% del tráfico cayó a "otros" antes del fix.
