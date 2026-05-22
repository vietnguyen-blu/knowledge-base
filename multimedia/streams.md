# streams

Go CRUD service that owns the authoritative `streams` and `waveforms` tables in the multimedia PostgreSQL DB. Provides REST API used by theia-workflow to create/read/update/delete stream records.

Module: `github.com/viki-org/streams` (Go 1.20)

## API endpoints (`pkg/endpoints/`)

- `GET/POST/PUT/DELETE /streams` — stream CRUD
- `GET /streams?video_id=` — list streams by video_id
- `GET/POST/PUT/DELETE /waveforms` — waveform CRUD
- `GET /healthcheck`

## Key packages

| Package | Purpose |
|---------|---------|
| `pkg/services/streams/` | Stream CRUD business logic |
| `pkg/services/waveforms/` | Waveform CRUD |
| `pkg/datastores/streams/` | PostgreSQL queries (Reform ORM) |
| `pkg/datastores/waveforms/` | Waveform DB queries |
| `pkg/datastores/countries/` | Region/country lookups |
| `pkg/entities/` | Reform-mapped DB structs |
| `pkg/transport/http/` | go-kit HTTP transport (Gorilla mux) |

## ORM

Uses **Reform** (`gopkg.in/reform.v1`) — codegen-based, entities in `pkg/entities/`, generated files suffixed `_reform.go`.

## Key entities

**`Stream`** (reform:streams) — see `db-schema/CLAUDE.md` for full column list. Key fields: `VideoID`, `Resolution`, `Profile`, `Format`, `Source`, `Regions`, `URL`, `Drmed`, `EncodeBatchID`, `EncodeBatchGroupID`, `Languages`

**`ArchivedStream`** — same columns, written by DB trigger on stream update/delete.

## Tests

```bash
make test
go test ./pkg/some/package/... -v -run TestName
```

Uses `go-sqlmock` for DB layer tests. Test fixtures in `pkg/test/`.

## DB schema reference

See `db-schema/CLAUDE.md` — `streams`, `archived_streams`, `waveforms` tables.
