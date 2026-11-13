# Changelog

All notable changes to `web`, the Gin-style web framework for Caju. The format follows
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/) and the project uses
[Semantic Versioning](https://semver.org/). Each GitHub release takes its notes from the section of
its version here.

## [Unreleased]

## [0.2.1] - 2026-11-13

### Changed
- The package moved to the cajulang organization: its name is now `github.com/cajulang/web`
  (`import "github.com/cajulang/web"`, `caju pkg add github.com/cajulang/web`).
- Links point to the language repository at github.com/cajulang/cajulang.

## [0.2.0] - 2026-11-11

### Added
- `run_workers(addr, n)`: several processes on the same address (Caju 0.3.0's
  `listen_and_serve_workers`).
- `client_ip()` from the address of the connection.

### Changed
- Requires Caju 0.3.0 (keep-alive and the epoll event loop of `std/http`).
- A faster router: no splitting of the path, the handler chain stored per route and parameters
  created only when a route has them.

## [0.1.0] - 2026-11-11

### Added
- First release: an engine with routes per method (`get`, `post`, `put`, `patch`, `delete`...),
  path parameters (`:id`) and wildcards, groups, middleware (`use`, `next`, `abort`), a `Context`
  with `param`, `query`, `bind` (JSON), `json` and `set`/`get`, `no_route`, `default_engine()` with
  logging and recovery, and `test_request` for tests without a server.

[Unreleased]: https://github.com/cajulang/web/compare/v0.2.1...HEAD
[0.2.1]: https://github.com/cajulang/web/compare/v0.2.0...v0.2.1
[0.2.0]: https://github.com/cajulang/web/compare/v0.1.0...v0.2.0
[0.1.0]: https://github.com/cajulang/web/releases/tag/v0.1.0
