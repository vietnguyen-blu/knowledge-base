# Multimedia DB Schema

DB name: `multimedia` (PostgreSQL).
Reconstructed from `database_migrations/multimedia/` migrations 001–078.
Tables owned by the `theia` role. Read access granted to `viki` role.

---

## master_videos

Primary source video registry. One row per encode source file (primary or dub/audio-only).

| Column | Type | Notes |
|--------|------|-------|
| id | SERIAL | PK |
| video_id | TEXT | FK to video (pattern `^\d+v$`) |
| url | TEXT | S3 source URL |
| file_size | BIGINT | |
| duration_in_s | TEXT | |
| thumbnail_url | TEXT | |
| file_format | TEXT | |
| drm | BOOLEAN | |
| nb_streams | INTEGER | |
| bitrate | INTEGER | |
| ffmpeg_format | TEXT | |
| timecode | TEXT | |
| trt | TEXT | |
| start_time | FLOAT | |
| end_time | FLOAT | |
| copyright_image | TEXT | |
| raw_analysis_id | TEXT | |
| meta | JSONB | |
| bumper_url | TEXT | |
| regions | TEXT[] | `{}` = WW, `{kcp}`, `{non_kcp}`, `{kcp,non_kcp}` |
| bumper_duration | FLOAT | |
| encodable_labels | TEXT[] | |
| source_present | BOOLEAN | |
| in_use | BOOLEAN | true = currently active/live |
| audio_only | BOOLEAN | true = dub/audio track encode |
| language | TEXT | language tag e.g. `en`, `ko` |
| master_group_id | VARCHAR | added in migration 077 |
| created_at | TIMESTAMPTZ | |
| updated_at | TIMESTAMPTZ | |

**Indexes:**
- `master_video_video_id_index` on `(video_id)`
- `master_video_unique_type_language_in_use` UNIQUE on `(video_id, audio_only, language, regions) WHERE in_use = true`
- `master_video_unique_type_in_use` UNIQUE on `(video_id, regions) WHERE in_use = true AND audio_only = false`

**Archived:** `master_videos_archived` (same columns + `action TEXT`)

---

## encode_jobs

Tracks individual encode job submissions to media-engine.

| Column | Type | Notes |
|--------|------|-------|
| id | SERIAL | PK |
| master_video_id | INTEGER | FK → master_videos |
| status | TEXT | created / encoding / completed / failed / cancelled / deleted / error / terminated |
| output_video_url | TEXT | nullable |
| reference_id | TEXT | nullable, media-engine reference |
| label | TEXT | encode label (resolution/profile combo) |
| thumbnail | TEXT | nullable |
| duration | INTEGER | nullable |
| created_time | TIMESTAMPTZ | nullable |
| download_start_time | TIMESTAMPTZ | nullable |
| encode_start_time | TIMESTAMPTZ | nullable |
| upload_start_time | TIMESTAMPTZ | nullable |
| completed_time | TIMESTAMPTZ | nullable |
| batch_id | TEXT | FK → encode_batches.batch_id |
| meta | JSONB | |
| encode_version | TEXT | |
| created_at | TIMESTAMPTZ | |
| updated_at | TIMESTAMPTZ | |

**Indexes:**
- `encode_jobs_batch_id_idx` on `(batch_id)`
- `encode_jobs_master_video_id_idx` on `(master_video_id)`
- `encode_jobs_reference_id_idx` on `(reference_id)`
- `unique_batch_label_idx` UNIQUE on `(batch_id, label)` — added migration 052

**Archived:** `encode_jobs_archived` (same columns + `action TEXT`)

---

## encode_batches

Groups encode_jobs for a single encode request. One batch per `master_video.encode()` call.

| Column | Type | Notes |
|--------|------|-------|
| id | BIGSERIAL | PK |
| batch_id | TEXT | UNIQUE, UUID string |
| master_video_id | INTEGER | FK → master_videos |
| status | TEXT | created / encoding / updating / ready_update / updated / failed / cancelled |
| encode_version | TEXT | added migration 063 |
| stream_replacement_action | TEXT | added migration 068 |
| meta | JSONB | full frontend payload (see meta shape below) |
| created_at | TIMESTAMPTZ | |
| updated_at | TIMESTAMPTZ | |

**meta shape** (set by CMS, passed through theia):
```json
{
  "admin_id": 1272,
  "priority": "library",
  "go_live": "immediately",
  "go_live_bool": true,
  "stream_update_option": {
    "title": "...",
    "value": "immediately",
    "stream_update": { "go_live": true, "manual": false, "force": false }
  },
  "streams_regions_state_change_action": null
}
```
**Read `go_live` from:** `meta.stream_update_option.stream_update.go_live` (dig path in Ruby) or `go_live_bool` (flat bool)

**Indexes:**
- `encode_batches_batch_id_index` on `(batch_id)`
- `encode_batches_master_video_id_index` on `(master_video_id)`

---

## encode_batch_groups

Groups one or more encode_batches (primary + dubs) for a single encode event.

| Column | Type | Notes |
|--------|------|-------|
| id | UUID | PK, `gen_random_uuid()` |
| video_id | VARCHAR | NOT NULL |
| encode_batch_ids | VARCHAR[] | NOT NULL, DEFAULT `{}` |
| status | VARCHAR | created / active / processing / failed / deprecated / cancelled |
| created_at | TIMESTAMPTZ | |
| updated_at | TIMESTAMPTZ | |

**Status meanings:**
- `processing` = encode in flight (running)
- `active` = currently live (streams pointing to this group)
- `cancelled` / `failed` / `deprecated` = terminal

**Index:** `encode_batch_groups_video_id_index` on `(video_id)`

---

## state_machine_executions

Tracks AWS Step Functions executions per encode label.

| Column | Type | Notes |
|--------|------|-------|
| execution_arn | TEXT | PK / unique identifier |
| state_machine_arn | TEXT | |
| state_machine_class | TEXT | |
| batch_id | TEXT | FK → encode_batches.batch_id |
| video_id | TEXT | |
| status | TEXT | running / succeeded / failed / timed_out / aborted |
| version | TEXT | |
| output | JSONB | nullable |
| created_at | TIMESTAMPTZ | |
| updated_at | TIMESTAMPTZ | |
| created_timestamp | TIMESTAMPTZ | |
| updated_timestamp | TIMESTAMPTZ | |

**Index:** `state_machine_executions_video_id_idx` on `(video_id)`

**Archived:** `state_machine_executions_archived` (same columns + `action TEXT`)

---

## media_engine_jobs

Tracks media-engine job submissions (SWF-based).

| Column | Type | Notes |
|--------|------|-------|
| id | SERIAL | PK |
| media_job_id | TEXT | media-engine job identifier |
| master_video_id | INTEGER | FK → master_videos |
| status | TEXT | |
| label | TEXT | |
| created_at | TIMESTAMPTZ | |
| updated_at | TIMESTAMPTZ | |

**Index:** `media_engine_jobs_media_job_id_idx` on `(media_job_id)` — migration 043

---

## streams

Live stream manifest records. Each row = one playable stream variant.

| Column | Type | Notes |
|--------|------|-------|
| id | SERIAL | PK |
| video_id | TEXT | NOT NULL, CHECK `~* '^\d+v$'` |
| resolution | TEXT | NOT NULL (240p / 360p / 480p / 540p / 720p / 1080p) |
| source | TEXT | NOT NULL, CHECK `kcp / viki / viki_kcp` |
| profile | TEXT | NOT NULL, CHECK `baseline / high` |
| url | TEXT | NOT NULL, CHECK `LIKE 'http%'` |
| type | TEXT | CHECK `normal / vtt` |
| format | TEXT | NOT NULL, CHECK `mp4 / m3u8 / mpd / wvc` |
| regions | TEXT[] | `{} / {kcp} / {non_kcp} / {kcp,non_kcp}` |
| bumper_duration | NUMERIC | |
| drmed | BOOLEAN | |
| cached | BOOLEAN | |
| start_time | FLOAT | DEFAULT 0 |
| manifest_version | TEXT | |
| duration | FLOAT | |
| encode_batch_id | TEXT | FK → encode_batches.batch_id |
| encode_batch_group_id | VARCHAR | FK → encode_batch_groups.id — added migration 077 |
| metadata | JSONB | |
| languages | TEXT[] | added migration 075 |
| created_at | TIMESTAMPTZ | |
| updated_at | TIMESTAMPTZ | |

**Unique constraint:** `(resolution, video_id, profile, type, format, source, regions, cached, drmed)`

**Index:** `streams_drmed_idx` on `(drmed)`

**Archived:** `archived_streams` (same columns + `action TEXT`)
- Index: `archived_streams_video_id_idx` on `(video_id)`

---

## stream_analysis

Per-label encode quality metrics.

| Column | Type | Notes |
|--------|------|-------|
| id | SERIAL | PK |
| video_id | TEXT | NOT NULL |
| resolution | TEXT | NOT NULL |
| profile | TEXT | NOT NULL |
| analysis | JSONB | |
| width | INTEGER | |
| height | INTEGER | |
| regions | TEXT[] | DEFAULT `{}` |
| encode_batch_id | TEXT | |
| encode_version | TEXT | |
| video_bitrate | NUMERIC | |
| audio_bitrate | NUMERIC | |
| video_codec | TEXT | |
| audio_codec | TEXT | |
| file_size_in_mb | NUMERIC | |
| created_at | TIMESTAMPTZ | |

**Index:** `stream_analysis_video_id_index` on `(video_id)`

---

## stream_replacement_conflicts

Configuration table for stream replacement action rules.

| Column | Type |
|--------|------|
| id | INTEGER PK |
| is_kcp_streams | BOOLEAN NOT NULL |
| is_viki_kcp_streams | BOOLEAN NOT NULL |
| is_viki_non_kcp_streams | BOOLEAN NOT NULL |
| is_viki_ww_streams | BOOLEAN NOT NULL |
| reason_type | VARCHAR(255) |
| reason_region | VARCHAR(255) |
| action | VARCHAR(255) |
| action_title | VARCHAR(255) |
| allow_multiple_encode | BOOLEAN NOT NULL |
| final_is_kcp_streams | BOOLEAN NOT NULL |
| final_is_viki_kcp_streams | BOOLEAN NOT NULL |
| final_is_viki_non_kcp_streams | BOOLEAN NOT NULL |
| final_is_viki_ww_streams | BOOLEAN NOT NULL |
| created_at / updated_at | TIMESTAMP NOT NULL |

---

## drm_packaging_info

DRM key metadata per video.

| Column | Type | Notes |
|--------|------|-------|
| id | BIGSERIAL | PK |
| video_id | TEXT | NOT NULL |
| key_id | TEXT | NOT NULL |
| media_id | TEXT | NOT NULL |
| track_types | TEXT[] | added migration 069 |
| content_id | TEXT | added migration 071 |
| created_at / updated_at | TIMESTAMPTZ | |

---

## preview_streams

Sprite/thumbnail preview streams for scrubbing.

| Column | Type | Notes |
|--------|------|-------|
| id | BIGSERIAL | PK |
| video_id | TEXT | NOT NULL |
| regions | TEXT[] | NOT NULL |
| iframe_url | TEXT | NOT NULL |
| vtt_url | TEXT | NOT NULL |
| thumbnail_url_template | TEXT | NOT NULL |
| tile_width / tile_height | INTEGER | |
| thumbnail_width / thumbnail_height | INTEGER | |
| interval_in_sec | INTEGER | |
| avg_bitrate | INTEGER | |
| created_at / updated_at | TIMESTAMPTZ | |

**Unique:** `(video_id, regions)`

---

## media_package_assets

AWS MediaPackage asset references for DRM packaging.

| Column | Type |
|--------|------|
| id | SERIAL PK |
| asset_id | TEXT NOT NULL |
| packaging_group | TEXT NOT NULL |
| package_config | TEXT NOT NULL |
| skd_asset_id | TEXT |
| video_id | TEXT NOT NULL |
| master_video_id | INTEGER NOT NULL |
| batch_id | TEXT NOT NULL |
| label | TEXT NOT NULL |
| in_use | BOOLEAN NOT NULL DEFAULT false |
| created_at / updated_at | TIMESTAMPTZ |

**Indexes:** `(video_id)`, `(asset_id)`

---

## video_meta

Video metadata cache (content state, on_air, licensed, holdbacks).

| Column | Type |
|--------|------|
| id | SERIAL PK |
| video_id | TEXT NOT NULL |
| video_type | TEXT NOT NULL |
| container_id | TEXT NOT NULL |
| on_air | BOOLEAN NOT NULL DEFAULT false |
| licensed | BOOLEAN NOT NULL DEFAULT false |
| valid_holdbacks | BOOLEAN NOT NULL DEFAULT false |
| duration | INTEGER |
| state | TEXT |
| created_at | TIMESTAMPTZ |

**Index:** `video_meta_video_id_index` on `(video_id)`

---

## Stream diff tables

### machined_stream_diffs
Auto-generated diffs between two master video encode outputs.

| Column | Type |
|--------|------|
| id | BIGSERIAL PK |
| encode_job_id | INTEGER NOT NULL REFERENCES encode_jobs UNIQUE |
| base_master_video_id | INTEGER NOT NULL |
| target_master_video_id | INTEGER NOT NULL |
| diff | JSONB NOT NULL |
| created_at / updated_at | TIMESTAMPTZ |

### draft_stream_diffs
Pending/in-progress stream diffs before promotion.

Same schema as `machined_stream_diffs`.

### released_stream_diffs
Final promoted stream diff per video.

| Column | Type |
|--------|------|
| id | BIGSERIAL PK |
| video_id | VARCHAR UNIQUE |
| diff | JSONB NOT NULL |
| created_at / updated_at | TIMESTAMPTZ |

---

## unused_streams

Orphaned stream CDN keys pending deletion.

| Column | Type |
|--------|------|
| id | SERIAL PK |
| key | TEXT NOT NULL UNIQUE |
| limelight_deleted_at | TIMESTAMPTZ |
| created_at / updated_at | TIMESTAMPTZ |

---

## Metrics tables (read-only dashboard data)

Populated by cron/reporting jobs. All have `created_at DATE`.

- `video_encode_metrics` — encode counts by video_type
- `video_encode_labels_metrics` — per-label encode stats
- `encode_time_metrics` — timing averages per label
- `encoded_streams_metrics` — stream quality averages
- `videos_state_metrics` — video resolution/state counts
- `streams_state_metrics` — stream format/state counts
- `encode_version_state_metrics` — encode version distribution
- `master_videos_state_metrics` — storage type counts
- `cdn_metrics` — CDN request/cost/cache-hit stats

---

## Dropped tables (no longer exist)
- `drm_keys` (dropped migration 062 — moved to manifest_service Redis)
- `drm_state_containers`, `drm_state_videos` (dropped migrations 060–061)
- Old `stream_diffs` (dropped migration 036)
- Various `*_archived` tables dropped in migration 067, recreated as leaner versions
