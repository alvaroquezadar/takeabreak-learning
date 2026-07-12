---
name: primer-ciclo-lanzamiento
type: feedback
description: Lanzamiento del carril primer_ciclo (CRM orgánico) — tres bugs descubiertos al armar el template real: unsubscribe links muertos, recomendados vacíos, y el resolver de packs ignoraba bolsas BO_.
---

# Patrón: Tres bugs de template CRM detectados al armar el primer email real

## Qué pasó

Al construir el template `primer_ciclo` para envío a producción (Jul 2026) se detectaron 3 bugs
que en dry-run no eran visibles porque el dry-run no renderiza HTML ni resuelve packs:

### Bug 1 — Unsubscribe links muertos (`href="#"`)

Los 7 templates transaccionales tenían `href="#"` en el link de baja. Brevo **no reemplaza
automáticamente** este href con su tag de opt-out. Solo `{{ unsubscribe }}` funciona.
Un cliente que hace clic en "Dar de baja" no se desuscribe realmente.

Fix: reemplazar `href="#"` por `href="{{ unsubscribe }}"` en todos los templates.

### Bug 2 — Sección "Te recomendamos" siempre vacía

`main_primer_ciclo()` llamaba a `obtener_top_vendidos(pedidos=[])` con lista vacía →
la función devolvía la lista de top vendidos global sin filtrar por historial → la sección
aparecía vacía porque el join con el historial del cliente nunca matcheaba.

Fix: nuevo `_recs_anclas()` que resuelve `ANCLAS_BANDEJA_KEYWORDS` contra stock vivo, independiente
del historial del pedido anterior.

### Bug 3 — Resolver de packs ignoraba bolsas BO_

El mu-plugin `tab-repetir-pack.php` solo elegía/rellenaba bandejas `BA_`. Las bolsas `BO_` dejaban
el pack en estado `'Falta acompañamiento'` — 31% de la cola bloqueada.

Fix: `tab-repetir-pack.php v0.8` incluye soporte para `BO_`. Verificado E2E con Playwright.

## Regla general

Al lanzar un carril de emails nuevo:
1. Renderizar el HTML completo del email antes del primer envío real (no solo dry-run de datos).
2. Verificar el link de unsubscribe con un envío de prueba a una cuenta controlada y hacer clic.
3. Probar el resolver de packs/recomendados con un pedido real de cada tipo de suscripción
   (BA_ y BO_, o cualquier variante del producto).
