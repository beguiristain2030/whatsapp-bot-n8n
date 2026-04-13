# K. Checklist de Producción

Ninguna bandera en rojo antes de abrir tráfico real. Las marcadas como **blocker** detienen el go-live.

## Multi-tenant

- [ ] **blocker** `tenant_id` se resuelve en `WA - Resolve Tenant` y se propaga a todos los sub-workflows.
- [ ] **blocker** Toda llamada al sistema principal incluye `tenant_id` en headers o body y el backend la valida.
- [ ] **blocker** Mapping `instance_id → tenant_id` vive en DB, no en archivo.
- [ ] Credenciales del proveedor WA separadas por tenant en n8n Credentials.
- [ ] Test cross-tenant: mensaje de tenant A no aparece en tenant B.

## Idempotencia

- [ ] **blocker** Índice único `(tenant_id, provider_message_id)` en tabla `messages`.
- [ ] **blocker** `Idempotency-Key` en `POST /v1/tickets` (= `correlation_id`).
- [ ] **blocker** `Idempotency-Key` en `POST /v1/media` (= `sha256`).
- [ ] Test: reentregar el mismo webhook 3 veces → 1 solo ticket.

## Webhook

- [ ] **blocker** Firma HMAC del proveedor verificada en `WA - Inbound Router`.
- [ ] **blocker** Webhook responde 200 en < 1s.
- [ ] Procesamiento real va a sub-workflow asíncrono.
- [ ] Rate limit por `phone` (anti-flood) configurado.
- [ ] IPs del proveedor en allowlist si el proveedor las publica.

## Errores controlados

- [ ] **blocker** `WA - Error Handler` conectado a todos los sub-workflows críticos.
- [ ] **blocker** Dead-letter queue persistiendo en el sistema principal.
- [ ] **blocker** Retries con backoff configurados en `Create Ticket`, `Send Outbound`, `Handle Media`.
- [ ] Mensaje de fallback al usuario cuando hay fallas: nunca queda mudo.
- [ ] Circuit breaker por dependencia operativo (o riesgo aceptado y documentado).

## Handoff

- [ ] **blocker** `mode=human` bloquea 100% de respuestas del bot (test unitario).
- [ ] **blocker** Escalate Human marca `mode=human` en la DB y notifica al operador.
- [ ] Handoff automático al superar `turn_count > 10` sin progreso.
- [ ] Handoff automático por `risk_score=high`.
- [ ] Handoff por monto alto en `retirar_saldo` (threshold por tenant).
- [ ] Alerta si `mode=human` lleva > 4h sin cierre.

## Logs y auditoría

- [ ] **blocker** Cada mensaje genera al menos un registro en `audit` con `correlation_id`.
- [ ] Logs estructurados (JSON) en cada workflow.
- [ ] `correlation_id` trazable end-to-end.
- [ ] Retención de logs / audit definida (¿30/90 días?).

## Media

- [ ] **blocker** Hash SHA-256 calculado antes de enviar al sistema principal.
- [ ] **blocker** Dedupe cross-user implementado.
- [ ] **blocker** Validación de mime y tamaño antes de descarga.
- [ ] Upload a S3 (o equivalente) con path por tenant.
- [ ] URL temporal del proveedor nunca expuesta al usuario.

## Riesgo

- [ ] Scoring de riesgo devuelto por el sistema principal y respetado por el Orchestrator.
- [ ] Acciones con dinero tienen reglas deterministas, no IA.
- [ ] Flags de bloqueo manual por usuario/tenant funcionan.

## Observabilidad

- [ ] Métricas expuestas: mensajes/min, intents, tickets, handoffs, errores, p95 latencia.
- [ ] Dashboards en Grafana/Datadog/lo-que-sea.
- [ ] Alertas configuradas: dead-letter > 0, error rate > 2%, circuit open, latencia p95 degradada.
- [ ] Runbook de incidentes escrito en `docs/runbook.md`.

## Seguridad

- [ ] Secrets en n8n Credentials, no en variables de entorno planas ni en workflows.
- [ ] Backups cifrados de credenciales y DB del sistema principal.
- [ ] Acceso a n8n detrás de auth (SSO o usuario/password fuerte + 2FA).
- [ ] HTTPS obligatorio en el webhook.
- [ ] Rotación de credenciales documentada.

## Testing

- [ ] Suite de test cases de [../tests/cases.md](../tests/cases.md) pasando.
- [ ] Tests de caos corridos al menos una vez en staging.
- [ ] Piloto con un tenant amigo durante ≥ 1 semana sin incidentes.
- [ ] Playbook de rollback probado (volver al workflow anterior en < 5 min).

## Operaciones

- [ ] Procedimiento de onboarding de nuevo tenant documentado.
- [ ] Procedimiento de offboarding documentado (cerrar conversaciones, revocar credenciales).
- [ ] Contacto de guardia claro.
- [ ] Calendario de mantenimiento y ventana de cambios acordado.
