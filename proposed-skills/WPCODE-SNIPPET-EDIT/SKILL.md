---
name: wpcode-snippet-edit
description: Use when editing WPCode snippets programmatically in WordPress — para evitar el gotcha de que SQL/WP-CLI no reconstruye el cache wpcode_snippets, dejando el cambio sin efecto en producción hasta un reinicio manual.
---

## Patrón

WPCode guarda los snippets en un custom post type pero **también los serializa
en cache** (`wpcode_snippets` en `wp_options`). Editar el contenido del snippet
por SQL o WP-CLI directamente actualiza la DB pero **no reconstruye el cache**.
El snippet activo sigue siendo el viejo hasta que alguien purga el objeto cache.

En producción esto significa: "apliqué el fix, verifiqué en DB, pero el bug sigue".

## La forma correcta

Usar la API interna de WPCode desde `wp eval-file`:

```php
<?php
// edit_snippet.php
$snippet = WPCode_Snippet_Library::instance()->get_snippet_by_id( 81637 );
$old = $snippet->get_code();
$new = str_replace( 'TOKEN_VIEJO', 'TOKEN_NUEVO', $old );

if ( $old === $new ) {
    echo "ERROR: no se encontró el string a reemplazar\n";
    exit(1);
}

$snippet->set_code( $new );
$result = $snippet->save();

if ( ! $result ) {
    echo "ERROR: save() devolvió false\n";
    exit(1);
}

echo "OK: snippet guardado y cache reconstruido\n";
```

```bash
# CRÍTICO: --user=<admin_id> — sin usuario, save() devuelve false silencioso
wp eval-file edit_snippet.php --user=4
```

## Verificación post-edición

```bash
# Confirmar que el cache tiene el valor nuevo (no el viejo)
wp eval 'echo serialize(get_option("wpcode_snippets"));' | grep -c "TOKEN_NUEVO"
```

## Checklist antes de editar

1. Backup: `wp eval 'echo WPCode_Snippet_Library::instance()->get_snippet_by_id(81637)->get_code();' > /tmp/snippet_81637_backup.php`
2. Verificar que el string a reemplazar existe en el snippet actual
3. `--user=<admin_id>` — el user ID numérico del admin (no el username)
4. Post-edición: contar ocurrencias del nuevo valor en el cache

## Por qué importa

Caso real (TAB, 2026-06-19): swap del token muerto de Meta en el snippet de
WhatsApp (`PacksMensuales`, id 81637). El snippet controló el envío de
confirmaciones WhatsApp de pedidos y cupones de plan mensual. Un edit por SQL
hubiera dejado el canal caído sin evidencia clara de por qué.

## Rollback

```bash
# Restaurar desde backup con la misma API (nunca por SQL)
wp eval-file restore_snippet.php --user=4
```
