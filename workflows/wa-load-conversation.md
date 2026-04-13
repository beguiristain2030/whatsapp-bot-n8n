# WA - Load Conversation

**Rol:** cargar (o crear) la conversación activa para el `(tenant_id, phone)`.

## Objetivo

- Devolver el objeto conversación completo para que el Orchestrator opere sobre él.
- Aplicar regla de expiración: si la conversación activa expiró, cerrarla y crear una nueva.
- Detectar temprano si `mode == human` y, en ese caso, cortar el flujo.

## Inputs

- `inbound-normalized.json` con `tenant_id`.

## Outputs

- Objeto `conversation` conforme a [../contracts/conversation-update.json](../contracts/conversation-update.json).
- Flag `bot_must_be_silent` (bool) si `mode=human`.

## Nodos (n8n)

1. **HTTP Request — GET /v1/conversations?tenant_id=...&phone=...&active=true**
   - Retries: 3 backoff.
2. **IF — 404?**
   - Sí → **HTTP Request — POST /v1/conversations** con estado inicial `bot_active`.
   - No → continuar.
3. **Function — Check expiry** — si `expires_at < now`, llamar `POST /v1/conversations/{id}/close?reason=timeout` y luego POST nueva.
4. **Function — Persist message (POST /v1/messages)** con [../contracts/message-log.json](../contracts/message-log.json). Si 409 (duplicado) → return early con `{ status: "duplicate" }`.
5. **Set — bot_must_be_silent = (conversation.mode === 'human')**.
6. **IF — bot_must_be_silent?**
   - Sí → **Execute Workflow — WA - Audit Logger** con `decision=bot_silent_human_mode` y **Stop**.
   - No → **Execute Workflow — WA - Detect Intent Rules**.

## Decisiones

- **Persistir mensaje antes de procesar**: el registro en `messages` garantiza auditoría aunque el pipeline explote después.
- **`mode=human` es un corte duro**: se evalúa aquí y no más abajo, para no gastar ciclos en un mensaje que no va a responderse.
- **Creación lazy de conversación**: nunca pre-crear; siempre "get or create".

## Errores posibles

| error | manejo |
|---|---|
| Backend caído | Retry; circuit breaker → modo degradado (Send Outbound genérico "problema técnico") |
| 409 en persist message | return duplicate; no es error |
| Optimistic lock 409 en PUT | Re-GET y re-merge con reintento máx 2 |
