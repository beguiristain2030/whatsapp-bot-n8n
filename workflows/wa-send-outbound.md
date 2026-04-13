# WA - Send Outbound

**Rol:** enviar mensajes al proveedor WhatsApp con retries, traduciendo el contrato interno al shape del proveedor activo.

## Objetivo

- Interfaz única de envío. Quien quiera mandar un mensaje llama sólo a este workflow.
- Traducir `outbound-message.json` al shape del proveedor.
- Manejar retries/backoff del proveedor.
- Persistir el mensaje outbound en el sistema principal (auditoría).

## Inputs

- [../contracts/outbound-message.json](../contracts/outbound-message.json).

## Outputs

- `{ provider_message_id, status: "sent|failed" }`.

## Nodos (n8n)

1. **Switch — tenant_context.provider** — rama por proveedor.
2. **Function — Map to {provider} shape** — una función por rama.
3. **HTTP Request — POST al endpoint del proveedor**
   - Credencial: dinámica por `credentials_ref` del tenant.
   - Retry: 5 con backoff 500ms → 2s → 8s → 30s → 2min.
4. **IF — error tras retries?**
   - Sí → **Execute Workflow — WA - Error Handler** con `error_class=integration`, return `{ status: "failed" }`.
5. **HTTP Request — POST /v1/messages** con `direction=outbound` y el `provider_message_id` devuelto por el proveedor, para auditoría.
6. **Return**.

## Decisiones

- **Un único punto de salida**: simplifica credentials, retries y métricas.
- **Persistir outbound en `messages`**: permite reconstruir la conversación completa desde la DB sin depender del proveedor.
- **Soporte de templates** (para mensajes fuera de la ventana de 24h de WhatsApp Business): el contrato ya lo prevé.

## Errores posibles

| error | manejo |
|---|---|
| Fuera de ventana 24h, intento de texto libre | downgrade a template si está configurado; si no, log y fallar |
| 429 rate limit del proveedor | respetar backoff, retry |
| Credencial inválida | log crit, alerta, dead-letter |
| Mensaje rechazado por policy | log error, no retry, notificar al sistema principal |
