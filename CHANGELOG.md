# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added
- Signals: temporarily log chord metadata extraction at WARNING level to diagnose
  callback linking in deployed Celery workflows.

### Changed
- Releases no longer build or publish Docker images; they publish only the Python
  package and GitHub Release artifacts.

## [0.1.3] - 2026-08-03

### Fixed
- Redis transport: `consume()` no longer crashes the server's background
  consumer thread on a transient Redis connection/timeout error (e.g. a
  dropped connection through a proxy/NAT). It now logs and retries after a
  short delay, matching the RabbitMQ transport's resilience. Previously any
  network blip permanently stopped event consumption until a manual restart.
- Signals: `task_sent` (the `PENDING` event) now carries `parent_id`/`root_id`
  from the message headers, like every other lifecycle event. Without it, a
  task's very first event created its graph node with no parent (added to
  `root_ids`), and a chord callback in particular would end up attached to
  whichever header task happened to trigger it (an implementation detail of
  Celery's chord-unlock mechanism) instead of the intended container.
- Graph: a CHORD callback is now always re-parented under the CHORD
  container, even if its own event already carries a different `parent_id`.
  Celery auto-assigns `parent_id` to whichever header task's execution
  context dispatches the callback; that incidental link was previously left
  in place, splitting the callback (and everything chained after it) into
  its own disconnected root graph instead of continuing the chain.

## [0.1.2] - 2026-08-03

### Fixed
- Graph: re-parent a GROUP/CHORD member that joins after the container node
  already exists (i.e. the 3rd+ member). Previously only members present at
  container-creation time were detached from their real upstream parent; a
  later member kept its original `parent_id` while also being added as a
  group child, leaving it listed under both — breaking the chain visually.

## [0.1.1] - 2026-08-03

### Fixed
- Graph: preserve parent links when multiple chords occur in one chain.

## [0.1.0] - 2026-08-03

This is the first release of **`stemtrace-pydantic-v1`**, an unofficial fork of
[stemtrace](https://github.com/iansokolskyi/stemtrace) v0.3.4 published under a new PyPI
name because it downgrades a core dependency. Version numbering restarts from 0.1.0 for
this fork; see [stemtrace's changelog](https://github.com/iansokolskyi/stemtrace/blob/main/CHANGELOG.md)
for history prior to the fork.

### Changed
- **Breaking:** downgraded `pydantic` from `>=2.0.0` to `>=1.10.15,<2.0.0`. All models
  translated to pydantic v1 APIs (`class Config` instead of `model_config`/`ConfigDict`,
  `.dict()`/`.json()`/`.parse_obj()` instead of `.model_dump()`/`.model_dump_json()`/
  `.model_validate()`). Frozen-model mutation now raises `TypeError` instead of
  `ValidationError` (pydantic v1 behavior), and `str` fields coerce scalars (e.g. `int`)
  instead of rejecting them (pydantic v1 is more lenient than v2 here).
- **Breaking:** capped `fastapi` to `<0.125.0` — 0.125.0 unconditionally imports
  `pydantic.TypeAdapter` (a pydantic v2-only symbol), which breaks under pydantic v1.
- **Breaking:** dropped Python 3.14 support (`requires-python` is now `>=3.10,<3.14`).
  FastAPI hard-disables pydantic v1 support on Python 3.14+ regardless of FastAPI
  version, so the server component cannot import under 3.14 with pydantic v1 installed.
- Removed `bump-my-version` from the `dev` extra — it requires `pydantic>=2.0.0` and
  made the dev dependency set unresolvable alongside the pydantic v1 pin. Run it via
  `uvx bump-my-version` instead if needed.

## [0.3.4] - 2026-06-25

### Fixed
- WebSocket: deny unauthenticated connections during the handshake instead of accepting and then closing them. The accept-then-close path half-opened the connection, so when an unauthenticated client (e.g. a port scanner) aborted mid-handshake, uvicorn's legacy `websockets` implementation ran `close()` on a protocol whose `transfer_data_task` was never set, raising `AttributeError: 'WebSocketProtocol' object has no attribute 'transfer_data_task'`. Closing before `accept()` rejects the handshake (HTTP 403) without opening the protocol object.
- Form login: build the login form action and redirects as relative paths instead of absolute URLs, and derive the session cookie's `Secure` flag from `X-Forwarded-Proto`. Behind a TLS-terminating proxy that doesn't forward the scheme to the ASGI scope (e.g. uvicorn without `--forwarded-allow-ips`), absolute URLs were downgraded to `http://`, so the browser posted the login form over http, the proxy answered `301 -> https`, the POST body was dropped, and login silently failed (with a mixed-content warning). stemtrace now works behind such a proxy without requiring forwarded-headers configuration.

## [0.3.3] - 2026-03-20

### Added
- Node alias from arguments: derive graph node display name from task args/kwargs via `node_alias_from_arguments` config option (thanks @ohadmata for the idea — #46, #48)
- Docker: multi-architecture image builds (amd64 + arm64) (thanks @ohadmata — #51)

### Fixed
- Graph: order task nodes before synthetic container nodes (GROUP/CHORD) for correct rendering (thanks @codyburrito — #52)

## [0.3.2] - 2026-02-28

### Fixed
- Redis transport: normalize `ssl_cert_reqs` URL parameter from `CERT_REQUIRED`/`CERT_NONE`/`CERT_OPTIONAL` to lowercase values expected by redis-py

## [0.3.1] - 2026-01-04

### Added
- UI: optional built-in form login flow for protecting the stemtrace UI

### Changed
- Config: split server broker URL from worker event transport URL (`broker_url` vs `transport_url`)
- API: avoid blocking worker/registry refresh operations
- Docs: clarify optional configuration and retention behavior

### Fixed
- Workers/Registry: refresh via Celery inspect to reduce stale worker/task registry data

## [0.3.0] - 2026-01-04

### Added
- RabbitMQ event transport

### Changed
- Tests: improved RabbitMQ transport coverage and reduced duplication

### Fixed
- UI: harden derived prefix sanitization
- CI/E2E: avoid artifact name conflicts

## [0.2.2] - 2026-01-03

### Added
- README: add release version badge

### Changed
- Packaging/docs: simplify badges and metadata
- README: pin PyPI badge to release version
- Tests: make version assertion dynamic

### Fixed
- Version bump: sync fix

## [0.2.1] - 2026-01-03

### Added
- Task Registry now includes task definition metadata from workers (module, signature, docstring, bound)

### Changed
- README: updated PyPI badge styling
- CI: upload Codecov coverage on `main`

## [0.2.0] - 2026-01-03

### Added
- Workers API endpoints: `GET /api/workers` and `GET /api/workers/{hostname}`
- Worker lifecycle tracking via `worker_ready` / `worker_shutdown` events to maintain a live worker registry
- Workers UI page for inspecting online/offline workers and their registered tasks
- Task Registry now shows which workers registered each task (`registered_by`)

## [0.1.1] - 2025-12-31

### Added
- Python 3.14 support in CI and project classifiers
- Mock-based E2E testing mode (no Docker required for local dev)
- Codecov integration for coverage reporting

### Changed
- Upgraded React and React-DOM to v19
- Upgraded @tanstack/react-query to latest
- Updated GitHub Actions to v6 (checkout, setup-python, setup-node, upload/download-artifact)
- Improved README with clearer Quick Start and FastAPI embedding docs

## [0.1.0] - 2025-12-27

### Added
- **Core domain models**: `TaskEvent`, `TaskState`, `TaskNode`, `TaskGraph`
- **Protocol definitions**: `EventTransport`, `TaskRepository`, `AsyncEventConsumer`
- **Event transports**: Redis Streams (`RedisTransport`), in-memory (`MemoryTransport`)
- **Celery signal integration**: Automatic event capture via `stemtrace.init_worker(app)`
- **Server components**:
  - `GraphStore` — Thread-safe in-memory graph storage with LRU eviction
  - `EventConsumer` / `AsyncEventConsumer` — Background event processing
  - `WebSocketManager` — Real-time event broadcasting
- **REST API**: `/api/tasks`, `/api/graphs`, `/api/health` endpoints
- **FastAPI integration**:
  - `StemtraceExtension` — Full extension with lifespan management
  - `create_router()` — Minimal router for custom setups
  - Auth helpers: `require_basic_auth`, `require_api_key`, `no_auth`
- **React UI**: Task list, graph visualization (react-flow), timeline view
- **CLI commands**: `stemtrace server`, `stemtrace consume`
- **Docker support**: Multi-stage Dockerfile, docker-compose.yml for local dev
- **E2E test suite**: Docker API tests + Playwright browser tests
- **Comprehensive test suite**: 350+ Python tests, 90%+ coverage

[unreleased]: https://github.com/iansokolskyi/stemtrace/compare/v0.3.0...HEAD
[0.3.0]: https://github.com/iansokolskyi/stemtrace/compare/v0.2.2...v0.3.0
[0.2.2]: https://github.com/iansokolskyi/stemtrace/compare/v0.2.1...v0.2.2
[0.2.1]: https://github.com/iansokolskyi/stemtrace/compare/v0.2.0...v0.2.1
[0.2.0]: https://github.com/iansokolskyi/stemtrace/compare/v0.1.1...v0.2.0
[0.1.1]: https://github.com/iansokolskyi/stemtrace/compare/v0.1.0...v0.1.1
[0.1.0]: https://github.com/iansokolskyi/stemtrace/releases/tag/v0.1.0
