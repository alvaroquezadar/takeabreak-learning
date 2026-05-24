---
name: external-api-schema-first
description: Use when integrating with any third-party API (Microsoft Clarity, GA4, WooCommerce, Brevo, etc.) before writing the parser or data model. La documentación oficial frecuentemente no refleja el schema real del JSON — siempre validar la respuesta real primero.
---

## Patrón

Las APIs de terceros documentan sus campos con nombres en prosa o con formatos que no coinciden con el JSON real. Escribir el parser basado en la documentación produce código que compila y corre sin errores, pero retorna filas vacías o ceros silenciosamente.

**Síntoma:** el script corre sin excepciones, pero todos los valores son 0 o vacíos. Los tests pasan porque fueron escritos con el schema de docs, no con fixtures de respuestas reales.

### Caso real: Microsoft Clarity (mayo 2026)

| Campo en documentación       | Campo real en JSON         |
|-----------------------------|---------------------------|
| "Dead Click Count"          | `subTotal` (en objeto genérico con tipo `DeadClickCount`) |
| "Scroll Depth"              | `averageScrollDepth`      |
| "Engagement Time"           | `activeTime`              |
| "distinct users"            | `distinctUserCount` (el código tenía `distantUserCount`) |

El cliente producía filas vacías en producción durante semanas antes de que se notara.

## Proceso correcto al integrar una API nueva

### 1. Hacer un request real y guardar la respuesta cruda

```python
import json, tempfile, datetime

def fetch_and_save_raw(url, headers, params) -> dict:
    resp = requests.get(url, headers=headers, params=params)
    resp.raise_for_status()
    data = resp.json()
    # Guardar para inspección
    fname = f"/tmp/api_raw_{datetime.date.today()}.json"
    with open(fname, "w") as f:
        json.dump(data, f, indent=2, ensure_ascii=False)
    print(f"Raw guardado en {fname}")
    return data
```

### 2. Inspeccionar el JSON real antes de escribir el parser

```bash
cat /tmp/api_raw_2026-05-18.json | python -m json.tool | head -60
# O para ver todas las claves del primer elemento:
python -c "import json; d=json.load(open('/tmp/api_raw.json')); print(list(d.keys()))"
```

### 3. Escribir el parser con el schema real; crear fixtures desde la respuesta real

```python
# tests/test_mi_cliente.py
# fixture = copia del JSON real (o subset), NO inventado

FIXTURE_REAL = {
    "metrics": [
        {
            "metricType": "DeadClickCount",
            "subTotal": 42,          # <- campo real, no "deadClickCount"
            "sessionsCount": 100
        }
    ]
}

def test_parse_dead_clicks():
    result = _parse_metrics(FIXTURE_REAL, dimensions=["Browser"])
    assert result[0]["dead_click_count"] == 42
```

### 4. Agregar `save_raw` como opción permanente de debug

```python
def fetch_metrics(project_id: str, days: int, save_raw: bool = False) -> list[dict]:
    resp = requests.get(URL, headers=headers, params=params)
    raw = resp.json()
    if save_raw:
        path = tempfile.mktemp(suffix=".json", prefix="clarity_raw_")
        with open(path, "w") as f:
            json.dump(raw, f, indent=2)
        logger.info(f"Raw guardado: {path}")
    return _parse_metrics(raw)
```

## Checklist al integrar una API externa

- [ ] Ejecutar la request real y guardar el JSON antes de escribir `_parse_metrics`
- [ ] Anotar las diferencias entre campos en docs vs. campos reales
- [ ] Fixtures de tests = subsets del JSON real (no inventados desde docs)
- [ ] Agregar `save_raw=True` o `debug=True` como opción de desarrollo
- [ ] Si hay typos en field names: buscar en el código con `grep -r "distantUserCount\|campo_mal_escrito"`

## Por qué importa

Implementado en tab-agente (commit 7c485d8 — mayo 2026, Clarity API). El cliente devolvía filas vacías silenciosamente en producción. El fix requirió reescribir el parser completo y añadir 16 tests con fixtures reales. El debugging habría sido inmediato si se hubiera inspeccionado la respuesta cruda primero.
