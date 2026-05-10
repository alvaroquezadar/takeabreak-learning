---
name: markdown-idempotent-generator
description: Use when writing a Python script that regenerates sections of a Markdown document on repeated runs. A placeholder-based approach only works on the first run; subsequent runs crash because the placeholder was consumed on the first write.
---

## Patrón

Un generador de Markdown que busca un placeholder para reemplazar el contenido solo funciona en la primera corrida. En la segunda, el placeholder ya no existe y el script crashea.

```python
# PROBLEMA: solo funciona una vez
placeholder = "<!-- contenido generado aquí -->"
if placeholder not in actual:
    raise RuntimeError("Placeholder no encontrado")
cabecera = actual.split(placeholder)[0]
nuevo_contenido = cabecera + secciones_generadas
```

En la primera corrida: funciona.
En la segunda: `placeholder not in actual` → RuntimeError.

Esto se produce en cualquier script que:
- Lee un documento Markdown existente
- Busca un marcador para saber dónde insertar el contenido nuevo
- El marcador es el mismo texto que se reemplaza (se consume a sí mismo)

## Fix: marcador estable separado del contenido

Usar un marcador que **no sea parte del contenido generado** — por ejemplo, el inicio de la primera sección generada:

```python
import re

actual = DOC_PATH.read_text(encoding="utf-8")

# Marcador estable: inicio de la primera sección generada
match = re.search(r"^## 1\. ", actual, re.MULTILINE)
if match:
    # Segunda corrida y siguientes: reemplazar desde la sección 1 en adelante
    cabecera = actual[:match.start()].rstrip() + "\n\n"
else:
    # Primera corrida (bootstrap): usar placeholder original
    placeholder = "<!-- contenido generado aquí -->"
    if placeholder not in actual:
        raise RuntimeError(
            f"Ni marcador de sección ni placeholder encontrados en {DOC_PATH}. "
            f"¿El documento fue inicializado correctamente?"
        )
    cabecera = actual.split(placeholder)[0]

nuevo_doc = cabecera + secciones_generadas
DOC_PATH.write_text(nuevo_doc, encoding="utf-8")
```

## Variante: marcador HTML explícito

Si el documento no tiene una sección `## 1.` natural, agregar un marcador HTML propio que **se preserva en el output** (no se consume):

```python
MARKER = "<!-- BEGIN_GENERATED -->"

actual = DOC_PATH.read_text(encoding="utf-8")
if MARKER in actual:
    cabecera = actual.split(MARKER)[0] + MARKER + "\n"
else:
    cabecera = actual + "\n" + MARKER + "\n"

nuevo_doc = cabecera + secciones_generadas
DOC_PATH.write_text(nuevo_doc, encoding="utf-8")
```

Aquí el marcador está en la cabecera estática del documento y **también está en el output** — por eso `MARKER in actual` siempre es True después de la primera corrida.

## También: actualizar campos en la cabecera (fecha de generación)

Si el documento tiene una línea como `**Fecha de generación:** 2026-05-08`, actualizarla en la misma corrida con regex:

```python
cabecera = re.sub(
    r"\*\*Fecha de generación:\*\*.*",
    f"**Fecha de generación:** {date.today().isoformat()}",
    cabecera,
)
```

Sin este paso, la fecha en el documento queda congelada en la de la primera corrida aunque el contenido se regenere.

## Por qué importa

El error no es visible en tests unitarios del generador (que corren desde cero). Solo aparece en el segundo run de producción. En un workflow programado que corre semanalmente, el fallo pasa desapercibido hasta la segunda semana.

Documentado en tab-agente (commit fe5785e, mayo 2026), donde el orquestador de Fase 1 fallaba en todas las corridas excepto la primera.
