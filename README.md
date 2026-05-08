# takeabreak-learning

Repositorio central de aprendizaje de Take a Break.

## Estructura

```
incidents/
  tab-agente/      ← incidents del repo tab-agente
  berlin/          ← incidents del repo la berlín
proposed-skills/   ← skills candidatos (revisar y mover a ~/.claude/skills/ si se aprueba)
```

## Flujo

1. Cada proyecto guarda incidents en su propia carpeta `memory/incidents/`
2. Los incidents se commitean al proyecto Y se copian acá
3. El agente semanal lee desde acá, detecta patrones, propone skills
4. Skills aprobados se mueven manualmente a `~/.claude/skills/` (global)

## Skills aprobados

Los skills en `proposed-skills/` aún no están activos. Para activar uno:
```bash
cp -r proposed-skills/NOMBRE ~/.claude/skills/
```
