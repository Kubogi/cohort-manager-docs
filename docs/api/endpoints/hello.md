# GET /api/hello

**Endpoint**: `GET /api/hello`

**Authentication**: ❌ **Public** (no Bearer token required)

**Roles**: anyone

**Source**: [routes/index.js:24](../../../backend/src/routes/index.js#L24)

## Description

Smoke-test endpoint. Used to confirm the API is reachable through Nginx without needing a token. Returns a fixed UTF-8 string.

## Request

```
GET /api/hello
```

No headers, no body required.

## Response

### 200 OK

```
Content-Type: text/html; charset=utf-8

Ciallo ~ (∠・ω< )⌒★
```

Note: not JSON. Bare `text/html` (Express's default for `res.send(string)`).

## Use cases

- **Production sanity check**: `curl https://your-domain/api/hello` to confirm the reverse proxy + backend are alive.
- **Liveness probe**: a load balancer or container orchestrator can hit this for `200 OK`. It doesn't touch Mongo, so it's cheap.
- **Debugging mixed-content / CORS**: easiest endpoint to ping from the browser console.

## Notes

- The response body is an easter egg; do not parse it in clients.
- This is the only endpoint that bypasses `authGuard` other than `POST /api/auth/login`. Take care not to leak any sensitive content here in the future.
