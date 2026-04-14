# AGENTS.md

This file provides guidance to AI coding agents when working with code in this repository.

## Build & Test Commands

```sh
# Build
go build -trimpath -ldflags '-s -w' ./cmd/shadowsocks-go
go build -trimpath -ldflags '-s -w' ./cmd/shadowsocks-go-domain-set-converter

# Windows build (requires special flags for tfo-go)
go build -trimpath -ldflags '-s -w -checklinkname=0' -tags tfogo_checklinkname0 ./cmd/shadowsocks-go

# Test all packages
go test -v ./...

# Test a single package
go test -v ./domainset/
go test -v ./ss2022/

# Vet
go vet ./...
```

## Architecture

This is a proxy platform implementing Shadowsocks 2022 and other protocols. The two binaries are `shadowsocks-go` (proxy server/client) and `shadowsocks-go-domain-set-converter` (domain set format tool).

**Entry flow:** `cmd/shadowsocks-go/main.go` loads JSON config → creates `service.Config` → calls `Config.Manager()` → `Manager.Run(ctx)`.

### Key packages

- **service/** — Service manager orchestrating servers, clients, router, DNS, and API. `Config` is the top-level config struct with `Servers[]`, `Clients[]`, `ClientGroups[]`, `DNS[]`, `Router`, `API`.
- **ss2022/** — Shadowsocks 2022 protocol: AEAD ciphers, TCP/UDP relay, salt pool, credential store, padding/reject policies.
- **socks5/**, **httpproxy/**, **direct/**, **ssnone/** — Other protocol implementations.
- **conn/** — Low-level connection utilities (~57 files). Heavily platform-specific with OS-optimized paths (Linux splice, recvmmsg/sendmmsg; per-OS socket options). Files use `_linux.go`, `_windows.go`, `_unix.go`, `_stub.go` suffixes.
- **router/** — Traffic routing engine matching on domain sets, IP prefix sets, and GeoIP.
- **domainset/** — Domain matching (exact, suffix, keyword, regexp). Supports plaintext, gob, and v2fly/dlc formats.
- **prefixset/** — IP prefix matching using BART trie.
- **netio/** — Network I/O abstractions (`StreamClient`, `StreamServer` interfaces).
- **zerocopy/** — Zero-copy packet/stream interfaces used by relay implementations.
- **jsoncfg/** — JSON config file loading/saving.
- **logging/** — Zap logger initialization with multiple presets.

### Core abstractions

- `netio.StreamClient` / `netio.StreamServer` — TCP stream relay interfaces.
- `zerocopy.UDPClient` — UDP relay with zero-copy packet handling.
- Servers and clients are configured via `service.ServerConfig` and `service.ClientConfig`, which support multiple protocols through a `protocol` field.

### Platform-specific code

The `conn/` package has extensive platform-specific implementations. Linux gets splice(2) for TCP, recvmmsg(2)/sendmmsg(2) for UDP batching, and TPROXY support. Build tags and file suffixes control compilation.

## Conventions

- Go module requires **Go 1.26+**.
- JSON config uses `json:"fieldName,omitzero"` tags.
- Structured logging with `go.uber.org/zap` throughout. Services implement `ZapField()` for log context.
- Commit messages use emoji prefixes and `package: description` format (e.g., `🍧 conn: expose more PMTUD options`).
- SIGUSR1 triggers config reload on Unix (uPSK store hot-reload).
- Configuration examples live in `docs/`.
