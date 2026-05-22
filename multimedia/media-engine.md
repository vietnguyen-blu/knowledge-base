# media-engine

Rails distributed transcoding pipeline. Orchestrates video encoding, analysis, DRM key generation, thumbnail/waveform generation via AWS Simple Workflow Service (SWF) on EC2 Spot Fleet instances.

## Architecture

- **HTTP API** (Puma): receives job requests from theia, returns status
- **SWF Workers** (rake tasks in `lib/tasks/`): poll SWF for activity tasks, execute transcode steps

## Key models

| Model | Purpose |
|-------|---------|
| `MediaJob` | Top-level job — groups multiple tasks for one encode request |
| `MediaTask` | Individual processing task (encode/analyse/thumbnail/waveform/validate) |
| `MediaTaskActivity` | Timing and lifecycle events per task |
| `WorkflowExecution` | SWF workflow state and execution history |
| `SpotFleetRequest` | EC2 spot fleet lifecycle |
| `InputAnalysis` / `OutputAnalysis` | Media quality analysis results |
| `FirstPassEncodeAnalysis` | Multi-pass encode analysis |
| `VideoDiff` | Video comparison / quality diff |
| `StorageTransitionJob` | Media file storage lifecycle (S3 → Glacier) |

## Key services

| Service | Purpose |
|---------|---------|
| `SwfService` | SWF workflow orchestration, job queuing |
| `FleetService` | EC2 spot fleet start/stop/scale |
| `CapacityService` | Task capacity planning |
| `MediaTaskEtaService` | ETA prediction for encode tasks |
| `StorageTransitionService` | S3 → Glacier transitions |
| `KeyosService` | DRM key management |
| `StatsdService` | Metrics |

## SWF activity workers

Located in `lib/activities/`:
- `BaseMediaTaskActivity` — base encoder
- `AnalyseMediaTaskActivity` — ffprobe analysis
- `EncodeActivity` — ffmpeg transcode
- `GenerateImagesActivity` — thumbnails
- `GenerateWaveformActivity` — audio waveform
- `GeneratePreviewActivity` — preview sprites
- `ValidateAndUploadMediaActivity` — output validation + S3 upload
- `VideoDiffActivity` — quality comparison
- `DbTasksActivity` — DB operations

Workflow config in `lib/workflows/workflow_config.rb` — maps tasks to SWF workflow/activity names.

## Tests

```bash
make rspec
bundle exec rspec spec/path/to/file_spec.rb
```

Factories in `spec/factories/`. Integration transcode tests in `spec/transcode/`.
