---
name: deploy-visibility-vs-cause
description: Use when an error appears in a burst immediately after a deploy — para distinguir si el deploy causó el error o solo lo hizo visible al agregar alertas/logs. Evita revertir el deploy equivocado mientras el bug real queda sin fix.
---

## El patrón

Un deploy agrega logging, email de alerta, o manejo de error. De repente
aparecen N errores en cadena del mismo tipo. Conclusión apresurada: "el deploy
lo rompió". Conclusión correcta a verificar: "el deploy reveló algo que ya
estaba roto".

Confundir causa con visibilidad lleva a:
- Revertir el deploy (eliminando la visibilidad junto con el bug)
- No corregir el error de fondo
- El bug vuelve a operar en silencio

## Diagnóstico: diff del backup

```bash
# Si el deploy hizo backup (siempre debería):
diff archivo.php.bak-<nombre-deploy> archivo.php

# Preguntas clave al leer el diff:
# 1. ¿El diff agrega un wp_mail / error_log / try-catch que antes no existía?
# 2. ¿La lógica de negocio de la sección con errores cambió?
# 3. ¿Los errores reportados son de órdenes ANTERIORES al deploy?
```

Si el error aparece en órdenes/eventos anteriores al deploy → el deploy no
los causó, solo los detectó.

## Señales de "el deploy reveló, no causó"

| Señal | Interpretación |
|-------|---------------|
| Errores en datos pre-deploy | Bug preexistente |
| El diff solo agrega alertas | El deploy no tocó la lógica |
| La alerta email es nueva pero el HTTP 400 BSale ya existía en logs | Visibilidad nueva, error viejo |
| N errores iguales en burst post-deploy | El bug acumulaba en silencio |

## Señales de "el deploy causó el error"

| Señal | Interpretación |
|-------|---------------|
| Error en datos creados DESPUÉS del deploy | Regresión real |
| El diff cambia la lógica de negocio, no solo el alerting | Posible bug nuevo |
| Los logs anteriores al deploy son limpios | El deploy introdujo algo |

## Protocolo mínimo

```bash
# 1. Identificar el deploy exacto (diff)
diff backup.php actual.php | less

# 2. Verificar si los registros afectados son anteriores o posteriores al deploy
# (timestamps, IDs de orden, etc.)

# 3. Separar: ¿qué cambió en el comportamiento vs. qué cambió en la visibilidad?
```

## Caso real (TAB, 2026-06-14)

Deploy "bsalefix" agregó `wp_mail` de alerta cuando BSale falla.
Apareció cadena de N emails "BSale FALLO" para órdenes $0 de planes prepagados.
Conclusión correcta: BSale fallaba para esas órdenes desde siempre; el deploy
solo agregó el aviso. Fix: guard PHP para saltar órdenes ≤ $0 (no revertir el deploy).

## Regla general

> Cuando agregas alertas, encontrarás errores que ya existían.
> El valor del deploy es que ahora los puedes ver.
> El fix correcto es arreglar el error de fondo, no silenciar la alerta.
