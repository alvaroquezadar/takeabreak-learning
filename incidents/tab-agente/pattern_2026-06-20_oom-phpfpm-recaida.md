---
name: oom-phpfpm-recaida
type: feedback
description: Segundo OOM en 3 días (recaída Jun 17 → Jun 20). Causa confirmada: PHP-FPM perfil Bitnami "large" (max_children=50) con ~300 MB por child de WooCommerce en VM de 8 GB sin swap. Fix de fondo = resize VM 8→16 GB. Lección de diagnóstico: el serial console puede mostrar un buffer anterior al evento — verificar con dmesg.
---

## Patrón de OOM en WooCommerce + Bitnami

| Componente | Consumo típico |
|-----------|---------------|
| php-fpm child × WooCommerce | ~300 MB × max_children |
| clamd (ClamAV residente) | ~1 GB |
| MariaDB buffer pool | 2 GB (`my-large.conf`) |
| **Total peor caso (vm 8 GB, 50 children)** | **~18 GB > RAM** |

El OOM killer bota mysqld → sitio colgado. TCP acepta conexiones pero HTTP no
responde. Load average se dispara (204 en este caso) por thrashing sin swap.

## Diagnóstico correcto

```bash
# En el servidor, tras reconectar SSH
dmesg | grep -i "oom\|killed"
# Muestra: "Killed process 465 (mysqld)" con tabla de uso de memoria
```

**Trapa**: el serial console de GCP puede mostrar un buffer que corta ANTES
del OOM. "El serial muestra el SO sano" ≠ "no hubo OOM".
Verificar siempre con `dmesg` una vez que SSH conecte.

## Fix de fondo aplicado (Jun 20)

```
# /opt/bitnami/php/etc/php-fpm.d/memory-large.conf
pm.max_children = 18    # era 50
pm.start_servers = 5
pm.min_spare_servers = 5
pm.max_spare_servers = 12
pm.max_requests = 5000

# /opt/bitnami/mariadb/conf/my-large.conf
innodb_buffer_pool_size = 1536M  # era 2024M
```

Y swap de 2 GB (`/swapfile` + `fstab` + `vm.swappiness=10`).

**Pero el fix real fue el resize VM**: `n2d-custom-4-8192` → `n2d-standard-4`
(4 vCPU / 16 GB). Costo extra verificado: +~$18/mes (descuento por uso sostenido).

## Regla

Swap = red de seguridad, no solución. Si el peor caso de memoria supera la RAM
disponible de forma consistente, swap solo retrasa el OOM. Dimensionar la VM
para el workload real.

## Precaución con Bitnami

Los perfiles (`memory-large`, `my-large`) tienen el comentario:
> "modified on server size changes"

Un resize por consola de GCP puede pisar los caps configurados manualmente.
Con 16 GB hay margen de sobra, pero revisar los perfiles tras cualquier resize
futuro.

## Plan de mejora pendiente (caídas futuras)

Ordenado por riesgo creciente:
1. Redis (object cache) — descarga DB, bajo riesgo
2. Page cache (WP Super Cache) — descarga PHP-FPM, probar en QA primero
3. Cloudflare CDN — DNS lo maneja Samuel (cambiar MX + DKIM/SPF/DMARC antes)
