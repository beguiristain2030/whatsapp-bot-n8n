# WA - Detect Intent Rules

**Rol:** clasificación de intención por reglas. Cumple el contrato intercambiable con un eventual `WA - Detect Intent AI`.

## Objetivo

- Devolver `{ intent, confidence, entities, requires_human, reasoning }`.
- No hacer side-effects; sólo clasificar.

## Inputs

```json
{ "tenant_id": "...", "text": "...", "context": { "state": "...", "current_intent": "..." } }
```

## Outputs

```json
{ "intent": "cargar_saldo", "confidence": 1.0, "entities": { "amount": 5000 }, "requires_human": false, "reasoning": "keyword_match:cargar" }
```

## Nodos (n8n)

1. **Function — Normalize text** — lowercase, strip accents, collapse whitespace.
2. **Function — Load rules** — leer [../mappings/intent-keywords.json](../mappings/intent-keywords.json) (en n8n Static Data o fetch del backend).
3. **Function — Match by priority**:
   ```js
   const normalized = normalize(items[0].json.text);
   const rules = loadRules().intents.sort((a,b) => b.priority - a.priority);
   for (const r of rules) {
     if (r.keywords.some(k => normalized.includes(k))) return match(r);
     if (r.regex.some(rx => new RegExp(rx).test(normalized))) return match(r);
   }
   return { intent: 'unknown', confidence: 0 };
   ```
4. **Function — Extract entities** — sólo para el intent detectado. Aplicar regex de entidades definidas en [../docs/intent-logic.md § 3](../docs/intent-logic.md).
5. **Set — requires_human** — `true` si `intent === 'handoff'` o regla específica del tenant.
6. **Return**.

## Decisiones

- **Sin side-effects**: este workflow no persiste nada. Eso lo hace el Orchestrator.
- **Priority first**: la primera regla que matchea gana. No hay scoring blando en MVP; es determinista.
- **Entidades sólo para el intent detectado**: reduce ruido y falsos positivos.

## Errores posibles

| error | manejo |
|---|---|
| Texto vacío (mensaje de tipo image sin caption) | intent = `unknown` con razón `no_text`; el orchestrator decide según `context.current_intent` |
| Regex mal formado | log error, skip regla, continuar |
| JSON de reglas corrupto | alerta crítica, fallback a set vacío → todo cae en `unknown` |
