---
name: WP-ACTION-SCHEDULER-QUEUE
description: Use when working with WordPress Action Scheduler (as_enqueue_async_action, as_schedule_cron_action) to enqueue background tasks — to diagnose why actions queue but never execute, or to choose between async-queue and synchronous execution for critical paths.
---

# WP Action Scheduler — diagnóstico de cola frenada y bypass sincrónico

## Patrón

WP Action Scheduler puede estar silenciosamente frenado: las acciones se encolan (`pending`)
pero nunca ejecutan. El sitio muestra WP-Cron activo y el runner de AS agendado, pero nada corre.
No hay error en los logs de PHP — el encolado tiene éxito, el no-run es silencioso.

## Por qué importa

Cualquier código que depende de AS para envíos críticos (purchase tracking, emails transaccionales,
integraciones externas) puede parecer live y no hacer nada. El dry-run no detecta esto porque
el dry-run nunca encola. El test post-cutover tampoco si solo verifica que el flag cambió.

## Cómo diagnosticar

```bash
# Ver acciones acumuladas en pending
wp action-scheduler list --status=pending --per-page=20 --orderby=scheduled_date --order=asc

# Si hay acciones de hace horas/días → cola frenada
# Verificar que el runner se auto-agenda
wp cron event list | grep action_scheduler

# Intentar correr la cola manualmente
wp action-scheduler run --limit=5
```

Si `wp action-scheduler run` funciona manual pero no en WP-Cron, el problema es la ejecución
de WP-Cron en el server (PHP-FPM timeout, proceso sin requests HTTP, etc.).

## Bypass: modo sincrónico

Para rutas críticas (purchase tracking, emails transaccionales):

```php
// En lugar de encolar:
// as_enqueue_async_action('mi_action', $args)

// Forzar sincrónico:
if ( false ) {  // deshabilitar AS para esta ruta
    as_enqueue_async_action('mi_action', $args);
} else {
    mi_action_handler($args);  // ejecutar directo
}
```

Reducir timeouts de llamadas externas (GA4/Meta) a ≤2s para no bloquear el request del cliente.

## Regla

Al hacer cutover de cualquier proceso que depende de AS:
1. Después del flip, verificar que las primeras acciones encoladas pasan de `pending` a `complete`
   en ≤2 min.
2. Si en 5 min siguen `pending` → rollback del flag + investigar cola.
3. Para envíos donde el "no enviado" es peor que la latencia extra (purchase tracking, emails
   de ciclo), preferir sincrónico sobre async.
