---
name: f3-action-scheduler-silent-stall
type: feedback
description: El cutover de F3 (purchase server-side) parecía live pero no enviaba nada. La Action Scheduler queue de WordPress estaba silenciosamente frenada — encolaba bien, nunca ejecutaba. Fix: modo sincrónico forzado.
---

# Patrón: WP Action Scheduler silenciosamente frenado — diagnóstico y bypass

## Qué pasó

El 02-jul se hizo el cutover de F3 (`purchase` server-side) de dry-run a live. Todo pareció OK:
`TAB_PURCHASE_SS_DRY_RUN=false`, php-lint OK, log sin errores.

El 04-jul, revisando órdenes reales post-flip, se descubrió que F3 no estaba enviando nada.

**Síntomas:**
- Órdenes pagadas sin `_tab_purchase_sent` (el flag que F3 escribe al enviar OK).
- 116 acciones `tab_pss_send_real` en Action Scheduler con estado `pending`, acumuladas sin ejecutar.
- También MailChimp y FB heartbeat apilados → la cola estaba frenada site-wide.

WP-Cron y el AS runner estaban configurados pero no corrían. La causa raíz es infra del server
(no resuelta aún). F3 encolaba correctamente; la cola nunca drenaba.

**Daño colateral:** 36 acciones alcanzaron a completarse antes de cancelarlas → esas enviaron CAPI
a Meta duplicando con el plugin `facebook-for-woocommerce` (ruido acotado Jul 2-4).

## Cómo se diagnosticó

```bash
wp action-scheduler list --status=pending --per-page=20
# Si se acumulan acciones pending sin ejecutar por horas → cola frenada.
wp cron event list
# Verificar que el runner de AS esté agendado Y ejecutándose.
```

## Fix aplicado

Modo sincrónico forzado: en el hook `tab_pss_handle`, cambiar
```php
if ( function_exists('as_enqueue_async_action') )
```
por
```php
if ( false )   // fuerza envío directo, no depende de la cola
```
Timeout GA4/Meta reducido 6→2 s (para no bloquear el checkout del cliente).

Adicionalmente, F3 quedó GA4-only (comentado `TAB_META_CAPI_TOKEN`) para evitar doble conteo con
el plugin Meta, que ya maneja su propio CAPI.

## Regla general

Al integrar con WP Action Scheduler:
1. Nunca asumir que la cola está activa por el hecho de que el runner esté configurado.
2. Primer test post-cutover: verificar que las acciones encoladas cambian de `pending` a `complete`
   en ≤2 min (WP-Cron interval típico).
3. Si se detecta que AS está frenado site-wide, preferir modo sincrónico para envíos críticos
   (purchase tracking, emails transaccionales) en lugar de depender de la cola.
4. Backup de wp-config antes de cada flip de flag de producción.
