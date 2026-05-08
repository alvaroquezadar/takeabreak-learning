---
name: gspread-quota
description: Use when writing Python code that accesses Google Sheets via gspread and may make repeated calls in a loop or batch process. Covers quota exhaustion (429) and exception masking patterns.
---

## Patrón

Dos errores recurrentes en proyectos Python con gspread que generan fallos en producción:

### 1. gc.open_by_key() en loop = quota 429

Cada `gc.open_by_key(SHEET_ID)` consume 1 read del quota de Sheets API (60/min por Service Account).
En un proceso que hace flush cada N ítems (CLM, batch logging), esto escala rápido.

**Síntoma:** `gspread.exceptions.APIError: {'code': 429, 'message': 'Quota exceeded'}`
**Cuándo aparece:** corridas con > 60 operaciones en 1 minuto (journeys grandes, Falabella, Black Friday).

**Fix:** cachear el handle por sesión.

```python
# tab_sheets.py — patrón recomendado
_sheet_cache: dict = {}

def get_sheet_cached(gc, sheet_id: str):
    key = (id(gc), sheet_id)
    if key not in _sheet_cache:
        _sheet_cache[key] = gc.open_by_key(sheet_id)
    return _sheet_cache[key]
```

### 2. except Exception enmascarando 429 como WorksheetNotFound

Cuando un bloque `try: sh.worksheet("nombre")` usa `except Exception` para manejar "hoja no existe",
un 429 que llega en ese momento se captura igual, y el código intenta crear una hoja que ya existe → 400.

**Síntoma:** `gspread.exceptions.APIError: {'code': 400, 'message': 'Sheet already exists'}` seguido de crash.
**Por qué ocurre:** el 429 llega exactamente durante el `sh.worksheet()`, no durante la escritura.

**Fix:** usar la excepción específica.

```python
from gspread.exceptions import WorksheetNotFound

try:
    ws = sh.worksheet("Log_Comunicaciones")
except WorksheetNotFound:
    ws = sh.add_worksheet("Log_Comunicaciones", rows=100000, cols=len(HEADERS))
    ws.update(values=[HEADERS], range_name='A1')
# Los 429 ahora se propagan al handler general, que tiene retry/backoff.
```

## Por qué importa

Ambos errores se produjeron en producción en tab-agente la semana del 5-6 mayo 2026 (commits 0b3803c y 5b2447a), con impacto real en journeys CLM de Falabella. Son silenciosos: el workflow queda rojo pero no hay alerta al usuario final.

## Checklist al revisar código con gspread

- [ ] `gc.open_by_key()` está fuera de loops (o usa `get_sheet_cached`)
- [ ] Bloques `except Exception` alrededor de `sh.worksheet()` cambiados a `except WorksheetNotFound`
- [ ] Hojas nuevas creadas con `rows=100000` (no 10000 — se llena en semanas con logs activos)
