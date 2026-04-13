# H. Errores y Robustez

## 1. Idempotencia

**Clave de idempotencia:** `provider_message_id` (en WhatsApp: `wamid.HBgM...`).

### Implementación en n8n

En `WA - Normalize Incoming`, antes de procesar:

1. `GET /v1/messages?provider_message_id={id}&tenant_id={t}`
2. Si existe → return early (`{ status: "duplicate", ignored: true }`) + log.
3. Si no existe → `POST /v1/messages` (la DB tiene índice único sobre `(tenant_id, provider_message_id)`; ante colisión, el POST devuelve 409 y se trata como duplicado).

**Doble barrera:** n8n hace el check rápido; la DB hace el check definitivo con el índice único. Nunca confiar sólo en uno de los dos.

## 2. Retries y backoff

| Operación | Política |
|---|---|
| GET/PUT conversación | 3 intentos, backoff exponencial 200ms → 800ms → 3s |
| POST mensaje (dedupe) | 3 intentos, idempotente por clave |
| POST ticket | 5 intentos, backoff 500ms → 2s → 8s → 30s → 2min, con `Idempotency-Key = correlation_id` |
| Upload S3 | 3 intentos, backoff 500ms → 2s → 8s |
| POST /v1/media | 3 intentos, idempotente por `hash` |
| Send Outbound al proveedor WA | 5 intentos, backoff 500ms → 2s → 8s → 30s → 2min |
| Detect Intent (LLM, futuro) | 2 intentos, timeout 5s, fallback a reglas |

**Regla clave:** todas las operaciones `POST` no idempotentes llevan `Idempotency-Key`. El sistema principal **debe** respetarlas (responsabilidad del backend, no de n8n).

## 3. Dead-letter queue

Cuando se agotan los retries de un workflow crítico:

1. El error se enruta al sub-workflow `WA - Error Handler`.
2. El payload completo (input + error + stack) se persiste en:
   - `POST /v1/dead_letter` del sistema principal, con `type`, `correlation_id`, `payload`, `last_error`.
3. Se dispara alerta:
   - Webhook a Slack/Discord con resumen.
   - Métrica `dead_letter_total{type}` +1.
4. Si el workflow era un ticket del usuario: se envía un mensaje WhatsApp "Estamos teniendo un problema técnico, un operador te va a contactar" y se fuerza `handoff`.

## 4. Circuit breaker

Por cada dependencia externa (sistema principal, proveedor WA, S3, LLM), n8n mantiene en Static Data:

```json
{
  "dep": "main_api",
  "state": "closed|open|half_open",
  "failure_count": 0,
  "opened_at": null
}
```

Reglas:
- `closed` → 5 fallos consecutivos → `open`.
- `open` → por 60 segundos, todas las llamadas fallan rápido sin intentar.
- Después de 60s → `half_open` → siguiente llamada prueba; si OK → `closed`, si falla → `open` otros 60s.

En `open`, el bot entra en **modo degradado**: envía "Estamos con problemas técnicos, intentá en unos minutos" y termina.

## 5. Logging

Cada ejecución loguea un **registro estructurado por decisión**. Formato:

```json
{
  "ts": "2026-04-10T14:23:55Z",
  "level": "info|warn|error",
  "correlation_id": "...",
  "tenant_id": "...",
  "conversation_id": "...",
  "workflow": "WA - Conversation Orchestrator",
  "step": "intent_detected",
  "data": { "intent": "cargar_saldo", "confidence": 1.0 },
  "duration_ms": 42
}
```

Los logs se envían al sistema principal vía `POST /v1/logs` (batch) o, si se prefiere, a un stdout capturado + shipper (Loki/Datadog). En MVP, basta con `/v1/logs`.

## 6. Fallback técnico

Escenarios y respuestas:

| escenario | respuesta al usuario | estado n8n |
|---|---|---|
| Sistema principal caído (circuit open) | "Estamos con problemas técnicos, intentá en unos minutos." | log + no persist |
| LLM caído, modo `ai` | fallback a reglas automáticamente | log warn |
| S3 caído | "Tenemos un problema recibiendo imágenes, probá más tarde." | log error, no crear ticket |
| Proveedor WA falla al enviar | reintentos; si agotan, log + dead-letter | — |
| Error en Orchestrator (código) | Error Handler → "Estamos teniendo un problema, un operador te va a contactar" + handoff | log error + alerta |

## 7. Manejo de fallas externas (responsabilidades)

| Dependencia | Falla típica | Quién la absorbe |
|---|---|---|
| Proveedor WA (webhook) | Reentregas | Idempotencia en n8n + DB |
| Proveedor WA (envío) | 5xx, rate limit | Retries en n8n, circuit breaker |
| Sistema principal | 5xx, timeouts | Retries + circuit breaker + dead-letter |
| S3 | Network errors | Retries |
| LLM | Timeouts, rate limit | Fallback a reglas |

## 8. Invariantes de robustez

1. Ningún mensaje del usuario puede perderse silenciosamente. Si no puede procesarse, va al dead-letter **con** respuesta al usuario.
2. Ningún ticket puede crearse dos veces por el mismo mensaje. Idempotencia end-to-end.
3. El bot nunca queda "mudo" para el usuario. Siempre hay respuesta, aunque sea genérica.
4. Un tenant no puede afectar a otro. Un tenant con errores no impacta el circuit breaker de los demás (los breakers son por tenant + dependencia).
5. El estado persistido es consistente o no existe. Usar transacciones en el sistema principal.

## 9. Test de caos (recomendado en staging)

- Bajar el sistema principal por 2 minutos → verificar modo degradado.
- Bajar S3 → verificar mensaje correcto al usuario.
- Duplicar manualmente un webhook → verificar que sólo se procesa una vez.
- Inyectar latencia de 10s en una dependencia → verificar timeout y retry.
- Mandar 100 mensajes en 5s al mismo `phone` → verificar anti-flood.
