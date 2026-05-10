---
name: Cohorte — "cliente nuevo" calculado contra ventana filtrada, no histórico completo
type: feedback
description: Las métricas de cohorte (volumen clientes nuevos, LTV, activación) calculaban primera compra sobre los pedidos del período de análisis (6 meses). Resultado: ~1.100 clientes históricos contados como nuevos. Los números de cohorte duplicaban los del informe mensual (3.066 vs ~1.960).
project: tab-agente
date: 2026-05-10
commits:
  - bb22827
---

## Qué pasó

El orquestador de Fase 1 (`fase1_orquestador.py`) filtraba los pedidos al período de 6 meses **antes** de pasarlos a las métricas de cohorte.

`data_loader.cargar_y_limpiar(pedidos, inicio, fin)` devolvía solo los pedidos del período. Luego `volumen_clientes_nuevos_por_mes()` buscaba la primera compra de cada email *dentro de esa lista*, que es siempre la primera aparición en el período — no necesariamente la primera compra histórica.

**Resultado:** un cliente con primera compra en 2024-08 aparecía como "nuevo en noviembre 2025" si su compra más antigua en la ventana era de noviembre.

La discrepancia fue detectada por comparación directa con el agente_email_mensual_tab: 3.066 (orquestador) vs ~1.960 (email mensual). Diferencia de ~1.100 clientes históricos mal clasificados.

## Por qué importa

No genera error ni crash. Los números parecen plausibles aislados. Solo se detecta cuando se comparan dos fuentes distintas del mismo dato. Sin esa validación cruzada, el error pasa a producción sin resistencia.

## Fix aplicado

Nuevo helper `calcular_primeras_compras_historicas(pedidos_raw)` que mira **todo el histórico** (sin filtrar por período) y devuelve `{email: fecha_1ra_compra_histórica}`.

Todas las métricas dependientes reciben este dict como parámetro:
```python
primeras_hist = calcular_primeras_compras_historicas(pedidos_raw)  # sin filtrar

volumen = volumen_clientes_nuevos_por_mes(primeras_hist, inicio, fin)
activacion = tasa_segunda_compra(pedidos_hist, primeras_hist, cohorte_mes, ventana)
ltv = ltv_por_cohorte(pedidos_hist, primeras_hist, cohorte_mes, ventana)
```

Los pedidos filtrados al período (`pedidos_hist`) se usan para medir actividad *dentro del período*, pero la pertenencia a cohorte se determina siempre desde el histórico completo.

## Regla derivada

> En cualquier análisis de cohorte, **la pertenencia a cohorte se calcula sobre el histórico completo** — nunca sobre el slice del período de análisis.

Si el script filtra los pedidos antes de calcular `primera_compra`, el resultado es incorrecto para clientes cuya primera compra está fuera de la ventana.

## Defensa sugerida

```python
# CORRECTO: primeras compras desde histórico sin filtrar
primeras_hist = calcular_primeras_compras_historicas(pedidos_raw)

# INCORRECTO: primera aparición en el período ≠ primera compra histórica
primeras_incorrectas = {
    p["email"]: p["fecha"]
    for p in sorted(pedidos_filtrados, key=lambda x: x["fecha"])
    # ... si pedidos_filtrados solo tiene 6 meses, está mal
}
```

Test de regresión que protege este patrón:
```python
def test_volumen_no_cuenta_cliente_con_primera_compra_anterior_al_periodo():
    pedidos_raw = [
        make_order("a@x.com", date(2024, 8, 1)),   # histórico — fuera del período
        make_order("a@x.com", date(2025, 11, 15)),  # dentro del período
    ]
    primeras = calcular_primeras_compras_historicas(pedidos_raw)
    volumen = volumen_clientes_nuevos_por_mes(primeras, date(2025, 11, 1), date(2026, 5, 1))
    assert volumen.get(date(2025, 11, 1), 0) == 0  # NO es nuevo en nov-25
```
