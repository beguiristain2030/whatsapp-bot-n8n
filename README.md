# WhatsApp Bot en n8n — Multi-tenant, producción

Motor de bot de WhatsApp sobre n8n. El estado conversacional **no vive en n8n**; vive en el sistema principal (API externa + DB). n8n es únicamente el orquestador de eventos y el puente con el proveedor de WhatsApp.

## Diseño en una línea

`WhatsApp → Webhook → Normalize → Resolve Tenant → Load Conversation → Detect Intent → Orchestrator → (Ticket | Handoff | Reply) → Audit`

## Navegación rápida

- Arquitectura: [docs/architecture.md](docs/architecture.md)
- Modelo de conversación: [docs/conversation-model.md](docs/conversation-model.md)
- Intención y reglas: [docs/intent-logic.md](docs/intent-logic.md)
- Media y riesgo: [docs/media-risk.md](docs/media-risk.md)
- Errores y robustez: [docs/errors-robustness.md](docs/errors-robustness.md)
- Plan de implementación: [docs/implementation-plan.md](docs/implementation-plan.md)
- Checklist de producción: [docs/production-checklist.md](docs/production-checklist.md)
- Contratos JSON: [contracts/](contracts/)
- Workflows n8n: [workflows/](workflows/)

## Principios no negociables

1. Todo evento lleva `tenant_id`, `conversation_id`, `correlation_id`.
2. El estado conversacional se persiste fuera de n8n.
3. `mode = human` bloquea completamente al bot.
4. Idempotencia por `provider_message_id`.
5. Logging y auditoría de cada decisión.
6. Media desacoplada en sub-workflow propio.
7. IA opcional en MVP, preparada por contrato.
