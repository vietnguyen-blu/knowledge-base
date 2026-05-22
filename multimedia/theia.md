# theia

Rails encoding orchestration service. Manages master videos, encode batches, Step Functions executions, DRM state, and stream lifecycle. DB: `multimedia` (PostgreSQL).

## Architecture

- **HTTP API** (Puma): `app/controllers/v4/`
- **Worker** (rake task): `lib/tasks/step_functions/run_theia_workflow_consumer.rake` — consumes RabbitMQ `:encode_flow` messages via `TheiaWorkflowQueueRouter`

## Key files

| File | Purpose |
|------|---------|
| `app/controllers/v4/master_videos_controller.rb` | master video lifecycle (encode, cancel, streams) |
| `app/controllers/v4/encode_batch_controller.rb` | batch status, stream updates |
| `app/controllers/v4/streams_controller.rb` | stream CRUD |
| `app/models/master_video.rb` | `encode()` — creates batch + group, queues SFN |
| `app/models/encode_batch.rb` | AASM state machine |
| `app/models/encode_batch_group.rb` | group status constants |
| `lib/services/master_video_service.rb` | multi-master encode orchestration |
| `lib/services/encode_batch_group_service.rb` | group creation and queries |
| `lib/services/encode_batch_status_update_service.rb` | AASM state transitions |
| `lib/services/stream_publishing_service.rb` | triggers post-process publish |
| `lib/services/trigger_state_machine_execution_service.rb` | AWS SFN execution trigger |
| `lib/services/stream_service.rb` | stream deletion and management |
| `lib/services/statsd_service.rb` | metrics wrapper (prefix: `theia-workflow`) |
| `config/initializers/stats.rb` | STATSD global (Datadog StatsD) |

## EncodeBatch AASM states

```
created → encoding → updating → updated
                  → ready_update → updating
                  → failed
       → cancelled
```
CANCELLABLE_STATUSES = `[created, encoding, ready_update]`

## EncodeBatchGroup statuses

`created` → `processing` (encode running) → `active` (live) | `cancelled` | `failed` | `deprecated`

Running statuses: `[created, processing]`

## encode_batch.meta shape

Set by CMS on encode trigger, stored as JSONB:
```json
{
  "admin_id": 1272,
  "priority": "library",
  "go_live_bool": true,
  "stream_update_option": {
    "value": "immediately",
    "stream_update": { "go_live": true, "manual": false, "force": false }
  },
  "streams_regions_state_change_action": null
}
```
Read go_live: `meta.with_indifferent_access.dig(:stream_update_option, :stream_update, :go_live).to_bool`

## Regions

- `[]` / `{}` = worldwide
- `['kcp']` = KCP only
- `['non_kcp']` = non-KCP only
- `['kcp', 'non_kcp']` = both

## Audio-only / dub encodes

`master_video.audio_only = true` for dub tracks. Treated separately from primary in batch group logic.

## Metrics

`STATSD.increment('theia.<metric>')` — direct calls.
`StatsdService.increment('<metric>')` — prefixed as `theia-workflow.<metric>`.

## Tests

```bash
make rspec
bundle exec rspec spec/path/to/file_spec.rb
```

Factories in `spec/factories/`. Pattern: `create(:encode_batch, batch_id: 'b1', meta: { ... })`

## DB schema reference

See `db-schema/CLAUDE.md`.
