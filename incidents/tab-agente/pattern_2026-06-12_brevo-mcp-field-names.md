---
name: brevo-mcp-field-names
type: feedback
description: El MCP de Brevo (@houtini/brevo-mcp) usaba "opens"/"uniqueOpens" que no existen en la API real — Brevo llama aperturas "views"/"uniqueViews". El open rate salía siempre 0. Fix aplicado vendorizando el paquete y parcheando los nombres de campo.
---

## Contexto

Campaña de reactivación (Brevo id 15): 28.492 envíos, 380 clicks, **0 aperturas**.
Imposible: clicks implica apertura. El bug estaba en el conector, no en Brevo.

## Causa

El paquete `@houtini/brevo-mcp` leía `stats.uniqueOpens` y `stats.opens` —
nombres que no existen en la respuesta de Brevo. La API real usa:

| Concepto | Campo real en Brevo API |
|----------|------------------------|
| Aperturas únicas | `uniqueViews` |
| Aperturas totales | `viewed` |
| Clicks únicos | `uniqueClicks` ← este sí cuadraba |

Bug secundario: el conector leía solo `campaignStats[0]` — primera lista. En
campañas enviadas a múltiples listas (ej. Cyber Week) subcontaba todo.

## Fix aplicado

Vendorizado a `~/tab-agente/vendor/brevo-mcp/`:
- `aggregateStats()` suma todas las listas de `campaignStats[]`
- Lee `uniqueViews ?? uniqueOpens` (alias de compatibilidad)
- Config MCP apunta a `node vendor/brevo-mcp/.../dist/index.js` en vez de `npx`

## Regla

**Nunca parchar la caché de npx** — es transitoria. Un `npx -y` puede refrescarla
y el bug vuelve en silencio. Vendorizar cuando el paquete importa para métricas
de negocio.

## Verificación rápida si open rate vuelve a 0

```bash
# Testear contra la API real de Brevo
curl -s "https://api.brevo.com/v3/emailCampaigns/15/sendingStatistics" \
  -H "api-key: $BREVO_API_KEY" | jq '.campaignStats[].uniqueViews'
```

Si devuelve números: el campo existe y el conector tiene un bug de nombres.
