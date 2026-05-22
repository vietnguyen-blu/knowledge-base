# knowledge-base

Claude Code context files for the viki-org monorepo.

## Structure

```
knowledge-base/
└── multimedia/          # multimedia platform services
    ├── db-schema.md     # full multimedia PostgreSQL schema (migrations 001–078)
    ├── theia.md
    ├── theia-workflow.md
    ├── media-engine.md
    ├── playback-streams.md
    ├── streams.md
    ├── watchmarker-service.md
    ├── watchmarker-go-worker.md
    └── manifest-service.md
```

## How it works

Each file is symlinked as `CLAUDE.md` in the corresponding service directory:

```
theia/CLAUDE.md -> ../knowledge-base/multimedia/theia.md
```

Claude Code auto-loads `CLAUDE.md` from the working directory, so running claude inside any service picks up its knowledge file automatically.

## Editing

Edit files directly here — symlinks pick up changes instantly. Do not edit the symlinks themselves.
