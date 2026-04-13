# tests/

Casos de integración end-to-end para el bot.

- [cases.md](cases.md) — lista obligatoria por fase.
- Formato ejecutable futuro: Postman collection / newman / k6.

## Cómo correr (manual, MVP)

1. Levantar n8n + backend en `staging`.
2. Resetear fixtures (`make reset-fixtures` en backend).
3. Simular webhooks con `curl`:
   ```bash
   curl -X POST https://n8n-staging.tuapp.com/webhook/wa-inbound/evolution_api \
     -H "X-Hub-Signature-256: sha256=..." \
     -H "Content-Type: application/json" \
     -d @fixtures/tc01_text_cargar.json
   ```
4. Verificar resultados en backend (`GET /v1/conversations`, `/v1/tickets`, `/v1/messages`).
5. Marcar cada caso en [cases.md](cases.md) como pasado o no.

## Próximo paso (Fase 3)

- Automatizar con newman/k6 en CI.
- Fixtures versionadas en `tests/fixtures/`.
