# exports/

Exportaciones JSON de los workflows de n8n. Versionadas con git.

## Convención

- `wa-inbound-router.v1.0.0.json`
- `wa-normalize-incoming.v1.0.0.json`
- ...

## Flujo recomendado

1. Modificar el workflow en la UI de n8n (en `dev`).
2. Exportar (`... → Download`) y guardar acá con bump de versión.
3. Commit en git con mensaje `feat(wa): <workflow> v1.1.0 — <cambio>`.
4. Importar en `staging` → probar → importar en `prod`.

## Rollback

Para revertir, re-importar el JSON de la versión anterior y desactivar la versión en uso.

## Estado actual

Vacío. Llenar cuando se construyan los workflows reales en n8n (Fase 1).
