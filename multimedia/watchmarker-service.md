# watchmarker-service

Mixed Go + Ruby service that manages user watch markers (playback position). Go binary serves the HTTP API; Ruby handles Rake tasks and seeding.

Module: `watchmarker-service`

## Architecture

- **Go HTTP API** (`pkg/`) — watch marker read/write, CSL (content state list), GDPR
- **Ruby** (`lib/`, `config.ru`) — Rake tasks, seeding, initializers

## API endpoints

**v4:**
- `GET /v4/watch_markers.json` — fetch watch markers
- `GET /v4/users/:user_id/watch_markers.json` — user's markers
- `DELETE /v4/users/:user_id/watch_markers.json` — delete user's markers
- `GET|DELETE /v4/watch_markers/privacy_info.json` — GDPR

**v5:**
- `GET /v5/watch_markers.json` — fetch watch markers
- `GET /v5/csl/:user_id/views.json` — CSL views for user

## Key packages

| Package | Purpose |
|---------|---------|
| `pkg/stores/watchmarker/` | BigTable-backed watch marker storage |
| `pkg/stores/csl/` | BigTable-backed CSL storage |
| `pkg/stores/video/` | Redis-backed video metadata cache |
| `pkg/services/analytics/` | Watch marker analytics and recording |
| `pkg/services/csl/` | Content state list service |
| `pkg/endpoints/v4/` `v5/` | Endpoint definitions |
| `pkg/middlewares/auth/` | Authentication middleware |

## Data stores

- **BigTable** (Google Cloud) — primary persistent store for watch markers and CSL (`watchmarkers`, `csl` tables)
- **Redis** (primary + secondary) — video metadata cache, event streams

## Tests

```bash
make test
go test ./pkg/... -v -run TestName
```
