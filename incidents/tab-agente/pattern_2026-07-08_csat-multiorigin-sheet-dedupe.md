---
name: csat-multiorigin-sheet-dedupe
type: feedback
description: El dedupe de recuperar_csat_brevo fallaba para filas nativas de Typeform porque leía el order_id de columna fija (C), válida solo para filas escritas por el propio script. Typeform escribe el pedido en columna F.
---

# Patrón: Dedupe en Sheets falla cuando las filas tienen distintos orígenes

## Qué pasó

`recuperar_csat_brevo.py` — función `obtener_emails_ya_registrados()` construía la clave de dedupe
como `f"{email}_{order_id}"` leyendo el `order_id` siempre de columna C (índice 2).

Esa columna la escribe el propio script al insertar una fila nueva. Pero las filas que Typeform
escribe directamente tienen el `order_id` en columna F (índice 5). Resultado: esas filas quedaban
con `order_id` vacío en la clave → `"marizavabus@gmail.com_"` en lugar de
`"marizavabus@gmail.com_115074"`.

Cuando Brevo volvía a traer el click de esa respuesta horas después, el script no la reconocía como
ya registrada y la insertaba duplicada. Los duplicados inflaban el conteo de detractores.

**Caso real detectado (Jul 2026):** 2 de los "4 detractores de julio" eran duplicados:
- `marizavabus@gmail.com` (pedido 115074)
- `socampo@gmail.com` (pedido 114597)

## Fix aplicado

`extraer_email_order_de_fila()` — función nueva que detecta el origin de la fila (columna C vacía →
es nativa de Typeform → leer de columna F; C no vacía → es fila del script → leer de columna C).
La clave de dedupe siempre tiene `order_id` correcto independiente del origen.

Test de regresión: fila nativa + candidato del mismo pedido → se detecta como duplicado.

## Regla general

Antes de construir una clave de dedupe en un Sheet compartido con múltiples writers:
1. Verificar si hay escritores distintos con layouts distintos en el mismo Sheet.
2. Si los hay, la función de lectura debe ser origin-agnostic (detectar la fuente por contenido,
   no por posición fija de columna).
3. El test debe incluir al menos un caso de cada origen.
