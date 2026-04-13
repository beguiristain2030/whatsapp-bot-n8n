# F. Lógica de Intención

## 1. Catálogo de intenciones (MVP)

| intent | descripción | campos requeridos | acción final |
|---|---|---|---|
| `crear_usuario` | Alta de nuevo usuario en sistema | `full_name`, `document_id` | POST `/v1/users` |
| `cargar_saldo` | Usuario quiere acreditar saldo | `amount`, `receipt_image_url` | POST `/v1/tickets` tipo=deposit |
| `retirar_saldo` | Usuario quiere retirar saldo | `amount`, `destination_account` | POST `/v1/tickets` tipo=withdrawal |
| `consulta` | Pregunta genérica (saldo, estado, ayuda) | — | respuesta directa / lookup |
| `soporte` | Problema reportado | `issue_description` | POST `/v1/tickets` tipo=support |
| `handoff` | Pide hablar con humano | — | Escalate Human |
| `unknown` | No clasificable | — | fallback |

Este catálogo es **la fuente de verdad** del bot. Cualquier intención nueva se agrega primero aquí, luego en [../mappings/intent-keywords.json](../mappings/intent-keywords.json), luego en el Orchestrator.

## 2. Clasificación por reglas (MVP)

Ver [../mappings/intent-keywords.json](../mappings/intent-keywords.json).

El sub-workflow `WA - Detect Intent Rules`:

1. Normaliza el texto: lowercase, sin acentos, collapse whitespace.
2. Matchea contra keywords/regex por intent, en orden de **prioridad descendente**:
   - `handoff` (máxima prioridad — el usuario siempre puede escalar)
   - `retirar_saldo` (acción con dinero, match estricto)
   - `cargar_saldo`
   - `crear_usuario`
   - `soporte`
   - `consulta`
3. Si hay match múltiple, gana el de mayor prioridad.
4. Si no hay match → `unknown`.

**Importante:** las acciones con dinero requieren keywords **explícitas** (por ejemplo "retirar", "retiro", "sacar plata"). Nunca se infieren por sinónimos vagos.

## 3. Extracción de entidades

Por regex simples (MVP):

| entidad | regex | intents que la usan |
|---|---|---|
| `amount` | `(?:\$\s*)?(\d{1,3}(?:[.,]\d{3})*|\d+)` (normalizada a entero) | cargar_saldo, retirar_saldo |
| `document_id` | `\b\d{7,9}\b` | crear_usuario |
| `full_name` | inferido por turno dedicado, no regex | crear_usuario |
| `destination_account` | `CBU:\s*\d{22}` o `alias:\s*[\w.-]{6,20}` | retirar_saldo |

La extracción se ejecuta **después** de la clasificación y sólo para la intención detectada. Los valores extraídos se mergean con `collected_data`.

## 4. Fallback strategy

Cuando `intent = unknown`:

```
turn 1 → "No te entendí del todo. ¿Querés cargar saldo, retirar, o hablar con un operador?"
turn 2 → mostrar menú numerado: "1) Cargar  2) Retirar  3) Consulta  4) Operador"
turn 3 → si sigue unknown → handoff automático
```

El contador de fallbacks vive en `collected_data._fallback_count`. Se resetea cuando hay una intent válida.

## 5. Cuándo escalar a humano (reglas)

| Condición | Acción |
|---|---|
| Usuario pide humano explícitamente | handoff inmediato |
| `_fallback_count >= 3` | handoff automático |
| `turn_count > 10` sin avanzar en `collected_data` | handoff automático (anti-loop) |
| Intent = `retirar_saldo` con `amount > threshold_tenant` | handoff obligatorio (alto riesgo) |
| Risk score (§ ver [media-risk.md](media-risk.md)) ≥ `high` | handoff obligatorio |
| Error en Create Ticket después de retries | handoff como fallback |
| Media duplicada detectada | handoff (posible fraude) |
| `mode` ya es `human` | no escalar, sólo registrar |

## 6. Integración futura de IA (sin romper el sistema)

El contrato de entrada/salida del sub-workflow de intención es fijo:

**Input:**
```json
{ "tenant_id": "...", "text": "...", "context": { "state": "...", "current_intent": "..." } }
```

**Output:**
```json
{
  "intent": "cargar_saldo",
  "confidence": 0.92,
  "entities": { "amount": 5000 },
  "reasoning": "...",
  "requires_human": false
}
```

Para migrar a IA:

1. Crear `WA - Detect Intent AI` que cumpla el mismo contrato (llamada a LLM con prompt determinista).
2. Activarlo por feature flag por tenant: `features.intent_engine = "ai" | "rules" | "hybrid"`.
3. En modo `hybrid`: reglas primero; si `unknown`, caer a AI. Esto da un piso determinista y un techo inteligente.
4. Para acciones con dinero, **siempre** exigir validación de reglas incluso si AI detectó la intent. La IA no decide retiros sola.

**Riesgo cubierto:** hallucinations del LLM no pueden ejecutar acciones financieras por sí solas.

## 7. Ejemplos de clasificación

| texto usuario | intent | entities | nota |
|---|---|---|---|
| "hola, quiero cargar 5000" | `cargar_saldo` | `{amount: 5000}` | feliz |
| "saquen 2000" | `retirar_saldo` | `{amount: 2000}` | keyword fuerte |
| "necesito un humano" | `handoff` | — | prioridad máxima |
| "no sé" | `unknown` | — | fallback |
| "quiero retirar pero primero cargo 1000" | `cargar_saldo` | `{amount: 1000}` | primera acción detectada gana salvo prioridad mayor |
| "mi nombre es Juan Pérez, DNI 12345678" | depende de `current_intent` | `{full_name, document_id}` | entidades entran en contexto activo |
