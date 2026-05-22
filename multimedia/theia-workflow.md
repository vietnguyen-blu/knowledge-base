# theia-workflow

Go AWS Step Functions worker service. ~21 activity workers that poll SFN for task tokens and process encode/stream pipeline steps.

Module: `theia-workflow` (Go 1.22)

## Entry point

```bash
make build        # produces ./worker binary
./worker <cmd>
```

Commands in `cmd/workers/commands/` (Cobra CLI):
- `update-streams` — mark master videos in_use, push streams to API
- `update-db` — write encode results to DB
- `request-media-engine` — submit job to media-engine
- `check-media-engine-job-status` — poll media-engine job status
- `create-manifests-and-upload` — build and upload HLS/DASH manifests
- `check-label-encodable` — validate label can encode
- `queue-consumer` — RabbitMQ consumer for SFN triggers
- `notify-and-send-metrics`, `alert-failures`, `sme-info`, and more

## Key packages

| Package | Purpose |
|---------|---------|
| `pkg/services/update-streams/` | Mark master videos in_use, push streams |
| `pkg/services/encode-jobs/` | Create/update/query encode_jobs |
| `pkg/services/encode-batch-group/` | Batch group state management |
| `pkg/services/streams/` | Stream CRUD via streams API |
| `pkg/services/manifest/` | HLS/DASH manifest generation |
| `pkg/services/media-engine/` | Media engine job orchestration |
| `pkg/services/sm-executions/` | Trigger Step Functions executions |
| `pkg/services/update-db/` | Write encode results to DB |
| `pkg/services/stream-analysis/` | Write stream analysis records |
| `pkg/services/metrics/` | StatsD metrics |
| `pkg/services/slack/` | Slack notifications |

## Key models (`pkg/models/`)

- `MasterVideo` — gorm → `master_videos`
- `EncodeJob` — gorm → `encode_jobs`; statuses: `created/encoding/completed/failed/cancelled/deleted/error/terminated`
- `StateMachineExecution` — gorm → `state_machine_executions`
- `SmExecData` — Step Functions input payload (MasterVideoID, VideoID, BatchID, BatchGroupID, GoLive, AudioOnly, Meta, Label, EnableDrmEncode, etc.)
- `SmExecMeta` — meta sub-struct with GoLiveBool, StreamUpdateOption

## Data layer

Stores in `pkg/stores/` — interface + GORM implementation per entity:
`encode-jobs/`, `master-streams/`, `encode-batches/`, `raw-analysis/`, `stream-analysis/`, `images/`, `waveforms/`

## Config

`config/config.go` — loaded from Vault + Consul via `vaultconsul.Decode()`.
Local dev: `local.config` (`key====value` format).

## Tests

```bash
make test
go test ./pkg/services/<name>/... -v -run TestName
```

Table-driven tests with `gomock` (mocks in `pkg/mocks/`). `sqlmock` for DB layer. `vikilog.NewTesting()` for logger.

## DB schema reference

See `db-schema/CLAUDE.md`.
