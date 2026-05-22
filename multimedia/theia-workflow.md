# theia-workflow

Go AWS Step Functions worker service. Implements ~21 activity workers that poll SFN for task tokens, process encode/stream pipeline steps, and write results back to the multimedia DB and streams API.

Module: `theia-workflow` (Go 1.22)

## Entry point

```bash
make build        # produces ./worker binary
./worker <cmd>    # e.g. ./worker update-streams
```

Commands in `cmd/workers/commands/` (Cobra CLI):
- `update-streams` — push stream updates after encode completes
- `update-db` — write encode results to DB
- `request-media-engine` — submit job to media-engine
- `check-media-engine-job-status` — poll media-engine job
- `create-manifests-and-upload` — build and upload HLS/DASH manifests
- `check-label-encodable` — validate label can encode
- `queue-consumer` — RabbitMQ consumer for SFN triggers
- `notify-and-send-metrics`, `alert-failures`, `sme-info`, and more

## Key packages

| Package | Purpose |
|---------|---------|
| `pkg/services/update-streams/` | Mark master videos in_use, push streams to API |
| `pkg/services/encode-jobs/` | Create/update/query encode_jobs rows |
| `pkg/services/encode-batch-group/` | Batch group state management |
| `pkg/services/streams/` | Stream CRUD via streams service API |
| `pkg/services/manifest/` | HLS/DASH manifest generation |
| `pkg/services/media-engine/` | Media engine job orchestration |
| `pkg/services/sm-executions/` | Trigger Step Functions executions |
| `pkg/services/update-db/` | Write encode results to DB |
| `pkg/services/stream-analysis/` | Write stream analysis records |
| `pkg/services/metrics/` | StatsD metrics emission |
| `pkg/services/slack/` | Slack notifications |

## Key models (`pkg/models/`)

**`MasterVideo`** — gorm mapped to `master_videos` table (see db-schema/CLAUDE.md)

**`EncodeJob`** — gorm mapped to `encode_jobs`; statuses: `created/encoding/completed/failed/cancelled/deleted/error/terminated`

**`StateMachineExecution`** — gorm mapped to `state_machine_executions`

**`SmExecData`** — Step Functions input payload shape:
```go
type SmExecMeta struct {
    AdminID         int                `json:"admin_id"`
    Priority        string             `json:"priority"`
    GoLive          string             `json:"go_live"`
    GoLiveBool      bool               `json:"go_live_bool"`
    StreamUpdateOptn StreamUpdateOption `json:"stream_update_option"`
}
type StreamUpdateOption struct {
    StreamUpdate struct {
        GoLive bool `json:"go_live"`
        Manual bool `json:"manual"`
        Force  bool `json:"force"`
    } `json:"stream_update"`
}
```

**`UpdateStreamsWorkerInput`** key fields: `MasterVideoID`, `VideoID`, `BatchID`, `BatchGroupID`, `GoLive`, `AudioOnly`, `IsPostProcess`

## Data layer

Stores in `pkg/stores/` — each has an interface + GORM implementation:
- `encode-jobs/`, `master-streams/`, `encode-batches/`, `raw-analysis/`, `stream-analysis/`, `images/`, `waveforms/`

Pattern: `store.FindEncodeBatchByBatchID(ctx, batchID)`, `store.Update(ctx, id, map[string]interface{}{...})`

## Config

`config/config.go` — loaded from Vault + Consul via `vaultconsul.Decode()`.
Local dev: `local.config` with `key====value` format.

Key config fields: DB host/name/user/password, OceanusDB (secondary), Redis URL, AWS keys/buckets, MediaEngine host, StatsD host, AppID/Domain/Secret.

## Test patterns

```bash
make test                                    # go test ./...
go test ./pkg/services/update-streams/... -v -run TestName
```

- Table-driven tests with `gomock` (generated mocks in `pkg/mocks/`)
- `sqlmock` for DB layer tests
- `vikilog.NewTesting()` for logger
- Shared `ctrl := gomock.NewController(t)` at outer test level; `ctrl.Finish()` deferred once
- `BatchID` in `updateStreamsParams` must be set when testing post-process to trigger the skip-self logic in batch group loop

## DB schema reference

See `db-schema/CLAUDE.md` for full multimedia schema.
