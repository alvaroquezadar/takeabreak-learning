---
name: Pack enum divergence — KeyError silencioso en informe ventas web
type: feedback
description: get_pack_type() y el dict de métricas quedaron desincronizados cuando se agregó pack_28. El informe falló 5 días consecutivos sin alerta.
project: tab-agente
date: 2026-05-08
commits:
  - f141e76
---

## Qué pasó

`get_pack_type()` detecta packs por número de platos y retorna un string como `'pack_28'`.
`calc_metrics()` inicializaba el dict de métricas con llaves fijas: `pack_5`, `pack_7`, `pack_14`, `plan_mensual`, `addon`.

Cuando llegó el primer pedido de Pack 28 platos, `m["packs"]["pack_28"]` lanzó `KeyError`.

**Resultado:** 5 corridas consecutivas falladas (03→07-may-2026), sin informe diario de ventas.

## Por qué importa

El bug no tenía alerta propia — solo se notó porque no llegó el informe. En producción con alertas de fallo genéricas, este tipo de error se vuelve invisible días.

## Fix aplicado

Agregar `"pack_28": 0` al init de `packs` y `packs_n` en `calc_metrics()`. Agregar fila HTML correspondiente.

## Regla derivada

Cada vez que `get_pack_type()` reconoce un tipo nuevo, los dicts de métricas y las tablas HTML deben actualizarse en el mismo commit.

Patrón de riesgo: función que retorna strings dinámicos + dict inicializado con llaves hardcodeadas = divergencia garantizada si cambia el catálogo.

## Defensa sugerida

```python
# Al inicializar métricas, construir desde la lista canónica:
PACK_TYPES = ["pack_5", "pack_7", "pack_14", "pack_28", "plan_mensual", "addon"]
m = {"packs": {k: 0 for k in PACK_TYPES}, "packs_n": {k: 0 for k in PACK_TYPES}}
```

Así, agregar un tipo nuevo en `PACK_TYPES` actualiza automáticamente el dict de métricas.
