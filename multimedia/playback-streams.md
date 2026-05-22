# playback-streams

Go edge service that serves streams to video players. Two processes:
- **API** (`cmd/app/`) — HTTP endpoints for stream fetch, DRM, manifests
- **Worker** (`cmd/worker/`) — consumes queue events (RabbitMQ + Google Pub/Sub), populates Redis cache

Module: `playback-streams` (Go 1.22)

## API endpoints (`pkg/endpoints/v5/`)

- `GET /v5/playback_streams` — main stream fetch with DRM/manifest (primary endpoint)
- `GET /v5/playback_capability` — device capability check
- `GET /v5/drm` — DRM license endpoint

## Key packages

| Package | Purpose |
|---------|---------|
| `pkg/services/playback_stream/` | Core stream fetch + manifest generation |
| `pkg/services/playback_stream/cdn_signer/` | CDN URL signing |
| `pkg/services/playback_stream/manifest/` | HLS/DASH manifest building |
| `pkg/services/drm/` | DRM key/license delivery |
| `pkg/services/device_db/` | Device capability lookups |
| `pkg/services/blocking/` | Geo/business rule blocking |
| `pkg/services/monetisation/` | TVOD/SVOD entitlement |
| `pkg/services/featureflags/` | Per-request feature flag evaluation |
| `pkg/services/queue/` | Queue event consumer |

## Redis stores (`pkg/stores/`)

All data served from Redis populated by the worker. No direct DB reads in API path.

- `streams` — stream metadata by video_id
- `videos` — video metadata cache
- `tvod_privileges` — TVOD entitlement cache
- `bumpers` — bumper content cache
- `applications` — app/client config
- `devices` — device whitelist/blacklist
- `holdbacks` — content holdback rules
- `concurrent_stream_redis` — active concurrent stream tracking
- `devicedb_cache` — device capability cache

## Key models (`pkg/models/`)

- `StreamInfo` — video_id, url, drm, format, region, encode_version, CDN info
- `DeviceCapability` — device resolution/codec support
- `ConcurrentStream` — user concurrent stream state
- `UserDetails` — user playback context

## Worker

`cmd/worker/` — consumes RabbitMQ (`rabbit_consumer.go`) and Google Pub/Sub (`pubsub_consumer.go`) events to keep Redis state current. Worker writes; API reads.

## Tests

```bash
make test
go test ./pkg/some/package/... -v -run TestName
```
