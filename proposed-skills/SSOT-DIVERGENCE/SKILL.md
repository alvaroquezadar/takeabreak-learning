---
name: ssot-divergence
description: Use when reviewing or refactoring a Python project where the same business logic (order filtering, coupon classification, price calculation) exists inline in multiple scripts. Creating an SSOT module reveals divergence bugs in copies that have drifted silently.
---

## Patrón

Cuando la misma lógica de negocio se copia inline en varios scripts Python, cada copia diverge con el tiempo. El problema es silencioso: los scripts corren, producen números, nadie nota que los números son levemente distintos entre sí.

El acto de extraer la lógica a un módulo SSOT y migrar los scripts a importarlo suele revelar bugs que llevaban meses en producción.

### Qué se encontró en tab-agente (mayo 2026)

`clasificar_cupon()` y `es_pedido_real()` existían inline en `agente_clm_tab.py`, `agente_informe_ventas_web.py` y `agente_email_mensual_tab.py`. Al extraer a `tab_pedido.py`:

| Bug encontrado | Dónde vivía | Impacto |
|---|---|---|
| `startswith("CMTAB4")` en vez de `'cmtab' in code.lower()` | `agente_informe_ventas_web.py` | 4.212 pedidos CMTAB4 categorizados como "Otros/Corporativo" |
| `is_real_sale` solo miraba `total > 0`, sin filtro de status | `agente_informe_ventas_web.py` + `agente_email_mensual_tab.py` | $90M s/IVA sobre-reportado históricamente (3-15% según mes) |
| Cupones Falabella sin validar día martes | `agente_informe_ventas_web.py` | 60 pedidos mal clasificados como Falabella |

El mismo bug de `is_real_sale` estaba en DOS scripts distintos con distinta severidad: en `informe_ventas_web` se había corregido el status pero quedaba la copy en `email_mensual` sin corregir (commits 4b8bdc0 y 9a062d8 — 3 días después del primero).

## Por qué importa

- Los scripts con lógica inline siguen produciendo output válido — no crashean. Los números parecen razonables.
- Los errores se descubren meses después comparando reportes, no por alertas.
- Una SSOT migration fuerza a auditar cada copia: la mera pregunta "¿es igual a la SSOT?" encuentra el diverge.

## Cómo hacer la migración

```python
# tab_pedido.py (SSOT)
REAL_STATUSES = {"processing", "completed", "on-hold"}
CUPONES_EXCLUIR = {"influencer100", "toolkit100"}

def es_pedido_real(order: dict) -> bool:
    if order.get("status") not in REAL_STATUSES:
        return False
    if float(order.get("total", 0)) == 0:
        return False
    coupons = order.get("coupon_lines", [])
    if any(c.get("code", "").lower() in CUPONES_EXCLUIR for c in coupons):
        return False
    return True

def clasificar_cupon(order: dict) -> str:
    coupons = order.get("coupon_lines", [])
    codes = [c.get("code", "").lower() for c in coupons]
    if any("cmtab4" in c for c in codes):
        return "cmtab4"
    # ... resto de la clasificación
```

```python
# agente_informe_ventas_web.py (migrado)
from tab_pedido import es_pedido_real, clasificar_cupon

# Borra las copias inline. Importa y usa.
for order in orders:
    if not es_pedido_real(order):
        continue
    tipo = clasificar_cupon(order)
```

## Checklist de migración SSOT

- [ ] Listar todos los archivos que tienen la lógica duplicada (`grep -r "es_cmtab\|is_real\|clasificar" .`)
- [ ] Crear módulo SSOT con la implementación más correcta (no la más reciente)
- [ ] Por cada copia inline: comparar con SSOT línea a línea — anotar cualquier diferencia
- [ ] Migrar una a una, corriendo tests entre cada migración
- [ ] Si hay test de paridad, úsalo: verifica que las implementaciones siguen alineadas
- [ ] Documentar el cambio metodológico en el email/informe si el número cambia
