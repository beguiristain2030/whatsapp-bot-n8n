# WA - Error Handler

**Rol:** capturar errores irrecuperables, enviarlos al dead-letter, alertar, y asegurar que el usuario reciba una respuesta razonable.

## Objetivo

- Último recurso. Cuando otro workflow explota o agota retries, este lo recibe.
- Persistir el incidente.
- Decidir si escalar a humano o responder genérico.

## Inputs

```json
{
  "tenant_id": "...|null",
  "correlation_id": "...",
  "conversation_id": "...|null",
  "source_workflow": "...",
  "source_step": "...|null",
  "error_class": "...",
  "message": "...",
  "stack": "...|null",
  "retries_attempted": 5,
  "payload": {...}
}
```

## Outputs

- `{ dlq_id, handoff_triggered: bool }`.

## Nodos (n8n)

1. **Function — Classify severity** según `error_class`:
   - `http_5xx` / `dependency_down` / `timeout` → `severity=high`
   - `validation` / `http_4xx` → `severity=medium`
   - `unknown` / `logic` → `severity=high`
2. **HTTP Request — POST /v1/dead_letter** según [../contracts/error-event.json](../contracts/error-event.json).
3. **HTTP Request — POST webhook de alerta** (Slack/Discord) si `severity=high`. Plantilla:
   ```
   🚨 {workflow} | tenant={tenant_id}
   error: {error_class} {message}
   corr: {correlation_id}
   DLQ: {dlq_id}
   ```
4. **IF — afecta a un usuario en vivo (`conversation_id` presente)?**
   - Sí → **Execute Workflow — WA - Escalate Human** con `reason=ticket_creation_failed` (o la que corresponda). Esto cierra el loop con el usuario.
5. **Return**.

## Decisiones

- **Dead-letter en backend, no en n8n**: la DLQ debe sobrevivir reinicios de n8n y ser inspeccionable por humanos.
- **Alertas sólo para severity=high**: evita ruido en casos recuperables.
- **Siempre cerrar el loop con el usuario**: si había una conversación activa, se escala para que no quede sin respuesta.

## Errores posibles (dentro del Error Handler)

| error | manejo |
|---|---|
| Backend de DLQ caído | persist local a Static Data como fallback, retry en 5 min vía workflow scheduled |
| Webhook de alerta caído | log local, continuar |
