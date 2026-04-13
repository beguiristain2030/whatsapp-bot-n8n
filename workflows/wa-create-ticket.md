# WA - Create Ticket

**Rol:** crear un ticket en el sistema principal a partir de la conversación ya lista.

## Objetivo

- POST idempotente con `Idempotency-Key = correlation_id`.
- Devolver el `ticket_id` al Orchestrator para que se comunique al usuario.
- Garantizar que nunca se cree un ticket duplicado por reentrega de webhook.

## Inputs

- `conversation` (estado `ready_to_create_ticket`).
- `correlation_id`.
- `tenant_context`.

## Outputs

- `{ ticket_id, status, duplicate }`.

## Nodos (n8n)

1. **Function — Build payload** según [../contracts/ticket-create.json](../contracts/ticket-create.json).
   - Inferir `type` desde `current_intent`:
     - `cargar_saldo` → `deposit`
     - `retirar_saldo` → `withdrawal`
     - `crear_usuario` → `user_creation`
     - `soporte` → `support`
   - `payload` = `collected_data`.
   - `risk` = `conversation.metadata.risk` si existe.
2. **HTTP Request — POST /v1/tickets**
   - Headers: `Idempotency-Key: {{correlation_id}}`, `X-Tenant-Id`, `Authorization`.
   - Retry: 5 con backoff (500ms, 2s, 8s, 30s, 2min).
3. **IF — error tras retries?**
   - Sí → **Execute Workflow — WA - Error Handler** con `error_class=integration`, y propagar error al Orchestrator para que dispare handoff.
4. **Return** `{ ticket_id, status, duplicate }`.

## Decisiones

- **Idempotency-Key obligatorio**: sin esto, la reentrega de webhooks crearía múltiples tickets. El backend debe honrarla (responsabilidad del backend).
- **Retries largos**: un ticket de pago/retiro es crítico; vale la pena aguantar el backoff antes de escalar.
- **Errores se escalan, no se ocultan**: si se agotan retries, no se responde "listo" al usuario.

## Errores posibles

| HTTP | manejo |
|---|---|
| 201 | éxito |
| 409 con `duplicate=true` | éxito idempotente, usar `ticket_id` devuelto |
| 422 | log error, marcar conversación, mensaje al usuario "datos inválidos" y pedir corrección (o handoff) |
| 5xx | retry con backoff; si agotan → dead-letter + handoff |
| timeout | retry |
