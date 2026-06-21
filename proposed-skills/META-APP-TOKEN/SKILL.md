---
name: meta-app-token
description: Use when integrating with Meta Graph API (WhatsApp Business, Facebook) — para manejar correctamente tokens de acceso, evitar tokens hardcodeados que expiran o quedan atados a apps de terceros, y diagnosticar caídas silenciosas de canal.
---

## Patrones críticos

### 1. Token de app de tercero = deuda técnica

Si la integración usa un token perteneciente a una app que no controlas
(creada por una agencia, proveedor, etc.), cualquier cambio en esa cuenta
puede dejarte sin acceso. Señal: el token devuelve
`"API access deactivated. To reactivate, go to developer.facebook.com"` y
generar un token nuevo de esa misma app nace muerto.

**Fix:** crear tu propia app Meta bajo el Business Manager que controlas,
reconectarla a la WABA existente (sin cambiar número ni plantillas), y
generar un token nuevo desde ahí.

### 2. Token permanente = usuario del sistema, no token de página

Los tokens de página expiran o se invalidan cuando el admin que los generó
cambia su contraseña/permisos. Los tokens de usuario normal también.

```
Tipo correcto: System User Token
- Crear en Meta Business Suite → Usuarios del sistema
- Scopes: whatsapp_business_messaging + whatsapp_business_management
- expires_at = 0 → nunca vence
```

Verificar con Graph API antes de guardar:
```bash
curl "https://graph.facebook.com/v21.0/me?access_token=$TOKEN" | jq '.name, .id'
curl "https://graph.facebook.com/v21.0/debug_token?input_token=$TOKEN&access_token=$TOKEN" \
  | jq '.data.expires_at, .data.scopes'
# expires_at debe ser 0
```

### 3. Gotcha: el System User necesita rol de admin en la app

Si `expires_at=0` no aparece en los permisos disponibles al generar el token:
→ Ir a Roles de la App → agregar el usuario del sistema como admin de la app.
Sin ese rol, el token nace con expiración corta sin avisarlo.

### 4. Nunca hardcodear tokens en código PHP/WPCode/snippets

```php
// MAL — el token queda en la DB, logs, backups
$token = "EAABxxx...";

// BIEN — constante en wp-config.php (fuera de la DB, fuera del VCS)
$token = defined('TAB_WA_TOKEN') ? TAB_WA_TOKEN : '';
```

```php
// En wp-config.php (no commitear)
define('TAB_WA_TOKEN', getenv('WA_TOKEN_PROD'));
```

### 5. Fallo mudo = no hay alerta = canal caído por días

```php
// Patrón mínimo para detectar caídas
$response = ... // curl a Graph API
if ( ! $success ) {
    error_log( "WhatsApp FALLO pedido #$order_id: " . $error_msg );
    wp_mail( 'alvaro@takeabreak.cl', "WhatsApp FALLO - pedido #$order_id", $error_msg );
}
```

`echo`/`print_r` dentro de un hook de WordPress son invisibles en producción.
Siempre `error_log` + `wp_mail` de alerta.

## Diagnóstico rápido de canal caído

```bash
# ¿El token responde?
curl "https://graph.facebook.com/v21.0/me?access_token=$TOKEN"

# ¿La app está activa?
curl "https://graph.facebook.com/v21.0/debug_token?input_token=$TOKEN&access_token=$TOKEN" \
  | jq '.data.is_valid, .data.app_id'

# ¿El número está conectado?
curl "https://graph.facebook.com/v21.0/$PHONE_NUMBER_ID?fields=verified_name,status&access_token=$TOKEN" \
  | jq '.status'  # debe ser "CONNECTED"
```

## Caso real (TAB, 2026-06-19)

Canal de WhatsApp de TAB caído ~semanas en silencio: app de Meta desactivada
por tercero. Fix: nueva app propia + System User Token permanente + token
movido a `wp-config.php` + alerta de fallo agregada.
Impacto: clientes de Plan Mensual sin confirmación WhatsApp de pedido ni cupón.
