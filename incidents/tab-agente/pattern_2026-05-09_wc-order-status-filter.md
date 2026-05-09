---
name: WooCommerce — filtro de status incompleto en is_real_sale
type: feedback
description: Filtrar ventas reales WC solo por total > 0 es incorrecto. Pedidos cancelled/failed/refunded pueden tener total > 0 y contaminan métricas de revenue. El error estuvo activo en dos scripts distintos durante meses.
project: tab-agente
date: 2026-05-09
commits:
  - 4b8bdc0
  - 9a062d8
---

## Qué pasó

`agente_informe_ventas_web.py` y `agente_email_mensual_tab.py` tenían inline su propia función `is_real_sale()` / `es_cmtab()`. Ambas usaban `total > 0` como único filtro.

WooCommerce permite que pedidos en status `cancelled`, `failed`, y `refunded` conserven `total > 0` en el objeto JSON. El total refleja lo que el cliente intentó pagar, no lo que efectivamente se cobró.

**Impacto en revenue reportado mensual:**

| Mes | Antes | Después | Diferencia |
|-----|-------|---------|-----------|
| Mar 2026 | $112,7M | $106,3M | -5,7% |
| Abr 2026 | $87,2M | $73,7M | -15,5% |
| May 2026 | correcto desde inicio | — | — |

Pedidos contados incorrectamente en el historial:
- 1.519 pedidos `cancelled`
- 289 pedidos `failed`
- 1 pedido `refunded`

El bug en `informe_ventas_web` se corrigió el 8-may (commit 4b8bdc0) al migrar a `tab_pedido.py` SSOT. La copia en `email_mensual` se encontró y corrigió 3 días después (commit 9a062d8), demostrando que el bug sobrevivía porque la lógica era inline y no había mecanismo para detectar la divergencia.

## Por qué importa

El error no produce alertas ni crashes. Los números parecen razonables porque la magnitud de los pedidos cancelados (~3-5% habitual, ~15% en meses con más cancelaciones) está dentro del ruido esperado por variación de demanda.

El patrón de riesgo: cualquier función `is_real_sale(order)` que no tenga una whitelist de status explícita es incorrecta para WooCommerce.

## Regla derivada

```python
# tab_pedido.py — patrón correcto para TAB
REAL_STATUSES = {"processing", "completed", "on-hold"}

def es_pedido_real(order: dict) -> bool:
    if order.get("status") not in REAL_STATUSES:
        return False
    if float(order.get("total", 0)) == 0:
        return False
    coupons = order.get("coupon_lines", [])
    if any(c.get("code", "").lower() in CUPONES_EXCLUIR for c in coupons):
        return False
    return True
```

`on-hold` se incluye: son pedidos pagados esperando confirmación (transferencia bancaria, Pluxee, Amipass). Excluir `on-hold` subestimaría revenue de métodos de pago no instantáneos.

## Defensa: no repetir este bug

- Toda función de filtro de ventas reales debe tener status whitelist explícita.
- La lógica vive en `tab_pedido.es_pedido_real()`. No copiar inline en nuevos agentes — importar.
- Si se agrega un nuevo status relevante (ej. WC custom status de cliente corporativo), actualizar `REAL_STATUSES` en `tab_pedido.py` — aplica a todos los agentes automáticamente.
