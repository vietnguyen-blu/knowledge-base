# watchlist

Go service — source of truth for user watchlists. 3 sections per user. HTTP API + RabbitMQ worker.

Module: `watchlist`

## Entry points

- `main.go` — HTTP API server (Gorilla Mux)
- `rabbit_consumer/main.go` — RabbitMQ consumer worker (`watchlist-worker`)

## Key packages

| Package | Purpose |
|---------|---------|
| `applications/watchlist/` | `Service` (HTTP), `WorkerService` (RabbitMQ consumer) |
| `applications/watchlist/datastore/` | `WatchlistStore` (PostgreSQL/GORM), `WatchmarkerStore` (HTTP to watchmarker-service) |

## Key models

- `WatchlistRecord` — UserID, VideoID, ContainerID, ContainerType, SectionID, timestamps
- `VideoWatchMarker` — WatchlistRecord + watchMarker, duration, creditMarker
- `WatchlistPayload` — RabbitMQ message shape

## API endpoints

- `GET /v4/internal/users/:user_id/watchlist.json` — fetch watchlist (params: section_id, page, per_page, with_watchmarkers)
- `DELETE /v4/internal/users/:user_id/watchlist.json` — delete containers from section
- `GET /v4/healthcheck.json` / `/v4/ping.json`

## Data stores

- **PostgreSQL** (GORM, reader/writer split) — `watchlist_records` table
- **RabbitMQ** — consumes `watchlist_worker_events` queue (dead-letter exchange)
- **Redis** (master/slave) — dedup cache, key: `r:{userId}:{videoId}`, TTL 1h
- **watchmarker-service** — external HTTP call to enrich with watch position

## Tests

```bash
make test
go test ./... -v -run TestName
```

`gomock` + `go-sqlmock` + `testify/assert`. Mocks generated via `//go:generate mockgen`.
