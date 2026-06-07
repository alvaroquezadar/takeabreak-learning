---
name: Email mensual — proyección estacional por día de semana (efecto Falabella)
type: feedback
description: Semana 2026-06-06 — la proyección lineal inflaba ~$14M al inicio de mes porque no ponderaba el peso diferencial de los martes (efecto Falabella). Reemplazada por proyección que calibra pesos DOW desde historial de 3 meses.
---

**El problema:**

El email mensual proyectaba revenue fin de mes como `promedio_diario × días_restantes`. El martes pesa ~3.1× el promedio diario en TAB por el efecto Falabella: los cupones de 6 dígitos de Falabella se canjean masivamente en ese día. Un mes que arranca con 1 martes en los primeros 5 días inflaba la proyección en ~$14M respecto a la realidad.

**La solución (commit `bf99ae0`):**

`calcular_pesos_dow(meses_hist=3)` — agrega revenue por fecha calendario desde `JSON_HIST`, promedía por día de semana, normaliza a `promedio=1.0`. La proyección multiplica cada día restante por su peso calibrado en lugar de usar el promedio global.

```python
# Peso real calibrado desde historial:
# lun: ~0.7  mar: ~3.1  mie: ~0.8  jue: ~0.9  vie: ~1.2  sab: ~0.9  dom: ~0.4
pesos = calcular_pesos_dow(meses_hist=3)
proyeccion = sum(peso_dia[d] for d in dias_restantes) * netas_hoy / dias_transcurridos
```

El email ahora incluye en la nota explicativa cuántos martes quedan en el mes.

**Condición de fallback:**

Si `JSON_HIST` no existe o tiene datos insuficientes, cae a proyección lineal simple. El código no rompe en entornos sin historial.

**Lección:**

Para negocios con concentración de ventas en días específicos de la semana, la proyección lineal es estructuralmente incorrecta. Verificar si el efecto existe antes de asumir que la proyección es válida. En TAB: siempre revisar el peso de los martes en proyecciones parciales de mes.
