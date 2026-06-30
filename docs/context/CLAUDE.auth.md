# battlecaos-auth — Contexto para Claude

## ¿Qué hace este servicio?
Servicio de autenticación. Recibe el `idToken` de Google OAuth ya validado por el frontend
(Google Identity Services), lo verifica con `google-auth-library`, y emite un JWT propio
firmado con `JWT_SECRET` que el resto del sistema usa para identificar al jugador.

**Regla de oro:** este servicio no conoce nada de salas ni juego. Solo autenticación.

## Puerto y protocolo
- **HTTP:** `3001` (Express REST)
- No usa Socket.io
- No usa Redis Pub/Sub (solo `PING` para health check)

## Scaffolding completo

```
battlecaos-auth/
├── .env                    ← variables reales (no commitear)
├── .env.example            ← plantilla pública
├── .gitignore
├── package.json
├── CLAUDE.md
└── src/
    ├── index.js            ← Express: POST /auth/google + GET /health
    ├── redis.js            ← fábrica createRedis()
    ├── logger.js           ← logger con prefijo [auth]
    └── auth/
        ├── google.js       ← capa cliente: verifica idToken con Google
        └── jwt.js          ← capa servidor: firma/verifica JWT propios
```

**¿Por qué `src/auth/`?** `google.js` habla con servidores externos de Google. `jwt.js` firma
tokens internos del sistema. Separarlas permite testear cada capa independientemente.

## Variables de entorno

```env
# .env.example
REDIS_URL=redis://localhost:6379
AUTH_PORT=3001
JWT_SECRET=cambiar_en_produccion
GOOGLE_CLIENT_ID=tu-client-id.apps.googleusercontent.com
CLIENT_ORIGIN=http://localhost:5173
```

## package.json

```json
{
  "name": "battlecaos-auth",
  "version": "1.0.0",
  "type": "module",
  "scripts": {
    "start": "node src/index.js",
    "dev":   "nodemon src/index.js"
  },
  "dependencies": {
    "cors":                 "^2.8.5",
    "dotenv":               "^16.4.0",
    "express":              "^4.19.0",
    "google-auth-library":  "^9.9.0",
    "helmet":               "^7.1.0",
    "ioredis":              "^5.5.0",
    "jsonwebtoken":         "^9.0.0"
  },
  "devDependencies": {
    "nodemon": "^3.1.14"
  }
}
```

> `node --watch` falla silenciosamente en Node.js v22+ con ESM y top-level await.
> Se usa `nodemon` como reemplazo estable.
```

## Endpoints

### `POST /auth/google`
- **Body:** `{ "idToken": "<Google ID token>" }`
- **Respuesta OK (200):** `{ "token": "<JWT propio>" }`
- **Respuesta error (401):** `{ "error": "token_invalido", "detail": "..." }`

### `GET /health`
- **Respuesta OK (200):**
```json
{ "service": "auth", "status": "ok", "redis": "ok", "uptime": 42.1, "timestamp": 1234567890 }
```
- **Respuesta error (503):**
```json
{ "service": "auth", "status": "error", "redis": "down", "error": "connect ECONNREFUSED" }
```
- Hace `redis.ping()` antes de responder (DOMF1302). Incluye `redis`, `uptime` y `timestamp`.

## Payload del JWT emitido

```json
{
  "sub":     "google-uid-123",
  "name":    "Jugador Ejemplo",
  "picture": "https://lh3.googleusercontent.com/...",
  "iat":     1234567890,
  "exp":     1234654290
}
```

- Expiración: `24h`
- Firmado con: `process.env.JWT_SECRET`

## Sistema de logs

```js
// src/logger.js
const SVC = 'auth';
export const log = {
  info:  (...a) => console.log( `[${SVC}]`, ...a),
  warn:  (...a) => console.warn( `[${SVC}] WARN:`, ...a),
  error: (...a) => console.error(`[${SVC}] ERROR:`, ...a),
};
```

Uso esperado:
```
[auth] :3001
[auth] login OK — sub: google-uid-123
[auth] WARN: token_invalido — audience mismatch
[auth] ERROR: redis ping fail
```

## Testing
No tiene lógica de dominio compleja. Verificación manual:

```bash
# Con servidor arrancado:
curl -X POST http://localhost:3001/auth/google \
  -H "Content-Type: application/json" \
  -d '{"idToken": "<token real de Google>"}'
# → { "token": "eyJ..." }

curl http://localhost:3001/health
# → { "service": "auth", "status": "ok" }
```

Para test de integración real se necesita un `idToken` de Google válido (obtenido desde el frontend).

## Seguridad
- Usar `helmet()` en todos los middlewares
- CORS restringido a `CLIENT_ORIGIN`
- No loggear el `idToken` ni el JWT emitido (datos sensibles)
- `JWT_SECRET` mínimo 32 caracteres en producción

## Redis
Solo se usa para `ping()` en el health check. No hay Pub/Sub en este servicio.
Se crea una sola conexión `redis = createRedis()`.

## Tareas del Auth Service

| ID | Descripción | Sprint | Estado |
|---|---|---|---|
| DOMF001 | Init + Redis connect + GET /health base | 1 - Fase 0 | ✅ |
| DOMF101 | POST /auth/google — verificar Google token + emitir JWT | 1 - Fase 1 | ✅ |
| DOMF1302 | GET /health con `redis`, `uptime`, `timestamp` y 503 en error | 2 - Fase 3 | ✅ |

> Las tres tareas están integradas en un único `src/index.js` limpio.

## Arranque

```bash
cp .env.example .env   # rellenar GOOGLE_CLIENT_ID y JWT_SECRET
npm install
npm run dev
```
