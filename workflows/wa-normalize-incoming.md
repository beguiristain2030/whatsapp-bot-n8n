# WA - Normalize Incoming

**Rol:** convierte el payload crudo del proveedor al contrato [../contracts/inbound-normalized.json](../contracts/inbound-normalized.json).

## Objetivo

- Un único shape estándar para todo lo que sigue, sin importar el proveedor.
- Identificar tipo de mensaje (text/image/audio/document/...).
- Hacer un short-circuit de idempotencia lo antes posible.

## Inputs

```json
{ "raw": {...}, "provider": "meta_cloud|evolution_api|360dialog", "correlation_id": "...", "received_at": "..." }
```

## Outputs

- Objeto conforme a `inbound-normalized.json`.
- O `{ "status": "duplicate", "ignored": true }` si el `provider_message_id` ya existe.

## Nodos (n8n)

1. **Switch — provider** — 1 rama por proveedor soportado.
2. **Function — Map {provider} to normalized** — una función por rama. Extrae:
   - `provider_message_id`
   - `from.phone` (E.164)
   - `to.phone`
   - `instance_id`
   - `type`
   - `timestamp`
   - `content.text` / `content.media.*`
3. **Merge** — unifica todas las ramas.
4. **HTTP Request — GET /v1/messages?provider_message_id=...** — check de idempotencia (sin `tenant_id` todavía; el backend acepta lookup global para este caso).
5. **IF — duplicate?** — si sí, return early con `{ status: "duplicate" }`.
6. **Execute Workflow — WA - Resolve Tenant** — pasa el normalizado sin `tenant_id`.

## Decisiones

- **Un mapper por proveedor**: aislar la superficie de cambios. Si Meta actualiza el shape, sólo se toca su mapper.
- **Normalizar phone a E.164 en este paso**: evita mismatches aguas abajo.
- **Idempotencia en este paso, no en Router**: Router debe ser lo más liviano posible.

## Errores posibles

| error | manejo |
|---|---|
| Provider no soportado | throw → Error Handler |
| Payload sin `provider_message_id` | log warn, asignar uuid temporal y marcar `metadata.synthetic_id = true` |
| `type = unsupported` | normalizar igual, pero el orchestrator contesta "aún no soportamos ese tipo de mensaje" |

## Ejemplo de mapper Evolution API (pseudocódigo)

```js
const raw = $input.first().json.raw;
const msg = raw.data.message;
return {
  version: "1.0",
  correlation_id: $input.first().json.correlation_id,
  tenant_id: null,
  conversation_hint: normalizeE164(raw.data.key.remoteJid),
  provider: "evolution_api",
  instance_id: raw.instance,
  provider_message_id: raw.data.key.id,
  from: { phone: normalizeE164(raw.data.key.remoteJid), display_name: raw.data.pushName ?? null },
  to: { phone: raw.data.key.fromMe ? raw.data.key.remoteJid : raw.owner },
  type: detectType(msg),
  timestamp: new Date(raw.data.messageTimestamp * 1000).toISOString(),
  content: buildContent(msg),
  metadata: { received_at: $input.first().json.received_at }
};
```
