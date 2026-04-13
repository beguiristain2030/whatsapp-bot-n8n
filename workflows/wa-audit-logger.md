# WA - Audit Logger

**Rol:** centralizar el registro de decisiones del bot en el sistema principal.

## Objetivo

- Un único punto de escritura para auditoría.
- Formato estructurado y consistente.
- Nunca bloquear el flujo principal: este workflow es best-effort.

## Inputs

```json
{
  "tenant_id": "...",
  "correlation_id": "...",
  "conversation_id": "...",
  "workflow": "WA - Conversation Orchestrator",
  "step": "intent_detected",
  "decision": "ask_next_field|create_ticket|handoff|answer|fallback|bot_silent_human_mode|error",
  "data": {...},
  "duration_ms": 42
}
```

## Outputs

- Best-effort: 200/201 del backend, o log interno si falla.

## Nodos (n8n)

1. **Function — Enrich** — agregar `ts`, hostname, workflow version.
2. **HTTP Request — POST /v1/audit** — timeout 2s, 1 retry.
3. **IF — failed?**
   - Sí → **Set — warning log only**, no propagar error. NUNCA fallar el workflow llamante por auditoría.

## Decisiones

- **Best-effort**: la auditoría no puede tumbar al bot. Si el backend de audit está caído, se loguea local y se sigue.
- **Batch opcional**: en Fase 3 se puede hacer batching si el volumen lo amerita; en MVP, uno por uno.

## Errores posibles

| error | manejo |
|---|---|
| Backend caído | warning local, continuar |
| Payload muy grande | truncar campos `data.*` a N KB |
