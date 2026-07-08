Everything in the repo self-hosts: the landing site is static, the platform and the demo vendor app are plain Node processes, and state lives in MongoDB.

## Processes and ports

| Process | Port | Needs |
|---|---|---|
| Landing + docs site | static files (`npm run build` → `_site/`) | any static host / nginx |
| Platform (`platform/server.js`) | 8600 | MongoDB, model key (optional) |
| Demo vendor app (`mvp/server.js`) | 8484 | MongoDB, model key (optional) |

## Configuration

Environment (or `platform/.env` / `mvp/.env`):

| Variable | Default | Meaning |
|---|---|---|
| `MONGODB_URI` | `mongodb://localhost:27017` | One server, two databases: `manyshape-platform`, `manyshape-mail` |
| `GEMINI_API_KEY` / `GOOGLE_API_KEY` | - | Enables live generation (Gemini) |
| `GEMINI_MODEL` | `gemini-2.5-pro` | Use `gemini-2.5-flash` on free-tier keys |
| `ANTHROPIC_API_KEY` | - | Fallback provider (`claude-opus-4-8`) |
| `PLATFORM_PORT` / `PORT` | 8600 / 8484 | Listen ports |
| `PLATFORM_URL` | `http://localhost:8600` | Where the vendor app publishes its contract |
| `RESEED` | - | `1` resets the demo mailboxes |

## A single-VM layout (what our own deploy uses)

One small VM runs everything behind nginx:

- `manyshape.com` → static `_site/`
- `platform.manyshape.com` → proxy to `127.0.0.1:8600`
- `demo.manyshape.com` → proxy to `127.0.0.1:8484`

Both Node processes run under systemd; MongoDB binds to localhost only. See the deploy scripts for the provisioning script (AWS free-tier EC2 + Cloudflare in front).

> **Before exposing a self-hosted platform to the internet**, read the [security model](./security.md) - the reference implementation's auth has no email verification, the demo app's user identity is a header, and Cloudflare "Flexible" TLS leaves origin traffic unencrypted. Fine for demos; harden before real users.
