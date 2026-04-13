# Test Cases — WhatsApp Bot

Casos obligatorios antes de aprobar cada fase. Ejecutarlos como integration tests (curl/postman/newman) contra un n8n + backend de staging.

## Convenciones

- Cada caso tiene **Setup → Stimulus → Expected**.
- `wamid-N` = id sintético de mensaje del proveedor, único por caso.
- Backend y n8n limpios al inicio de la suite (fixtures reset).

---

## TC-01 — Happy path: cargar saldo completo

- **Setup**: tenant `tenant_acme` configurado, phone `+549...01` sin conversación previa.
- **Stimulus**:
  1. Webhook inbound: text `"quiero cargar 5000"`, wamid-1.
  2. Webhook inbound: image JPG 200KB con caption vacío, wamid-2.
- **Expected**:
  - Conversación creada, `current_intent=cargar_saldo`, `collected_data.amount=5000`.
  - Tras imagen: `media` persistida, `duplicate=false`, `risk_score=low`.
  - Ticket creado con `type=deposit`, `payload.amount=5000`, `payload.receipt.hash` válido.
  - 2 mensajes outbound al usuario: pedido de comprobante + confirmación ticket.
  - Conversación final en estado `resolved`.

## TC-02 — Idempotencia de webhook

- **Setup**: TC-01 completado.
- **Stimulus**: reenviar webhook con wamid-1 **3 veces**.
- **Expected**:
  - Sólo 1 row en `messages`.
  - Sólo 1 ticket (el original).
  - 2 de los 3 POST /v1/messages responden 409 `duplicate`.
  - El bot **no** envía respuesta a los duplicados.

## TC-03 — Handoff explícito

- **Stimulus**: text `"quiero hablar con una persona"`, wamid-3.
- **Expected**:
  - Conversación → `state=handoff_to_human`, `mode=human`.
  - POST /v1/handoff con `reason=user_requested`.
  - Send Outbound confirmando derivación.
  - Siguiente mensaje del usuario (wamid-4): **el bot no responde**, sólo se persiste en `messages` y se audita con `decision=bot_silent_human_mode`.

## TC-04 — Retiro con monto alto → handoff obligatorio

- **Setup**: tenant con `high_amount_threshold=100000`.
- **Stimulus**: text `"quiero retirar 250000"`, wamid-5.
- **Expected**:
  - Intent `retirar_saldo` detectado.
  - Regla de monto alto dispara handoff antes de pedir CBU.
  - `reason=high_amount`.
  - No se crea ticket de retiro.

## TC-05 — Duplicado de imagen cross-user

- **Setup**: user A (`+549...01`) ya envió una imagen con hash H en TC-01.
- **Stimulus**: user B (`+549...02`) envía la **misma imagen** (mismo hash), text previo `"cargar 3000"`.
- **Expected**:
  - `POST /v1/media` devuelve `duplicate=true`, `risk_score=high`.
  - Conversación de B → handoff con `reason=duplicate_media_cross_user`.
  - Risk event persistido.
  - No se crea ticket de deposit.

## TC-06 — Fallback exhausted

- **Stimulus**: 3 mensajes seguidos unknown: `"asdf"`, `"qwer"`, `"zxcv"`.
- **Expected**:
  - `_fallback_count` llega a 3.
  - Automático handoff `reason=fallback_exhausted`.

## TC-07 — Anti-loop por turn_count

- **Setup**: conversación en `collecting_data` para `cargar_saldo`, pidiendo `amount`.
- **Stimulus**: 11 mensajes sin dar un número válido.
- **Expected**: handoff `reason=turn_limit` al turno 11.

## TC-08 — Backend caído durante Create Ticket

- **Setup**: forzar 500 en `/v1/tickets` durante 5 min.
- **Stimulus**: flujo de `cargar_saldo` completo.
- **Expected**:
  - 5 reintentos con backoff.
  - Error Handler → dead-letter.
  - Handoff `reason=ticket_creation_failed`.
  - Usuario recibe mensaje técnico explicativo.
  - Alerta enviada al canal de alertas.

## TC-09 — Tenant no resuelto

- **Stimulus**: webhook desde un `instance_id` inexistente.
- **Expected**:
  - `Resolve Tenant` devuelve 404.
  - No se responde al usuario.
  - Se loguea security event.
  - Dead-letter con `tenant_id=null`.

## TC-10 — Mensaje no soportado (audio)

- **Stimulus**: audio de 30s.
- **Expected**:
  - Normalize → `type=audio`.
  - Orchestrator → respuesta "Aún no puedo procesar audios, escribime por texto o mandame una imagen".
  - Conversación no cambia de estado.

## TC-11 — Firma HMAC inválida

- **Stimulus**: POST al webhook con body válido pero firma aleatoria.
- **Expected**: 401, sin procesamiento, log de seguridad.

## TC-12 — Expiración de conversación

- **Setup**: conversación con `expires_at < now`.
- **Stimulus**: nuevo mensaje del mismo phone.
- **Expected**:
  - Conversación vieja cerrada con `resolved_by_timeout`.
  - Nueva conversación creada en `bot_active`.

## TC-13 — Cambio de intent mid-conversación

- **Setup**: `collecting_data` para `cargar_saldo`, ya tiene `amount=5000`, falta comprobante.
- **Stimulus**: text `"mejor quiero retirar 2000"`.
- **Expected**:
  - Bot confirma el cambio: "Veo que ahora preferís retirar. ¿Confirmás?".
  - Tras confirmación (`"si"`), se resetea `collected_data` y se empieza `retirar_saldo`.

## TC-14 — Cross-tenant aislado

- **Setup**: 2 tenants, mismo phone en ambos.
- **Stimulus**: mensaje a instance del tenant A.
- **Expected**: la conversación del tenant B no se toca.

## TC-15 — Media grande rechazada

- **Stimulus**: imagen 12 MB (límite 8 MB).
- **Expected**: respuesta "Imagen muy grande...", no se descarga.
