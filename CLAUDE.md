# CLAUDE.md

## What is this?

jackal is a high-performance XMPP server written in Go (1.19). See [README.md](README.md) for full architecture, project structure, and deployment docs.

## Quick Reference

```bash
make build          # Compile jackal (CGO_ENABLED=0 static binary)
make test           # Run tests: go test -timeout 1m --race ./...
make check          # Run all checks: fmt + vet + lint
make generate       # Regenerate moq mocks (go generate ./...)
make proto          # Regenerate protobuf Go code
make install        # Install jackal to $GOPATH/bin
make installctl     # Install jackalctl CLI to $GOPATH/bin
```

## Code Conventions

- **Imports:** Three groups — stdlib, external deps, internal (`github.com/ortuman/jackal/...`) — separated by blank lines
- **Context:** `context.Context` is always the first parameter
- **Errors:** Sentinel errors in `error.go` files, formatted `"<package>: <description>"`. Check with `errors.Is()`, never direct comparison
- **Logging:** `github.com/go-kit/log` with `level.Info(logger).Log("msg", "...", "key", val)`. Use `kitlog.NewNopLogger()` in tests
- **Interfaces:** Major components defined as interfaces in `interface.go` files, with private aliases to scope dependencies within packages
- **Constructors:** Accept interfaces, return unexported concrete types

## Testing Conventions

- **Naming:** `Test<TypeName>_<MethodName>`
- **Structure:** Given-When-Then with `// given`, `// when`, `// then` comments
- **Assertions:** `github.com/stretchr/testify/require` (not `assert`)
- **Mocks:** Generated with `moq` via `//go:generate` directives. Output: `<name>.mock_test.go`. Set `Mock.FuncNameFunc = func(...) { ... }` and verify with `Mock.FuncNameCalls()`
- **After interface changes:** Run `make generate` to regenerate mocks

## Key Architecture Decisions

- **Storage:** Decorator chain — `Measured (Prometheus) → Cached (Redis) → Base (PostgreSQL/BoltDB)`. Repository composes 10 sub-interfaces (User, Roster, Archive, etc.)
- **Routing:** `hosts.IsLocalHost()` → C2S router (local + cluster via gRPC) or S2S router (federation)
- **Modules:** Implement `Module` interface; IQ handlers also implement `IQProcessor`. Enabled via YAML config
- **Hooks:** Priority-based event system (`pkg/hook`) for decoupled module communication
- **Clustering:** etcd for distributed state, gRPC for inter-node stanza routing
- **C2S lifecycle:** State machine — Connecting → Connected → Authenticating → Authenticated → Bound → Disconnected → Terminated
