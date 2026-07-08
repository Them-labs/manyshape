The platform is the hosted half of Manyshape (and the part that's [priced](/#pricing)); this repo ships an MIT reference implementation you can self-host. All endpoints are JSON over HTTP with CORS enabled. Authenticated endpoints take `Authorization: Bearer <token>`.

## Auth

| Endpoint | Body | Notes |
|---|---|---|
| `POST /api/auth/signup` | `{ email, password }` | Password ≥ 8 chars, scrypt-hashed. Returns `{ userId, email, token, expiresAt }` |
| `POST /api/auth/login` | `{ email, password }` | Same reply. 404 no account · 401 wrong password |
| `POST /api/auth/logout` | - (auth) | Revokes the current session server-side |
| `GET /api/me` | - (auth) | `{ userId, email, createdAt, activeSessions }` |

Sessions last 30 days (MongoDB TTL index reaps expired ones). Clients should drop their stored token on any 401.

## Sessions

| Endpoint | Notes |
|---|---|
| `GET /api/sessions` (auth) | Active sessions: `{ id, createdAt, expiresAt, userAgent, current }` |
| `DELETE /api/sessions/:id` (auth) | Revoke one device |
| `DELETE /api/sessions` (auth) | Sign out everywhere |

## Registry

| Endpoint | Notes |
|---|---|
| `POST /api/registry/contracts` | Publish `{ contract, origins, contractUrl, capBase }`. Contract is validated; upserted by `id@version` |
| `GET /api/registry/contracts` | List published contracts (metadata) |
| `GET /api/registry/resolve?origin=…` | Find the contract published for a page origin - how the extension discovers apps |

The reference vendor app publishes itself at boot. In the hosted platform, publishing is where identity/signing attaches (see [pricing](/#pricing)).

## Experiences

The user-owned artifact: intent history + the compiled surface, upserted per `(user, app, name)`.

| Endpoint | Notes |
|---|---|
| `GET /api/experiences?app=…` (auth) | List yours (without surface bodies) |
| `GET /api/experiences/:id` (auth) | Full record including `surface` |
| `POST /api/experiences` (auth) | `{ app, name, framework, intents, surface }` - upsert |
| `DELETE /api/experiences/:id` (auth) | Delete |

## Hosted agent

`POST /api/agent` (auth) with `{ contract, referenceSurface, currentSurface?, intents }` → `{ surface, source }`. The platform runs the same `@manyshape/agent` pipeline as a self-hosted vendor: generate → transpile JSX if `framework: react` → server-side gate (header, static policy scan). Clients still run the full client-side gate before activating.

Model selection on the reference implementation: `GEMINI_API_KEY`/`GOOGLE_API_KEY` (model `GEMINI_MODEL`, default `gemini-2.5-pro`), else Anthropic credentials (`claude-opus-4-8`), else 503.
