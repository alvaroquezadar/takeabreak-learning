---
name: bsale-guard-cero
type: feedback
description: BSale rechaza documentos con monto $0 (error doc_001 "document amount is less than the minimum allowed"). Las órdenes de entregas de plan prepagado en TAB son siempre $0 y no necesitan documento tributario — la boleta se emite en la orden madre. El guard PHP debe saltarse estas órdenes antes de llamar a BSale.
---

## Modelo de negocio (contexto TAB)

- Plan Mensual = orden madre con monto real → **boleta BSale en esa orden**
- Cada entrega semanal posterior = orden $0 con cupón `cmtab4` → **no emite documento**

La integración BSale (`crearBoletaFactura()` en `Impreza-child/functions.php`)
corría para TODAS las órdenes en `processing`, incluidas las de $0.
BSale las rechazaba con `doc_001`, o las aceptaba emitiendo una boleta de $0
(folio gastado inútilmente).

## Señal de alerta

Cadena de emails "BSale FALLO — pedido X" tras un deploy que **agregó alertas**.
Antes pasaba en silencio; el deploy solo hizo visible un fallo preexistente.
Diagnóstico: comparar `diff functions.php.bak-<deploy> functions.php` para
aislar qué cambió realmente.

## Guard aplicado

```php
// Al inicio de crearBoletaFactura()
if ( (float) $order->get_total() <= 0 ) {
    $order->add_order_note(
        'BSale OMITIDO: orden total $0 (plan prepagado / sin monto) - no se emite documento.'
    );
    return;
}
```

Esto corta: (a) el intento de emisión BSale, (b) el email de alerta interna.
**No toca la confirmación al cliente** — esa la maneja `customer_processing_order`
de WC core, que tiene su propio listener.

## Puntos clave de diagnóstico

1. Ver `bsale_respuesta` en el meta de la orden (WC REST GET, read-only)
2. Confirmar que la orden madre tiene boleta → las entregas $0 no necesitan una
3. `crearBoletaFactura` → su `wp_mail` va a `alvaro@`, no al cliente

## Rollback

```bash
cp functions.php.bak-zeroguard-YYYYMMDD Impreza-child/functions.php
```
