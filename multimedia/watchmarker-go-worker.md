# watchmarker-go-worker

Go worker that consumes watchmarker events from RabbitMQ and video events from the central queue; writes watch marker state to BigTable; publishes CSL events to Google Pub/Sub.

Module: `watchmarker-go-worker`

## Entry point

```bash
./worker run watchmarker      # consume watchmarker events from RabbitMQ
./worker run video            # consume video events from central queue
./worker run video_update     # consume video update events
./worker run csl-producer     # publish CSL events to Pub/Sub
./worker run test-event       # generate test events
```

Cobra CLI in `cmd/worker/main.go`.

## Key packages

| Package | Purpose |
|---------|---------|
| `pkg/services/consumers/watchmarker/` | RabbitMQ-based watchmarker event consumer |
| `pkg/services/consumers/video/` | Central queue video event consumer (create/update/delete) |
| `pkg/services/publishers/` | Event publishing to queue |
| `pkg/services/csl/` | CSL producer for Google Pub/Sub |
| `pkg/stores/watchmarker/` | BigTable watchmarker storage |
| `pkg/stores/csl/` | BigTable CSL storage |
| `pkg/stores/videos/` | Redis video cache with API fallback |
| `pkg/stores/core_content/` | Core content API client with Redis cache |

## Data stores

- **BigTable** — watch markers and CSL state (separate instance from watchmarker-service)
- **Redis** — video and core content cache
- **RabbitMQ** — watch marker event queue
- **Google Pub/Sub** — CSL event stream output
- **Viki API** — core content / video metadata (with caching layer)

## Tests

```bash
make test
go test ./pkg/... -v -run TestName
```

Generated mocks in `pkg/mocks/`. Worker pool concurrency configurable via `NumWatchmarkerWorkers`, `NumVideoWorkers`.
