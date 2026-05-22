# theia

Rails encoding orchestration service. Manages master videos, encode batches, Step Functions executions, DRM state, and stream lifecycle. DB: `multimedia` (PostgreSQL).

## Architecture

- **HTTP API** (Puma): `app/controllers/v4/`
- **Worker** (rake task): `lib/tasks/step_functions/run_theia_workflow_consumer.rake` — consumes RabbitMQ `:encode_flow` messages, routes to `TheiaWorkflowQueueRouter`

## Key files

| File | Purpose |
|------|---------|
| `app/controllers/v4/master_videos_controller.rb` | encode, cancel_encode, signed_streams |
| `app/controllers/v4/encode_batch_controller.rb` | update_streams, streams_revert |
| `app/models/master_video.rb` | `encode()` — creates batch, batch group, queues SFN |
| `app/models/encode_batch.rb` | AASM state machine |
| `app/models/encode_batch_group.rb` | status constants |
| `lib/services/cancel_encode_service.rb` | cancel flow entry point |
| `lib/services/stream_publishing_service.rb` | triggers `publish_batch_group_streams` post-process |
| `lib/services/encode_batch_group_service.rb` | group creation and queries |
| `lib/services/encode_batch_status_update_service.rb` | AASM state transitions |
| `lib/services/trigger_state_machine_execution_service.rb` | AWS SFN execution trigger |
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

Running statuses (group still in flight): `[created, processing]`

## encode_batch.meta shape

Set by CMS on encode trigger, stored as JSONB:
```json
{
  "admin_id": 1272,
  "priority": "library",
  "go_live": "immediately",
  "go_live_bool": true,
  "stream_update_option": {
    "value": "immediately",
    "stream_update": { "go_live": true, "manual": false, "force": false }
  },
  "streams_regions_state_change_action": null
}
```
**Read go_live as:** `meta.with_indifferent_access.dig(:stream_update_option, :stream_update, :go_live).to_bool`

## Metrics

Direct `STATSD.increment('theia.<metric>')` for raw calls.
`StatsdService.increment('cancel_encode.<metric>')` for service-level metrics (prefix becomes `theia-workflow.cancel_encode.*`).

Key cancel_encode metrics: `no_running_group`, `completed`, `batch_cancelled.primary`, `batch_cancelled.audio_only`, `batch_cancelled.co_batch`, `group_cancelled`, `stream_revert_triggered`, `streams_deleted`

## Regions

- `[]` / `{}` = worldwide
- `['kcp']` = KCP region only
- `['non_kcp']` = non-KCP only
- `['kcp', 'non_kcp']` = both

## Audio-only / dub encodes

`master_video.audio_only = true` for dub tracks. Cancel flow for audio-only removes the batch from the running group rather than cancelling the whole group (unless it was the last batch still encoding).

## Tests

```bash
bundle exec rspec spec/path/to/file_spec.rb
make rspec  # full suite
```

Factory pattern: `create(:encode_batch, batch_id: 'b1', meta: { stream_update_option: { stream_update: { go_live: true } } })`

## DB schema reference

See `db-schema/CLAUDE.md` for full multimedia schema.
