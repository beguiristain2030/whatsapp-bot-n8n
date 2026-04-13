# WA - Conversation Orchestrator

**Rol:** cerebro del bot. Aplica la máquina de estados, decide la siguiente acción y dispara el sub-workflow correspondiente.

## Objetivo

- Decidir: pedir dato / crear ticket / escalar / responder consulta / fallback.
- Aplicar las reglas anti-loop y de escalamiento.
- Persistir el nuevo estado de la conversación al final.

## Inputs

- `inbound-normalized`
- `conversation` cargada
- Resultado de `WA - Detect Intent Rules`
- `tenant_context` (features)

## Outputs

- Conversación actualizada (PUT)
- Mensaje outbound enviado (vía `WA - Send Outbound`)
- Eventualmente: ticket creado / handoff disparado

## Nodos (n8n)

1. **Function — Guard: mode=human** — si sí, NO debería haber llegado aquí (Load Conversation corta); si llega, log crit y Stop.
2. **Function — Merge intent into conversation**:
   - Si la conversación estaba en `collecting_data` y el nuevo mensaje no es claro, mantener `current_intent` previo.
   - Si es un intent nuevo y claro, reemplazar.
3. **Function — Merge entities into collected_data**.
4. **Function — Recompute missing_fields** según catálogo de [../docs/intent-logic.md § 1](../docs/intent-logic.md).
5. **Function — Evaluate transitions** — consulta [../mappings/state-transitions.json](../mappings/state-transitions.json), decide evento (`fields_complete` / `field_missing` / `user_requested_human` / etc.).
6. **Switch — next_action**:
   - `ask_next_field` → construir pregunta por template y llamar **Send Outbound**.
   - `create_ticket` → **Execute Workflow — WA - Create Ticket**.
   - `handoff` → **Execute Workflow — WA - Escalate Human**.
   - `answer_consulta` → resolver vía backend `GET /v1/users/{phone}/balance` u otros, y **Send Outbound**.
   - `fallback_reply` → pregunta genérica + incrementar `_fallback_count`.
7. **HTTP Request — PUT /v1/conversations/{id}** con el nuevo estado, usando `If-Match`.
8. **Execute Workflow — WA - Audit Logger** con `decision`, `intent`, `state_transition`.

## Side effects (orden obligatorio)

```
1. Compute next state
2. Fire side effect (create ticket / handoff / answer)
3. Persist conversation  ← siempre DESPUÉS del side effect exitoso
4. Send outbound reply
5. Audit
```

**Racional:** si la persistencia fallara, el side effect ya ocurrió y el usuario tendrá respuesta. Es preferible un state "re-intento" que un usuario mudo.

## Reglas anti-loop (aplicadas siempre)

- `turn_count > 10` sin cambio en `collected_data` → forzar `handoff`.
- `_fallback_count >= 3` → forzar `handoff`.
- `risk_score >= high` → forzar `handoff`.
- `intent = retirar_saldo` y `amount > tenant.high_amount_threshold` → forzar `handoff`.

## Errores posibles

| error | manejo |
|---|---|
| Create Ticket falla tras retries | forzar handoff con reason `ticket_creation_failed` + mensaje al usuario |
| PUT conversación 409 | re-GET + re-merge + retry (máx 2) |
| Backend caído mid-flow | Error Handler + mensaje genérico al usuario |
| Estado imposible (ej. `resolved` + mensaje del usuario) | reabrir a `bot_active` y procesar |

## Templates de respuesta (ejemplos)

- `cargar_saldo - ask_amount`: "Perfecto, ¿cuánto querés cargar? Ponelo sólo en números."
- `cargar_saldo - ask_receipt`: "Genial. Ahora enviame el comprobante como imagen."
- `cargar_saldo - ticket_created`: "Listo ✓ Ticket #{ticket_id} creado. Un operador lo revisará a la brevedad."
- `retirar_saldo - handoff_amount`: "Para retiros de este monto necesito derivarte a un operador. Ya estoy avisando."
- `handoff - confirm`: "Ok, te derivo con una persona del equipo. En breve te contactan por acá mismo."
- `fallback_3`: "No estoy pudiendo ayudarte por acá. Te derivo con una persona."
