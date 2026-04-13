# I. Plan de Implementación

Tres fases, cada una con criterio de éxito explícito. Ninguna fase comienza hasta cerrar la anterior.

---

## FASE 1 — Base operativa (días 1-3)

**Objetivo:** un mensaje entra, se normaliza, se loguea, y el bot responde un eco estructurado. Cero lógica de negocio todavía.

### Incluye
- Proveedor WhatsApp elegido y conectado (Meta Cloud API, Evolution API o 360dialog).
- 1 instancia de n8n en ambiente `dev` con HTTPS público (ngrok/cloudflare tunnel/VPS).
- Webhook `WA - Inbound Router` recibiendo y respondiendo 200.
- `WA - Normalize Incoming` transformando al contrato `inbound-normalized.json`.
- `WA - Resolve Tenant` con mapping en archivo estático (no DB todavía).
- `WA - Send Outbound` enviando un mensaje de eco.
- Credenciales del proveedor cargadas en n8n Credentials.
- Logging básico por `console.log` / `WA - Audit Logger` a stdout.
- Repo git con esta estructura y workflows exportados a `exports/`.

### No incluye
- Sistema principal conectado.
- Detección de intención real.
- Estado conversacional.
- Media.
- Handoff.
- Retries avanzados ni circuit breaker.

### Criterio de éxito
- Envío un "hola" al número del bot → recibo un eco con `{tenant_id, correlation_id}` visible.
- Latencia del webhook 200 < 1 segundo.
- Logs muestran el `correlation_id` propagado por los sub-workflows.

---

## FASE 2 — MVP robusto (días 4-10)

**Objetivo:** el bot maneja intents reales, mantiene estado, crea tickets y escala a humanos. Lista para un piloto con tenant amigo.

### Incluye
- Sistema principal con endpoints mínimos listos:
  - `GET/POST /v1/messages` con idempotencia.
  - `GET/PUT /v1/conversations`.
  - `POST /v1/tickets`.
  - `POST /v1/handoff`.
  - `POST /v1/media`.
  - `POST /v1/audit`, `POST /v1/logs`, `POST /v1/dead_letter`.
- `WA - Load Conversation` leyendo estado real.
- `WA - Detect Intent Rules` con [../mappings/intent-keywords.json](../mappings/intent-keywords.json).
- `WA - Conversation Orchestrator` con máquina de estados completa.
- `WA - Create Ticket` con `Idempotency-Key`.
- `WA - Handle Media Receipt` con hash SHA-256, upload S3, dedupe.
- `WA - Escalate Human` marcando `mode=human`.
- `WA - Audit Logger` escribiendo a sistema principal.
- `WA - Error Handler` + dead-letter.
- Retries con backoff configurables.
- Tests de contrato en `tests/` (ver [../tests/cases.md](../tests/cases.md)).
- Multi-tenant real: mapping en DB, resolución por `instance_id`.
- Al menos 2 tenants configurados.

### No incluye
- IA para intención (queda el contrato preparado).
- Circuit breaker (queda como TODO marcado).
- Broadcast / campañas.
- Dashboard operador propio (se usa el del sistema principal).

### Criterio de éxito
- Happy path `cargar_saldo` end-to-end: texto + imagen → ticket creado → respuesta al usuario.
- Happy path `retirar_saldo` con handoff obligatorio por monto.
- Handoff manual (`"quiero hablar con alguien"`) funciona y el bot deja de responder.
- Duplicado de `provider_message_id` no genera dos tickets.
- Duplicado de hash de imagen cross-user genera `risk_score=high` y handoff.
- Error simulado en `POST /v1/tickets` genera mensaje al usuario y dead-letter.
- Los 2 tenants funcionan sin cross-contamination.
- Corrida de los test cases de [../tests/cases.md](../tests/cases.md) al 100%.

---

## FASE 3 — Producción (días 11-20)

**Objetivo:** listo para tráfico real de múltiples tenants con SLOs definidos.

### Incluye
- Ambientes separados: `dev`, `staging`, `prod` con secrets independientes.
- Firma HMAC del webhook verificada.
- Rate limit por `phone` y por `tenant` (en n8n o en un gateway delante).
- Circuit breaker por dependencia.
- Métricas exportadas:
  - mensajes por minuto por tenant
  - intents distribution
  - tickets creados
  - % handoff
  - errores por workflow
  - p50/p95 latencia de webhook
- Alertas configuradas (Slack/Discord) para: dead-letter, circuit breaker abierto, % error > 2%, `mode=human` > 4h sin cierre.
- Backups de configuración: export de workflows a git, versionado semver.
- Runbook de incidentes en `docs/runbook.md` (a crear en esta fase).
- Feature flags por tenant: `intent_engine`, `risk_rules`, `media_enabled`.
- Onboarding documentado para agregar un nuevo tenant (5-step guide).
- Opcional: `WA - Detect Intent AI` como segundo motor, activable por flag.

### No incluye
- Multi-región.
- Autoscaling del propio n8n (se asume 1 instancia bien dimensionada).
- Chat UI embebido en la web (fuera de alcance).

### Criterio de éxito
- SLOs:
  - Latencia webhook p95 < 800ms.
  - Latencia respuesta usuario p95 < 3s (sin media).
  - Disponibilidad mensual ≥ 99.5%.
  - Tasa de error por mensaje < 1%.
- Tests de caos pasados (ver [errors-robustness.md § 9](errors-robustness.md)).
- 1 semana en staging sin dead-letters no justificados.
- Playbook de rollback probado.
- Checklist de producción [production-checklist.md](production-checklist.md) completo.

---

## Timeline resumido

```
Día 1-3   ▓▓▓░░░░░░░░░░░░░░░░░  Fase 1 — Base
Día 4-10  ░░░▓▓▓▓▓▓▓░░░░░░░░░░  Fase 2 — MVP
Día 11-20 ░░░░░░░░░░▓▓▓▓▓▓▓▓▓▓  Fase 3 — Prod
```

Ajustar según dependencias del sistema principal: **si los endpoints del backend no están listos, la Fase 2 se bloquea.** Priorizar el backend en paralelo a la Fase 1.
