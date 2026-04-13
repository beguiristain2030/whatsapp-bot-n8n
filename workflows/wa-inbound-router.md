# WA - Inbound Router

**Rol:** único webhook público. Recibe del proveedor WhatsApp, valida, responde 200 rápido y dispara el pipeline.

## Objetivo

- Aceptar requests del proveedor con latencia < 1s.
- Validar firma HMAC.
- Generar `correlation_id`.
- Disparar `WA - Normalize Incoming` en modo "Execute Workflow" (async-like) y devolver 200.

## Inputs

- HTTP POST del proveedor WA (Meta Cloud / Evolution / 360dialog).
- Headers: `X-Hub-Signature-256` (o equivalente).

## Outputs

- HTTP 200 inmediato.
- Disparo de `WA - Normalize Incoming` con `{ raw_payload, correlation_id, received_at, provider }`.

## Nodos (n8n)

1. **Webhook** (POST, path `/wa-inbound/:provider`)
   - Response Mode: `responseNode` para poder responder antes de terminar.
2. **Function — Generate correlation_id**
   - `uuidv4()` o ULID, set en `$json.correlation_id`.
3. **Function — Verify HMAC**
   - Lee header firma, calcula HMAC del body con secret del proveedor (variable env `WA_WEBHOOK_SECRET_<provider>`).
   - Si no coincide → error 401 y salir.
4. **Respond to Webhook** — `{"status":"ok"}` 200.
5. **Execute Workflow** — `WA - Normalize Incoming`, payload:
   ```json
   { "raw": "={{$json.body}}", "provider": "={{$params.provider}}", "correlation_id": "={{$json.correlation_id}}", "received_at": "={{$now.toISO()}}" }
   ```

## Decisiones

- **200 antes de procesar**: el proveedor WhatsApp reintenta agresivamente si tarda >5s. Responder rápido y delegar al pipeline.
- **Un webhook por provider**: path parametrizado permite soportar varios proveedores sin multiplicar workflows.
- **HMAC obligatorio**: sin verificación, el webhook es un vector de spoofing trivial.

## Errores posibles

| error | manejo |
|---|---|
| Firma inválida | 401, log de seguridad, no procesar |
| Body no parseable | 400, log error |
| Execute Workflow falla | El webhook ya respondió 200; el error lo captura `WA - Error Handler` |
| Rate limit por IP excedido | 429 si hay gateway; en MVP basta con log |

## Pseudocódigo de verificación HMAC

```js
const crypto = require('crypto');
const signature = items[0].headers['x-hub-signature-256']; // "sha256=abcd..."
const body = items[0].body_raw;
const expected = 'sha256=' + crypto
  .createHmac('sha256', $env.WA_WEBHOOK_SECRET_META)
  .update(body)
  .digest('hex');
if (signature !== expected) throw new Error('invalid_signature');
return items;
```
