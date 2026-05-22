# atlas

Go image serving and transformation service. Handles image requests, transformations (resize/format/quality/watermark), caching, and CDN invalidation.

Module: `atlas` (Go)

## Architecture

- **API** (`cmd/app/`) — HTTP image serving
- **Worker** (`cmd/worker/`) — consumes RabbitMQ queue for CDN cache invalidation

No database or Redis — filesystem-based caching only.

## Key packages

| Package | Purpose |
|---------|---------|
| `pkg/services/atlas/` | Core: `GetImage()`, `InvalidateUserCache()`, `InvalidateAssetCache()` |
| `pkg/cache/` | `AssetsCache`, `UsersCache` — key: RequestPath, Origin, CacheKey, Transformation, Webp |
| `pkg/transform/` | Image transformations (resize, format, quality, watermark) |
| `pkg/task-executor/` | Rate limiting + multiplexing for downloads/transforms |
| `pkg/endpoints/atlas/` | `GetImagePath`, `InvalidateUserCache` endpoints |
| `pkg/handlers/http/` | File streaming with cache headers, error redirect |
| `pkg/middlewares/` | Auth, tracing, forced_status, header caching |

## External integrations

- **RabbitMQ** — cache deletion event queue (viki-org/queue-client-go)
- **AWS CloudFront** — CDN invalidation
- **Datadog** — StatsD metrics + APM tracing

## Tests

```bash
make test
go test ./pkg/... -v -run TestName
```

Uses `testify` (assert/mock). Mock generation via `golang/mock`. Test fixtures in `sample-test-files/`.
