# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

This repo also has an `AGENTS.md` (root) plus scoped `AGENTS.md` files under `src/`
(`domains/`, `usecase/`, `validations/`, `ui/`, `infrastructure/whatsapp/`,
`infrastructure/chatstorage/`, `infrastructure/chatwoot/`,
`infrastructure/chatwoot/pgimport/`). Read the one(s) covering the area you touch —
they carry authoritative, up-to-date rules and are not fully duplicated here.

## What this is

GOWA is a Go WhatsApp HTTP API server built on `whatsmeow` (the multi-device
WhatsApp Web protocol library), Fiber v3, and SQLite. The Go module and all
runtime code live under `src/`. A single `rest` process serves the REST API,
an MCP server at `/mcp`, and a websocket endpoint, all sharing one device
manager and one set of usecases. The embedded web dashboard was moved out to
a separate repo (`aldinokemal/gowa-ui`); this server downloads, SHA-256-verifies,
and serves it from `storages/ui/`.

## Commands

All Go commands run from `src/`:

```sh
go test ./path/to/affected/package/...   # scoped test run (prefer this)
go test ./...                            # full suite — shared contracts/startup/broad changes
go vet ./...
go build -o whatsapp .
go run . rest                            # start the server (authorized runtime check only)
```

- Default SQLite build uses CGO (`mattn/go-sqlite3`). Add `-tags purego` to build/test
  against `modernc.org/sqlite` instead when that variant is relevant to the change.
- Run `go mod tidy` after changing dependencies.
- There is no test/lint CI workflow in `.github/workflows/` — only Docker image build,
  release, and latest-tag workflows. Local `go test` / `go vet` are the validation gate.
- Direct runs read/write `src/storages` and `src/statics`; Docker Compose mounts
  root-level `storages/`/`statics/` into `/app` instead. Keep both paths excluded from
  hot reload in `src/.air.toml`.

## Architecture

Layering (see root `AGENTS.md` "Cross-layer contracts" for the enforced rules):

```
ui/{rest,mcp,websocket}  → transport parsing only, delegates to usecases
usecase/                  → orchestration: validate → resolve device/client → WhatsApp + storage ops → domain response
domains/                  → DTOs and interfaces only (no logic)
validations/              → ozzo-based request validation, called from usecases
infrastructure/whatsapp/  → whatsmeow client/device lifecycle, events, JIDs, webhooks
infrastructure/chatstorage/ → SQLite repository (chats/messages/etc.)
infrastructure/chatwoot/  → optional Chatwoot bridge (live sync + history import)
```

Key cross-cutting rules to know before editing:

- **Device scoping is load-bearing.** After login, storage/chat code must use
  `client.Store.ID.ToNonAD().String()` as `device_id` — never the device manager's
  registry alias. An explicit device context must never silently fall back to a
  global/default client.
- **Repository interface changes are three-file changes**: `src/domains/chatstorage/interfaces.go`,
  `src/infrastructure/chatstorage/sqlite_repository.go`, and
  `src/infrastructure/whatsapp/chatstorage_wrapper.go` must move together.
- **JSON/form field stability**: DTOs in `domains/` are shared by REST, MCP, views,
  and docs — renaming/removing a field is a breaking, cross-transport change.
- **MCP tools are consolidated**, not 1:1 with REST routes: `whatsapp_send`,
  `whatsapp_message`, `whatsapp_chat`, `whatsapp_group`, `whatsapp_app` dispatch
  internally on a `type`/`action` argument.
- **Config is package globals in `src/config/settings.go`**, populated in
  `src/cmd/root.go` with precedence: CLI flags > env/Viper > `.env` defaults.
  `AppVersion` there is a source constant — release builds don't inject it via ldflags.
- Startup wiring (`src/cmd/helpers.go`, `root.go`, `rest.go`) constructs one instance
  of every usecase/repo/device-manager and threads it into both REST and MCP.

For anything touching a specific area (send/message flows, Chatwoot routing,
webhook payloads, MCP auth, chat storage schema/migrations, presence/JID handling),
go to that area's `AGENTS.md` first — they encode non-obvious invariants (e.g.
`GetMessageByIDAndDevice` vs `GetMessageByIDChatAndDevice`, migration append-only
rules, Chatwoot idempotency keys) that aren't restated here.

## Working conventions

- Preserve unrelated working-tree changes; keep `.env`, SQLite DBs, session data,
  QR codes, generated media, and history dumps out of commits.
- Tests that mutate config, package globals, or scheduler state must stay serial
  and restore state afterward — use existing colocated test helpers/stubs/fakes
  rather than inventing new patterns.
- Running `go run . rest` against saved sessions can reconnect real WhatsApp
  devices — only do this within the user's authorized scope.
