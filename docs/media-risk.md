# G. Media y Riesgo

## 1. Pipeline de media

```
Mensaje con image/audio
        │
        ▼
WA - Handle Media Receipt
        │
        ├─▶ 1. Obtener URL temporal del proveedor (con token de instancia)
        ├─▶ 2. Validar mime/tamaño (límites configurables por tenant)
        ├─▶ 3. Descargar binario (HTTP Request con buffer)
        ├─▶ 4. Calcular SHA-256 del binario
        ├─▶ 5. Upload a object store: s3://tenant/{tenant_id}/{yyyy}/{mm}/{hash}.{ext}
        ├─▶ 6. POST /v1/media a sistema principal con hash + URL final
        │       └─▶ sistema responde: { duplicate: bool, risk_score: low|medium|high, media_id }
        ├─▶ 7. Mergear resultado en collected_data
        └─▶ 8. Continuar al Orchestrator
```

## 2. Reglas de validación previas a descarga

| check | límite default | acción si falla |
|---|---|---|
| mime whitelist | `image/jpeg, image/png, image/webp, application/pdf` | rechazar, pedir otro formato |
| tamaño máximo | 8 MB (configurable por tenant) | rechazar, pedir más liviano |
| duración audio | N/A en MVP | — |
| cantidad por conversación | 5 por turno | rechazar, aviso |

## 3. Hashing y dedupe

- Algoritmo: **SHA-256** sobre el binario crudo (no sobre URL ni metadata).
- El hash se envía al sistema principal, que mantiene un índice único `(tenant_id, hash)`.
- Si existe → `duplicate: true`.

**Reglas ante duplicado:**

| Contexto | Acción |
|---|---|
| Mismo usuario, misma conversación activa, <5min | Ignorar (reenvío accidental) |
| Mismo usuario, conversación anterior | Marcar `risk_score=medium`, continuar con advertencia |
| Usuario distinto, mismo hash, <24h | `risk_score=high`, handoff automático |
| Usuario distinto, mismo hash, cualquier ventana | `risk_score=high`, handoff + alerta de seguridad |

## 4. Risk scoring

El scoring **vive en el sistema principal**, no en n8n. n8n sólo consume el resultado.

Inputs que la API considera (referencia, no exhaustivo):

- Duplicado de hash (sí/no, ventana temporal, cross-user)
- Monto declarado vs histórico del usuario
- Velocidad de mensajes (flood)
- Horario atípico
- Conversación nueva vs recurrente
- Flags administrativas del tenant

Output normalizado:
```json
{ "risk_score": "low|medium|high", "reasons": ["duplicate_hash_cross_user"], "block": false }
```

## 5. Reglas de escalamiento por riesgo

| risk_score | acción del bot |
|---|---|
| `low` | continuar normal |
| `medium` | continuar, pero registrar warning y mostrar mensaje "Estamos revisando tu comprobante, puede tardar un momento" |
| `high` | detener flujo, escalar a humano con razón, **no** crear ticket automáticamente |
| `block: true` | responder mensaje genérico "Tu solicitud está en revisión manual", NO crear ticket, NO handoff automático (alguien del equipo de fraude lo revisa) |

## 6. Auditoría de media

Cada evento de media genera un registro en `audit` con:

```json
{
  "correlation_id": "...",
  "tenant_id": "...",
  "conversation_id": "...",
  "media_id": "...",
  "hash": "sha256:...",
  "size_bytes": 123456,
  "mime": "image/jpeg",
  "duplicate": false,
  "risk_score": "low",
  "storage_url": "s3://...",
  "decision": "continue|warn|escalate|block"
}
```

## 7. Observaciones de seguridad

- **Nunca** exponer la URL temporal del proveedor al usuario final.
- **Nunca** guardar el binario en el payload del workflow; usar buffer de n8n + streaming directo a S3.
- Las credenciales de S3 son por tenant si es posible (bucket por tenant o prefix con IAM policy restrictiva).
- El hash se computa en n8n; si se delegase al sistema principal, abriría un vector de confianza innecesario.
