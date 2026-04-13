# WA - Escalate Human

**Rol:** marcar la conversación como `mode=human`, notificar al sistema principal y enviar un mensaje al usuario confirmando la derivación.

## Objetivo

- Corte limpio: después de este workflow, el bot no vuelve a responder hasta que un operador cierre o el sistema revierta `mode=bot`.
- Dejar al operador un snapshot suficiente para actuar.

## Inputs

```json
{ "tenant_id": "...", "correlation_id": "...", "conversation_id": "...", "reason": "...", "context_snapshot": {...} }
```

## Outputs

- `{ handoff_id, mode: "human" }`

## Nodos (n8n)

1. **Function — Build payload** según [../contracts/handoff-human.json](../contracts/handoff-human.json).
2. **HTTP Request — POST /v1/handoff**
   - Headers: `Idempotency-Key: {{correlation_id}}`.
   - Retry: 3.
3. **Function — Build confirmation message** según `reason`:
   - `user_requested` → "Listo, te derivo con alguien del equipo. En breve te contactan por acá mismo."
   - `high_amount` → "Para este monto necesito derivarte a un operador. Ya estoy avisando."
   - `high_risk` / `duplicate_media_cross_user` → "Tu solicitud está en revisión manual. Un operador te va a contactar."
   - `ticket_creation_failed` → "Estamos teniendo un problema técnico. Un operador te va a contactar."
   - `fallback_exhausted` / `turn_limit` → "No estoy pudiendo ayudarte por acá. Te paso con una persona."
4. **Execute Workflow — WA - Send Outbound** con el mensaje.
5. **Execute Workflow — WA - Audit Logger** con `decision=handoff`.

## Decisiones

- **Mensaje de confirmación siempre**: el usuario nunca debe quedar sin respuesta al escalar.
- **Idempotencia**: un mismo mensaje del usuario no debe escalar dos veces.
- **Reason enumerada**: facilita métricas y análisis de causas de handoff.

## Errores posibles

| error | manejo |
|---|---|
| POST /v1/handoff falla | retry; si agotan, log crit + alerta. El mensaje al usuario igual se envía explicando el problema |
| `conversation_id` no existe | 404 → log error, crear nueva conversación? En MVP, fallar y loggear |
