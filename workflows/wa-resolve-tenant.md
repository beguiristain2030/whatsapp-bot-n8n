# WA - Resolve Tenant

**Rol:** resolver `tenant_id` y cargar `features` del tenant a partir de `instance_id` y/o `to.phone`.

## Objetivo

- Completar `tenant_id` en el mensaje normalizado.
- Cargar `features` y `credentials_ref` del tenant al contexto del workflow.
- Rechazar mensajes de instancias no mapeadas (no responder al usuario — sólo log).

## Inputs

- `inbound-normalized.json` con `tenant_id = null`.

## Outputs

- `inbound-normalized.json` con `tenant_id` poblado.
- `tenant_context`: `{ features, credentials_ref, provider }`.

## Nodos (n8n)

1. **Function — Read cache** — lee `tenant_cache` de Static Data del workflow; si el record existe y no expiró, usarlo.
2. **IF — cache hit?**
   - Sí → return con tenant_context.
   - No → continuar.
3. **HTTP Request — GET /v1/tenants/mapping?instance_id=...&destination=...**
   - Auth: service JWT.
   - Retry: 3 con backoff.
4. **IF — 404?** → ir a `no_match`.
5. **Function — Update cache** — guardar resultado con `expires_at = now + 300s`.
6. **Set — tenant_id & tenant_context** en `$json`.
7. **Execute Workflow — WA - Load Conversation**.

### Rama `no_match`

1. **Function — Log security event** — tipo `tenant_unresolved`.
2. **Execute Workflow — WA - Error Handler** con `error_class = validation`.
3. **Stop** (no responder al usuario).

## Decisiones

- **Cache local con TTL corto**: la latencia importa; la inconsistencia temporal de 5 min es aceptable.
- **En Fase 1, el mapping puede venir del archivo [../mappings/tenant-resolution.json](../mappings/tenant-resolution.json)**. En Fase 2, del backend.
- **Nunca responder al usuario si el tenant no se resuelve**: impide enumeration y abuso.

## Errores posibles

| error | manejo |
|---|---|
| Backend caído | Retry; si persiste → circuit breaker → rechazar mensaje |
| Mapping ambiguo (mismo número, 2 tenants) | Log crit, rechazar, alerta |
| Cache corrupta | Invalidar y volver a consultar |
