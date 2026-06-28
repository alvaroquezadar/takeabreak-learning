---
name: checkout-email-typo
description: Use when a WooCommerce checkout has recurring support cases caused by customers entering mistyped email domains (gmial.com, gmail.como, hotmal.com, etc.) that cause lost confirmation emails and coupons.
---

## Patrón

Clientes tipean su correo incorrectamente en checkout → el email de confirmación/cupón rebota (NDR 554, MX inexistente) → el cliente no lo recibe → contacto a soporte, pérdida de cupón o reembolso.

Este problema es especialmente grave cuando el email es el canal primario de entrega de un valor (cupón, acceso, código). Si hay un canal de respaldo (WhatsApp), la caída simultánea de ambos deja al cliente sin nada.

### Caso real (takeabreak.cl, 2026-06-20)

Clientes con `gmial.com` y `gmail.como` perdieron sus cupones de Plan Mensual porque:
1. El email rebotó (NDR 554)
2. El WhatsApp estaba caído por app Meta desactivada (incident separado)

## Implementación

JS no invasivo en `wp_footer` / `is_checkout` en `functions.php`:

```php
add_action('wp_footer', function() {
    if (!is_checkout()) return;
    ?>
    <script>
    (function() {
        var COMMON_DOMAINS = ['gmail.com','hotmail.com','yahoo.com','outlook.com',
                              'icloud.com','live.com','me.com'];
        var KNOWN_TYPOS   = {'gmial.com':'gmail.com','gmail.como':'gmail.com',
                              'hotmal.com':'hotmail.com','yaho.com':'yahoo.com'};

        function levenshtein(a, b) {
            var m = a.length, n = b.length, dp = [];
            for (var i = 0; i <= m; i++) { dp[i] = [i]; }
            for (var j = 0; j <= n; j++) { dp[0][j] = j; }
            for (var i = 1; i <= m; i++) {
                for (var j = 1; j <= n; j++) {
                    dp[i][j] = (a[i-1] === b[j-1])
                        ? dp[i-1][j-1]
                        : 1 + Math.min(dp[i-1][j], dp[i][j-1], dp[i-1][j-1]);
                }
            }
            return dp[m][n];
        }

        function suggestDomain(domain) {
            if (KNOWN_TYPOS[domain]) return KNOWN_TYPOS[domain];
            for (var i = 0; i < COMMON_DOMAINS.length; i++) {
                if (levenshtein(domain, COMMON_DOMAINS[i]) <= 1) return COMMON_DOMAINS[i];
            }
            return null;
        }

        document.addEventListener('DOMContentLoaded', function() {
            var emailField = document.getElementById('billing_email');
            if (!emailField) return;
            var hint = document.createElement('p');
            hint.style.cssText = 'color:#c0392b;font-size:.85em;margin-top:4px;display:none';
            emailField.parentNode.appendChild(hint);

            emailField.addEventListener('blur', function() {
                var val = emailField.value.trim();
                var atPos = val.lastIndexOf('@');
                if (atPos < 0) return;
                var domain = val.slice(atPos + 1).toLowerCase();
                var suggested = suggestDomain(domain);
                if (suggested && suggested !== domain) {
                    var correction = val.slice(0, atPos + 1) + suggested;
                    hint.innerHTML = '¿Quisiste decir <strong>' + correction
                        + '</strong>? <a href="#" style="color:#2980b9" class="tab-email-fix">Corregir</a>';
                    hint.style.display = 'block';
                    hint.querySelector('.tab-email-fix').addEventListener('click', function(e) {
                        e.preventDefault();
                        emailField.value = correction;
                        hint.style.display = 'none';
                    });
                } else {
                    hint.style.display = 'none';
                }
            });
        });
    })();
    </script>
    <?php
});
```

## Comportamiento

- No bloquea el envío (sugerencia, no validación hard)
- Se activa en `blur` del campo email del billing
- Detecta typos conocidos + Levenshtein ≤ 1 contra dominios comunes
- Muestra "¿Quisiste decir X?" con enlace para corregir con un clic

## QA mínimo (Playwright)

```
gmial.com   → sugiere + corrige a gmail.com      ✓
gmail.como  → sugiere gmail.com                  ✓
gmail.com   → no muestra sugerencia              ✓
```

## Dónde agregar

`functions.php` del tema activo (o plugin hijo de tema). Hacer backup antes: `functions.php.bak-emailtypo-YYYYMMDD`.

## Limitaciones

- Solo detecta typos de dominio, no de usuario (`alvro@` en vez de `alvaro@`)
- Levenshtein ≤ 1 puede tener falsos positivos con dominios poco comunes
- No reemplaza validación server-side (verificar MX antes de aceptar el pedido es más robusto pero más complejo)
