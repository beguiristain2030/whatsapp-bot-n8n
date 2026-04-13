# A. Arquitectura General

> Equipo: Arquitecto SaaS, Especialista n8n, Backend, Ing. Conversaciones, QA, Security.
> Audiencia: ingeniería de producto implementando el MVP y llevándolo a producción.

---

## 1. Vista de sistema

```
┌─────────────────┐     ┌───────────────┐     ┌────────────────────┐
│  WhatsApp       │     │   n8n         │     │  Sistema Principal │
│  (Meta Cloud /  │────▶│  (motor bot)  │────▶│  (API + DB)        │
│  Evolution API /│ WH  │               │ API │                    │
│  360dialog)     │◀────│               │◀────│  Tickets, Users,   │
└─────────────────┘     └───────┬───────┘     │  Conversaciones,   │
                                │             │  Operadores, Audit │
                                │             └────────────────────┘
                                │
                         ┌──────▼────────┐
                         │ Object Store  │  (media, comprobantes)
                         │ S3 / R2 / GCS │
                         └───────────────┘
```

**Regla de oro:** n8n no es fuente de verdad. La DB del sistema principal sí lo es. n8n orquesta, normaliza, decide y llama APIs. No guarda estado persistente por su cuenta (más allá de credenciales y workflows).

---

## 2. Responsabilidades — n8n vs Sistema Principal

| Responsabilidad | n8n | Sistema Principal |
|---|---|---|
| Recepción webhook del proveedor WA | ✅ | — |
| Normalización de payloads | ✅ | — |
| Resolución de tenant | ✅ (consulta mapping) | ✅ (fuente de verdad en DB) |
| Idempotencia (dedupe por `provider_message_id`) | ✅ (check + write) | ✅ (índice único) |
| Estado conversacional | ❌ | ✅ (read/write vía API) |
| Detección de intención (rules) | ✅ | — |
| Detección de intención (IA) | ✅ (llama LLM) | — |
| Persistencia de mensajes | ❌ | ✅ |
| Persistencia de conversación | ❌ | ✅ |
| Creación de tickets | ❌ (sólo dispara API) | ✅ |
| Handoff humano (bandera `mode=human`) | ❌ (sólo dispara API) | ✅ |
| Envío de mensajes salientes al proveedor | ✅ | — |
| Hashing de media y upload | ✅ | — |
| Dedupe de comprobantes | ❌ (consulta API) | ✅ |
| Auditoría final | ❌ | ✅ |
| Logging técnico del workflow | ✅ | — |
| Circuit breaker / retries a APIs externas | ✅ | ✅ |

**Decisión crítica #1:** El sistema principal expone una API REST estable (`/v1/...`). n8n la consume con un HTTP Request node reusable, con autenticación por JWT de servicio por tenant.

**Decisión crítica #2:** El estado conversacional se carga al inicio de cada ejecución y se escribe al final. Nunca se asume continuidad en memoria de n8n entre ejecuciones.

**Decisión crítica #3:** Multi-tenant se resuelve por `(instance_id | destination_number)` → `tenant_id`. La tabla de mapping vive en la DB principal y se cachea con TTL corto (ver §6).

---

## 3. Workflow principal y sub-workflows

### Workflow principal
- **WA - Inbound Router** — único webhook público. Recibe del proveedor, valida firma, responde 200 inmediato, dispara sub-workflow en modo "Execute Workflow".

### Sub-workflows (todos invocados por Execute Workflow)
1. **WA - Normalize Incoming** — transforma el payload del proveedor al contrato `inbound-normalized.json`.
2. **WA - Resolve Tenant** — identifica `tenant_id` desde `instance` / `to_number`.
3. **WA - Load Conversation** — GET conversación + estado + `mode`.
4. **WA - Detect Intent Rules** — clasificación por keywords/entidades.
5. **WA - Conversation Orchestrator** — máquina de estados, decide acción.
6. **WA - Create Ticket** — POST al sistema principal.
7. **WA - Handle Media Receipt** — descarga media, hash, upload, dedupe.
8. **WA - Escalate Human** — marca `mode=human` y notifica.
9. **WA - Send Outbound** — envía al proveedor WA con retries.
10. **WA - Audit Logger** — log centralizado de cada decisión.
11. **WA - Error Handler** — captura errores, dead-letter, alertas.

Ver [../workflows/](../workflows/) para el detalle de cada uno.

---

## 4. Flujo happy-path (texto)

```
1. Usuario envía "quiero cargar 5000"
2. Proveedor → POST /webhook/wa-inbound → Inbound Router
3. Router responde 200 y dispara Normalize
4. Normalize → contrato inbound-normalized
5. Resolve Tenant → tenant_id
6. Load Conversation → { state: "bot_active", mode: "bot" }
7. Detect Intent Rules → intent="cargar_saldo", entities={amount:5000}
8. Orchestrator:
   - ¿datos suficientes? → falta comprobante
   - pide comprobante → state="waiting_user", collecting="receipt_image"
9. Persist conversation (PUT)
10. Send Outbound → "Perfecto, enviame el comprobante"
11. Audit Logger → registra decisión
```

Siguiente mensaje (imagen):

```
1. Usuario envía imagen
2. Inbound Router → Normalize (type=image)
3. Handle Media Receipt → descarga, hash SHA-256, upload a S3, dedupe API
4. Load Conversation → { collecting: "receipt_image", intent: "cargar_saldo" }
5. Orchestrator: datos completos → Create Ticket
6. Ticket creado → state="resolved" o "waiting_human" según reglas
7. Send Outbound → "Ticket #1234 creado, un operador lo revisará"
8. Audit Logger
```

---

## 5. Flujo con handoff humano

- Usuario escribe "quiero hablar con una persona" → intent=`handoff`.
- Orchestrator → Escalate Human.
- Sistema principal marca `mode=human`.
- Desde ese momento, al inicio del Orchestrator: si `mode == human`, el bot **no responde nada**, sólo registra el mensaje en auditoría y termina. El operador ve el mensaje en el panel del sistema principal.
- El operador cierra la conversación → sistema principal vuelve `mode=bot`.

**Regla:** el bot jamás responde si `mode=human`. No hay excepciones en MVP.

---

## 6. Decisiones críticas

| # | Decisión | Racional |
|---|---|---|
| D1 | API REST como única interfaz entre n8n y sistema principal | Evita acoplamiento a DB, permite versionado |
| D2 | Estado conversacional en sistema principal | n8n no es DB; ejecuciones son efímeras |
| D3 | Idempotencia por `provider_message_id` con índice único en DB | Reentregas del proveedor son habituales |
| D4 | `correlation_id` generado en Inbound Router, propagado a todos los sub-workflows y logs | Trazabilidad end-to-end |
| D5 | Cache de `tenant_mapping` en n8n (Static Data) con TTL 5 min | Reduce latencia, acepta inconsistencia corta |
| D6 | `mode=human` bloquea todo, sin excepciones en MVP | Evita falsos positivos que molestan al usuario real |
| D7 | Media va a object store primero, luego hash, luego API | Permite reintento sin re-descargar del proveedor |
| D8 | LLM es opcional, detrás de un sub-workflow `WA - Detect Intent AI` intercambiable con `Rules` | MVP por reglas, IA como drop-in |
| D9 | Webhook responde 200 antes de procesar (fire-and-forget con Execute Workflow async) | Proveedores WA castigan latencia > 5s |
| D10 | Un workflow por responsabilidad, no workflows "monolito" | Debug, reuso, test unitario |

---

## 7. Riesgos y mitigaciones

| Riesgo | Impacto | Mitigación |
|---|---|---|
| Reentrega de webhooks del proveedor | Duplicados, doble ticket | Idempotencia por `provider_message_id` en DB y short-circuit en n8n |
| Latencia del webhook > 5s | Proveedor retry masivo, ciclo | Responder 200 de inmediato, procesar async |
| Cambio de schema del proveedor | Bot caído | Normalize aislado; un solo punto de cambio |
| LLM hallucina intenciones de riesgo (retiro) | Fraude | Reglas deterministas para acciones con dinero, LLM sólo sugiere |
| Estado inconsistente n8n ↔ DB | Loops o respuestas incorrectas | DB es fuente de verdad; n8n no guarda estado |
| Media muy grande / tipo no soportado | Crash workflow | Validar tamaño/mime antes de procesar; límites explícitos |
| Operador olvida cerrar `mode=human` | Bot "silencioso" indefinido | Alerta al sistema principal si `mode=human` > N horas |
| Tenant mal resuelto | Fuga de datos cross-tenant | Validar `tenant_id` en **cada** llamada API; rechazar si null |
| Errores en Create Ticket | Usuario queda sin respuesta | Dead-letter + reintento + escalamiento a humano |
| Exposición del webhook | Spam / abuso | HMAC firma del proveedor, allowlist IP si aplica, rate limit |

---

## 8. Recomendaciones de producción

- **Un webhook por entorno**: `dev`, `staging`, `prod`. Nunca compartir.
- **Credenciales por tenant**: cada tenant tiene su instancia/API key del proveedor WA guardada en n8n Credentials.
- **Observabilidad**: exportar métricas (Prometheus via `/metrics` en el sistema principal) de: mensajes/min por tenant, intent distribution, tickets creados, tasa de handoff, errores.
- **Versionado de workflows**: exportar JSON a `exports/` y commitear a git con convención semver.
- **Tests de contrato**: el contrato `inbound-normalized.json` es el límite; romperlo requiere bump de versión.
- **Feature flags**: IA, reglas de riesgo, dedupe de media — activables por tenant desde el sistema principal.

---

## 9. Fuera de alcance (explícito)

- Chat multimodal avanzado (voz → texto).
- Flujos de pago dentro de WhatsApp.
- Sincronización bidireccional con CRMs externos.
- Broadcast/campañas salientes proactivas.

Todo esto es factible pero no es parte del MVP de este bot.
