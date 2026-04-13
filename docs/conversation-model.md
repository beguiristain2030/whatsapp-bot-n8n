# E. Modelo de Conversación

## 1. Estados

| Estado | Descripción | El bot responde? |
|---|---|---|
| `bot_active` | Conversación fresca o en reposo, lista para nueva intención | Sí |
| `waiting_user` | Bot envió pregunta, espera respuesta libre | Sí cuando llega |
| `collecting_data` | Bot está recolectando campos específicos para completar una intención | Sí |
| `ready_to_create_ticket` | Todos los datos listos, próximo paso es POST a sistema principal | Sí (transición automática) |
| `handoff_to_human` | Escalada, `mode=human` | **No** |
| `resolved` | Intención cerrada (ticket creado, consulta respondida) | Sí (nueva intención la reabre) |

**Nota:** `handoff_to_human` implica `mode=human` en la tabla de conversación. El `mode` es la bandera autoritativa; el `state` es semántico.

## 2. Estructura de datos (persistida en sistema principal)

```json
{
  "conversation_id": "conv_01HXYZ...",
  "tenant_id": "tenant_acme",
  "phone": "+5491155555555",
  "instance_id": "wa_instance_01",
  "state": "collecting_data",
  "mode": "bot",
  "current_intent": "cargar_saldo",
  "collected_data": {
    "amount": 5000,
    "receipt_image_url": null,
    "receipt_hash": null
  },
  "required_fields": ["amount", "receipt_image_url"],
  "missing_fields": ["receipt_image_url"],
  "last_bot_question": "Enviame el comprobante como imagen, por favor.",
  "last_user_message_id": "wamid.HBgM...",
  "turn_count": 3,
  "created_at": "2026-04-10T14:22:10Z",
  "updated_at": "2026-04-10T14:23:55Z",
  "expires_at": "2026-04-11T14:22:10Z",
  "handoff": {
    "active": false,
    "reason": null,
    "requested_at": null,
    "operator_id": null
  },
  "metadata": {
    "source": "whatsapp",
    "provider": "evolution_api",
    "correlation_ids": ["corr_01HX..."]
  }
}
```

## 3. Persistencia

- **Tabla `conversations`** en DB del sistema principal.
- Índices: `(tenant_id, phone)` único activo, `(tenant_id, state)`, `(updated_at)`.
- **TTL lógico:** `expires_at = updated_at + 24h`. Si un mensaje llega después del expiry, se crea una conversación nueva (cierre automático de la vieja con estado `resolved_by_timeout`).
- **Escrituras**: siempre vía API `PUT /v1/conversations/{id}` con optimistic locking por `updated_at`.

## 4. Transiciones de estado

```
         ┌────────────────────────────────────────────┐
         │                                            │
         ▼                                            │
    bot_active ──intent detectada──▶ collecting_data  │
         │                                │           │
         │                                │           │
         ▼                                ▼           │
    waiting_user                  ready_to_create_ticket
         │                                │           │
         │                                ▼           │
         │                           ticket creado ───┤
         │                                            │
         └──────handoff requerido──▶ handoff_to_human │
                                          │           │
                                          ▼           │
                                     (operador)       │
                                          │           │
                                          ▼           │
                                      resolved ───────┘
```

### Reglas de transición (ejecutadas en el Orchestrator)

| Desde | Evento | Hacia | Condición |
|---|---|---|---|
| `bot_active` | intent ≠ `unknown` | `collecting_data` | intent requiere campos |
| `bot_active` | intent = `consulta` | `resolved` | respuesta inmediata |
| `bot_active` | intent = `handoff` | `handoff_to_human` | set `mode=human` |
| `collecting_data` | todos los campos OK | `ready_to_create_ticket` | — |
| `collecting_data` | falta un campo | `collecting_data` | pedir siguiente |
| `collecting_data` | usuario pide humano | `handoff_to_human` | — |
| `collecting_data` | turn_count > 10 sin progreso | `handoff_to_human` | anti-loop |
| `ready_to_create_ticket` | ticket creado OK | `resolved` | — |
| `ready_to_create_ticket` | ticket falla N veces | `handoff_to_human` | dead-letter |
| `resolved` | nuevo mensaje usuario | `bot_active` | reapertura |
| `handoff_to_human` | operador cierra | `resolved` | vía API del sistema |

## 5. Continuidad multi-turno

Cada turno del bot sigue el ciclo:

```
1. Load conversation → estado + datos previos
2. Mergear mensaje nuevo con collected_data
3. Re-evaluar missing_fields
4. Decidir siguiente acción (pedir dato / crear ticket / escalar)
5. Persistir conversación actualizada
6. Responder
```

**Invariante:** después de cada mensaje, `missing_fields ⊆ required_fields`, y `collected_data` contiene exactamente las keys ya resueltas.

## 6. Reglas duras

- `mode=human` **bloquea** cualquier respuesta automática. El Orchestrator sale temprano si lo detecta.
- Si `current_intent` cambia a mitad de una recolección, se confirma al usuario ("Veo que ahora preferís cancelar, ¿confirmás?").
- Nunca borrar `collected_data` sin pasar por un estado terminal (`resolved` o `handoff_to_human`).
- `turn_count` se incrementa sólo con mensajes del usuario (no con los del bot).
