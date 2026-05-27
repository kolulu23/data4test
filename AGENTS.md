# AGENTS.md - Data4Test (盾测)

## Project Overview

Full-stack automated testing platform: Go backend (GoAdmin + Gin) + Vue 2/TypeScript frontend. Module name: `data4test` (Go 1.23).

## Commands

```bash
# Backend dev server (requires config.json and MySQL)
go run .

# Frontend dev (watched rebuild)
yarn install && yarn dev

# Frontend production build (webpack → statik embed into Go binary)
yarn build         # runs build:web then build:go in sequence

# Cross-platform Go binary build (outputs to ./deploy/)
make build

# Run all Go tests (currently zero test files in repo)
make testrun       # go test -v ./...

# Generate ORM table models from database
adm generate -c adm.ini   # requires `adm` CLI from GoAdmin

# Docker quick start (MySQL + app)
docker-compose up -d
# Default access: http://127.0.0.1:9088, login admin/admin
```

## Architecture

```
main.go             Entry point, all HTTP routes live here (Gin router)
biz/                Core business logic (65 files) — the real application code
tables/             ORM table definitions (32 files, mix of auto-generated and hand-written)
pages/              Dashboard page renderers
models/             Thin DB connection layer (base.go only)
login/              Custom login component (has its own Makefile for asset compilation)
web/                Vue 2 + TypeScript frontend source (ts/, vue/, types/)
mgmt/               Runtime data directory — NOT source code:
  mgmt/sql/         Database init/update SQL scripts
  mgmt/doc/         System docs, design specs, change log, release notes
  mgmt/data/        Test data file storage
  mgmt/upload/      File upload storage
  mgmt/log/         Runtime logs (auto-generated)
  mgmt/case/        Test case files (.xmind)
  mgmt/common/      Shared template files
```

## Prerequisites & Setup

1. **MySQL** must be running and configured in `config.json`
2. **`config.json` must exist** at repo root — the `init()` function panics on startup if missing
3. Initialize database: import `mgmt/sql/init.sql` (schema + seed data), then apply `mgmt/sql/update.sql` for incremental changes
4. Server binds to port **9088**, always runs in `gin.ReleaseMode` (stdout discarded)
5. Frontend build is **two-step**: webpack compiles TS/Vue → `web/static/js/`, then `statik` bundles it into Go source at `statik/`. The `statik` Go tool must be installed separately.

## Key Gotchas

- **Package replace**: `github.com/GoAdminGroup/filemanager` is replaced with a fork `github.com/JosingCai/filemanager` in go.mod — do not revert this.
- **`bootstrap.go`** is referenced in `config.json` (`"bootstrap_file_path"`) but is essentially empty (1 line `package main`). The real bootstrap happens via `init()` in `main.go`.
- **No test files exist** — `make testrun` succeeds because it finds nothing to test, not because tests pass.
- **Docker DB image** (`dbDockerfile`) concatenates `init_all_20250702.sql` into `init.sql` at build time and discards `update.sql`. For local dev, you need to run `update.sql` separately after importing `init.sql`.
- **Login redirect logic** (in `main.go`): role-based routing — `Administrator`/`Operator` → schedule page, `ApiManage` → Postman-like console. Defined by `redirect_path` in `config.json`.
- **Webpack 3** (legacy): the webpack config deletes `web/static/js` on every build and deletes `statik/` in production mode.
- **Makefile `test` target** references `black-box-test` and `user-acceptance-test` as dependencies, but neither target is defined — they silently do nothing.
- **Session timeout config**: uses `//go:linkname` to reach GoAdmin's private `config._global` for runtime updates. Do not remove the `_ "unsafe"` import or the linkname directive.

## Session Timeout API

**`GET/POST /admin/api/session_config`** (Administrator only, behind GoAdmin auth):

| Field | Type | Range | Meaning |
|---|---|---|---|
| `session_life_time` | int | 0–315360000 | Seconds. 0 = never expire (mapped to ~10yr internally) |

```bash
# Query current value
curl -b cookies.txt http://127.0.0.1:9088/admin/api/session_config

# Set to 24 hours
curl -b cookies.txt -X POST -H 'Content-Type: application/json' \
  -d '{"session_life_time": 86400}' http://127.0.0.1:9088/admin/api/session_config

# Disable timeout (never expire)
curl -b cookies.txt -X POST -H 'Content-Type: application/json' \
  -d '{"session_life_time": 0}' http://127.0.0.1:9088/admin/api/session_config
```

Changes take effect immediately and persist to `config.json`. The `domain` field in config.json controls the cookie Domain attribute (leave empty for exact-host matching).

## Development Conventions

When making changes (per `mgmt/doc/file/development/must_know.md`):
- Record all changes in `mgmt/doc/file/update/change_log.md` with date, type tag (`[Bug]`/`[Optimize]`/`[Feature]`), and description
- Append any SQL schema changes to `mgmt/sql/update.sql` with date comment
- Document new features in the relevant files under `mgmt/doc/file/`
- Self-verify before merging; mark unverified or temporary changes in the change log

## CI Scripts

- `ci/getInitSQL.sh` — mysqldump-based script for extracting init SQL from a running DB (used to produce `mgmt/sql/init.sql`)
- `ci/gitInitPackage.sh` — assembles the deploy package
