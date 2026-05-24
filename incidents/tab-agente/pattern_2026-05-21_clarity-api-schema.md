---
name: Clarity API — schema real incompatible con documentación (CamelCase + campos genéricos)
type: incident
description: El cliente de Clarity producía filas vacías porque los nombres de campo en la respuesta real usan CamelCase con un esquema genérico (subTotal, sessionsCount) distinto al que describe la documentación con espacios y nombres propios.
project: tab-agente
date: 2026-05-18
commits:
  - 7c485d8
---

## Qué pasó

`scripts/analisis/fuentes/clarity_client.py` fue implementado con los nombres de métricas de la documentación oficial de Microsoft Clarity ("Scroll Depth", "Engagement Time", "Dead Click Count"). La API real devuelve:

| Métrica (docs)    | Campo real en response     |
|-------------------|---------------------------|
| Dead Click Count  | `subTotal` (en obj genérico) |
| Rage Click Count  | `subTotal`                 |
| Scroll Depth      | `averageScrollDepth`       |
| Engagement Time   | `activeTime`               |
| Users (distinct)  | `distinctUserCount` (typo inicial: `distantUserCount`) |

El parser mapeaba los nombres de docs → campos que no existían en la respuesta → todas las filas del output eran vacías o cero. Ningún test fallaba porque los tests habían sido escritos con el schema de docs también.

## Raíz del problema

La documentación de Clarity describe métricas con nombres en prosa ("Dead Click Count") pero la API devuelve JSON con nombres CamelCase en un schema genérico. No se validó la respuesta real antes de escribir el parser.

## Fix aplicado

- Agregar `save_raw=True` al cliente para guardar la respuesta cruda en un archivo temporal con timestamp.
- Inspeccionar la respuesta real y reconstruir `_parse_metrics()` con el mapeo validado.
- Escribir 16 tests unitarios usando fixtures del schema real (no de la documentación).
- Descubrimiento: `distinctUserCount` estaba mal escrito como `distantUserCount` en el código original.

## Regla derivada

Para cualquier API externa: escribir el parser después de ver la respuesta real, no la documentación.
Agregar `save_raw=True` (o equivalente) como opción de debug antes de mergear el primer código de integración.

Ver skill propuesto: `EXTERNAL-API-SCHEMA-FIRST`
