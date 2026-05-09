---
name: github-actions-state
description: Use when building a scheduled (cron) GitHub Actions workflow that necesita recordar estado entre runs — monitores de uptime, detectores de transición, notificadores que no deben spamear.
---

## Patrón

Los workflows de GH Actions con cron necesitan a veces recordar el estado del run anterior: ¿el sitio estaba UP o DOWN? ¿Ya se envió la alerta? ¿Cuál fue el último valor medido?

La solución instintiva — GH Actions cache — es frágil. El cache puede ser eviccionado, no sobrevive cambios en el workflow, y no hay garantía de que exista en el primer run. Resultado: el workflow asume estado inicial y puede enviar falsos positivos.

### Solución: commitear estado al repo

Solo en transiciones (UP→DOWN, DOWN→UP, etc.), escribir un archivo al repo y commitearlo. El ruido en `git log` es mínimo: 0-2 commits/mes para un monitor normal.

### Implementación (tab-agente, monitor_tab.py, mayo 2026)

```python
# Leer estado previo del archivo commiteado
STATE_FILE = "monitor_state.txt"

def read_state() -> str:
    try:
        with open(STATE_FILE) as f:
            return f.read().strip()  # "UP", "SLOW", "DOWN"
    except FileNotFoundError:
        return "UP"  # asumir UP si no hay estado (primer run)

def write_state(state: str):
    with open(STATE_FILE, "w") as f:
        f.write(state)
```

```yaml
# .github/workflows/monitor.yml
jobs:
  check:
    permissions:
      contents: write  # necesario para commitear el state file

    steps:
      - uses: actions/checkout@v4
        with:
          token: ${{ secrets.GITHUB_TOKEN }}

      - name: Run monitor
        run: python agente_monitor_tab.py  # escribe monitor_state.txt si hay transición

      - name: Commit state if changed
        run: |
          git config user.email "monitor@noreply"
          git config user.name "Monitor Bot"
          if git diff --quiet monitor_state.txt; then
            echo "Sin cambio de estado — no se commitea."
          else
            # pull-rebase por si otro workflow commiteó mientras tanto
            git pull --rebase origin main || true
            git add monitor_state.txt
            git commit -m "monitor: estado → $(cat monitor_state.txt)"
            git push origin main || (sleep 5 && git pull --rebase origin main && git push origin main)
          fi
```

### Por qué solo en transiciones

Si el monitor escribe el archivo en cada run, hay 288 commits/día (cada 5 min). Solo escribir cuando `nuevo_estado != estado_previo` reduce el ruido a ~0-2 commits/mes (fallas reales de producción).

```python
prev_state = read_state()
new_state = check_endpoints()  # "UP", "SLOW", "DOWN"

if new_state != prev_state:
    send_alert_email(prev_state, new_state)
    write_state(new_state)
    # el workflow commitea el archivo solo en este caso
```

### Estados recomendados para monitores de latencia

```python
LATENCY_WARN_S     = 8   # responde pero degradado → SLOW
LATENCY_CRITICAL_S = 15  # aunque responda 200 → DOWN

# DOWN domina sobre SLOW; SLOW domina sobre UP
def classify(latency_s, status_ok):
    if not status_ok or latency_s > LATENCY_CRITICAL_S:
        return "DOWN"
    if latency_s > LATENCY_WARN_S:
        return "SLOW"
    return "UP"
```

## Por qué importa

El patrón cache-para-estado es común y falla en producción de formas no obvias. El commit-para-estado es auditable (se puede revisar la historia de transiciones), sobrevive cambios en el workflow, y funciona en el primer run sin lógica especial.

Implementado en tab-agente commit b62f6ec (8-may-2026).
