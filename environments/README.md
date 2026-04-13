# environments/

Configuración por ambiente.

- [.env.example](.env.example) — plantilla. Copiar a `.env.dev`, `.env.staging`, `.env.prod` y NO commitear.
- Las credenciales sensibles (API keys del proveedor, JWT de servicio) se cargan en **n8n Credentials**, no en variables de entorno planas.
- Variables en `.env` sirven para URLs, timeouts, feature flags.

## Reglas

1. Tres ambientes separados: `dev` / `staging` / `prod`.
2. Cada ambiente tiene su propio webhook URL y su propio secret HMAC.
3. **Nunca** apuntar `dev` al backend de `prod`.
4. Rotar secrets trimestralmente como mínimo.
