# WA - Handle Media Receipt

**Rol:** procesar imágenes/documentos: validar, descargar, hashear, subir a object store, dedupe, devolver media_id.

## Objetivo

- Aislar todo el manejo binario fuera del Orchestrator.
- Proveer `media_id`, `hash`, `storage_url` al Orchestrator.
- Marcar `risk_score` si se detecta duplicado cross-user.

## Inputs

- `inbound-normalized` con `type ∈ {image, document}`.
- `tenant_context`.

## Outputs

```json
{
  "media_id": "med_01HX...",
  "hash": "sha256:abcd...",
  "storage_url": "s3://...",
  "duplicate": false,
  "risk_score": "low",
  "block": false
}
```

## Nodos (n8n)

1. **Function — Validate mime/size** — rechazar si fuera de whitelist o > tamaño max.
2. **HTTP Request — Download from provider URL** (token de instancia, response binario).
3. **Function — Compute SHA-256** del buffer.
4. **HTTP Request — PUT a object store** (S3/R2/GCS). Path: `tenant/{tenant_id}/{yyyy}/{mm}/{hash}.{ext}`.
   - Retry: 3.
5. **HTTP Request — POST /v1/media** con `{ tenant_id, hash, storage_url, mime, size, conversation_id }`.
   - Headers: `Idempotency-Key: {{hash}}`.
   - Backend devuelve `{ media_id, duplicate, risk_score, block }`.
6. **IF — block?** → log + responder mensaje "tu solicitud está en revisión manual" y **no** crear ticket. Stop.
7. **IF — risk_score=high y duplicate cross-user** → forzar handoff via Escalate Human con reason `duplicate_media_cross_user`.
8. **Return** al Orchestrator.

## Decisiones

- **Hash en n8n, no en backend**: evita re-stream del binario y mantiene la integridad bajo control del lado que tiene el archivo.
- **Idempotencia por hash**: si la misma imagen llega dos veces, el media_id es el mismo.
- **Storage path por tenant**: aísla buckets y permite políticas IAM por tenant en el futuro.
- **Nunca exponer la URL del proveedor al usuario**.

## Errores posibles

| error | manejo |
|---|---|
| Mime no soportado | reply "Formato no soportado, enviá JPG/PNG/PDF" |
| Tamaño excesivo | reply "Imagen muy grande, enviá una más liviana" |
| Download falla | retry 3; si persiste, reply "No pude recibir tu imagen, reintentá" |
| S3 falla | retry; circuit breaker; reply técnico |
| Backend `/v1/media` falla | retry; si agotan, dead-letter |
