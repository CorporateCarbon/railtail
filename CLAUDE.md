# CLAUDE.md

## Overview
railtail is a HTTP/TCP proxy for Railway workloads connecting to Tailscale nodes.
It listens on a local address and forwards traffic to a target Tailscale node
address, working around userspace-networking restrictions that otherwise force
containers to dial Tailscale nodes via a SOCKS5/HTTP proxy (confirmed: README.md).

This is a **Corporate Carbon Group fork** of an open-source project. The origin
remote is `github.com/CorporateCarbon/railtail`; `upstream` is
`github.com/half0wl/railtail` (confirmed: `git remote -v`). Note the Go module
path is still `github.com/half0wl/railtail` (confirmed: go.mod). Which CCG group
entity owns this is not evident from the repo (inferred: unknown — not stated
anywhere in-tree).

## Tech stack & languages
- Go 1.25.3 (confirmed: go.mod, Dockerfile)
- `tailscale.com/tsnet` v1.90.4 — embedded userspace Tailscale node (confirmed: go.mod, main.go)
- `golang.org/x/sync/errgroup`, `github.com/northbright/iocopy` — TCP bidirectional copy (confirmed: tcp.go)
- Stdlib `net/http/httputil.ReverseProxy` for HTTP mode (confirmed: http.go)

## Architecture / key components
- `main.go` — entry point. Loads config, starts a `tsnet.Server`, opens a local
  TCP listener on `[::]:<LISTEN_PORT>`, then branches:
  - **HTTP/HTTPS mode** if `TARGET_ADDR` has an `http(s)://` scheme — runs an
    `http.Server` proxying through the tsnet HTTP client (TLS verify is skipped:
    `InsecureSkipVerify: true`).
  - **TCP tunnel mode** otherwise — accepts connections and forwards each in a goroutine.
- `tcp.go` — `fwdTCP`: dials the target over tsnet, bidirectional copy with keepalives.
- `http.go` — `fwdHttp`: reverse-proxy a single request to the target.
- `internal/config` — struct-tag-driven config (flag/env/default). CLI flags
  override env vars. Validates `TARGET_ADDR` scheme/port.
- `internal/config/parser` — reflection-based flag/env parser.
- `internal/logger` — slog-based structured logging helpers.
- `internal/util` — includes `IsExpectedCopyError` (ignore benign copy errors).

## Configuration (confirmed: README.md, internal/config/config.go)
| Env | CLI flag | Notes |
| --- | --- | --- |
| `TARGET_ADDR` | `-target-addr` | Required. `host:port` = TCP mode; `http(s)://host:port` = HTTP mode |
| `LISTEN_PORT` | `-listen-port` | Required. Local listen port |
| `TS_HOSTNAME` | `-ts-hostname` | Required. Tailscale hostname |
| `TS_AUTH_KEY` / `TS_AUTHKEY` | N/A | Required. Env only |
| `TS_LOGIN_SERVER` | `-ts-login-server` | Optional. Control server URL (e.g. Headscale) |
| `TS_STATEDIR_PATH` | `-ts-state-dir` | Optional. Default `/tmp/railtail` |

## Build / run / test commands (confirmed)
```sh
go build -o railtail ./.        # local build (see Dockerfile: CGO_ENABLED=0, -ldflags="-w -s")
go run . -help                  # print usage/flags
docker build -t railtail .      # multi-stage build -> distroless/static image
```
No test files exist in the repo (no `*_test.go`), so there is no test command to run.
CI: `.github/workflows/ghcr-publish.yml` builds and pushes a Docker image to GHCR
on GitHub release `published` events.

## Key files & directories
- `main.go`, `tcp.go`, `http.go` — proxy core
- `internal/config/`, `internal/logger/`, `internal/util/` — support packages
- `Dockerfile` — distroless build; entrypoint `/usr/local/bin/railtail`
- `README.md` — usage, Railway deploy, RDS example

## Conventions / notes
- Designed to run as a separate Railway service reached over Railway's Private
  Network only. **Do not expose publicly** (README warning).
- HTTP mode skips TLS verification of the upstream target (main.go) — intentional
  for private-network targets; be aware when editing.
- Module path (`half0wl`) differs from the CCG origin; keep this in mind for
  import paths if renaming.
