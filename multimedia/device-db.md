# device-db

Go service for device capability tracking and safety net attestation. HTTP API + background worker.

Module: `device-db` (Go 1.22.7)

## Entry points

- `cmd/server` — HTTP API
- `cmd/devicedb` — CLI background worker
- `deviceactivity/` — separate Go module; Pub/Sub consumer → BigTable writer for device activity events

## Key packages

| Package | Purpose |
|---------|---------|
| `pkg/endpoints/v5/` | `device_capability`, `safetynet` endpoints |
| `pkg/services/capabilities/` | Get/create device capabilities |
| `pkg/services/safetynet/` | Safety net attestation + nonce management |
| `pkg/services/play_integrity/` | Play Integrity API |
| `pkg/services/override/` / `priority/` | Override resolution |
| `pkg/stores/` | GORM wrappers for all DB tables |
| `pkg/parsers/v1/` | Device capability parsers (android, ios, roku, tv, web) |
| `pkg/models/db/` | GORM structs (`v2_` prefix) |

## API endpoints

- `GET /v5/devices/:device_id/capabilities.json` — fetch device capabilities
- `POST /v5/capabilities/:device_id/dump.json` — store capability dump
- `POST /v5/safetynet/:device_id/nonce.json` — create safety net nonce
- `GET /v5/liveness.json` / `readiness.json`

## Data stores

- **PostgreSQL** (GORM, read/write replicas) — tables: `v2_devices`, `v2_capabilities`, `v2_device_capabilities`, `v2_device_types`, `v2_capabilities_dumps`, `v2_capabilities_stats`, `v2_device_level_player_overrides`, `v2_model_level_player_overrides`, `v2_dead_letters`
- **Redis** — nonce cache, key: `safetynet-{uuid}`
- **Google Pub/Sub** — capability dump publishing
- **BigTable** — device activity storage (via `deviceactivity/` worker)

## Schema migrations

`database_migrations/devicedb/` — 72+ Sequel migrations.

## Tests

```bash
make test
go test ./... -count=1 -covermode=count
```

`gomock` + `testify/assert`. Mocks via `//go:generate mockgen`. Redis mocked with `redigomock`.
