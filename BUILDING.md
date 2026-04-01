# Building jackal from Source

This guide walks through all steps required to build jackal from source, including installing prerequisites and resolving common compilation errors.

## Prerequisites

- **Go 1.19+** — [https://go.dev/dl](https://go.dev/dl)
- **make**
- **Git**

Verify your Go installation:

```bash
go version    # should print go1.19 or later
```

## Step 1 — Clone the repository

```bash
git clone https://github.com/ortuman/jackal.git
cd jackal
```

## Step 2 — Install required Go tools

These tools are not in `go.mod` and must be installed globally before any `make` target can succeed.

### moq — mock code generator

Used by `make generate` to produce `*.mock_test.go` files. Without it, every package that uses generated mocks will fail to compile during tests.

```bash
go install github.com/matryer/moq@latest
```

### goimports — import formatter (required by `make check` / `make fmt`)

```bash
go install golang.org/x/tools/cmd/goimports@latest
```

### golint — style linter (required by `make check` / `make lint`)

```bash
go install golang.org/x/lint/golint@latest
```

After installing tools, ensure `$GOPATH/bin` is on your `$PATH`:

```bash
export PATH="$PATH:$(go env GOPATH)/bin"
```

Add the line above to your shell profile (`~/.zshrc`, `~/.bashrc`, etc.) to make it permanent.

## Step 3 — Download Go module dependencies

```bash
go mod download
```

Or let the build do it automatically — `go mod tidy` is invoked by `make` when `go.sum` is out of date.

## Step 4 — Generate mock files

All packages use `moq`-generated mocks for testing. These files are **not** committed to the repository and must be generated before building or running tests:

```bash
make generate
```

This runs `go generate ./...`, which processes every `//go:generate moq ...` directive across all packages and produces `*.mock_test.go` files alongside the source.

> **Note:** If you see errors like `undefined: transportMock` or `undefined: repositoryMock` in `_test.go` files, it means `make generate` has not been run (or `moq` is not installed).

## Step 5 — Build the binary

```bash
make build
```

This compiles a static binary (`CGO_ENABLED=0`) with `netgo` tags:

``` bash
./jackal
```

To also build the `jackalctl` CLI:

```bash
make jackalctl
```

To install both binaries to `$GOPATH/bin`:

```bash
make install      # installs jackal
make installctl   # installs jackalctl
```

## Step 6 — Run tests

```bash
make test
```

This runs `make generate` first, then executes the full test suite with race detection and coverage:

```bash
go test -timeout 1m --race -coverprofile=coverage.txt -covermode=atomic ./...
```

## Step 7 — Run all checks (optional)

```bash
make check
```

Runs `generate` + `fmt` + `vet` + `lint` in sequence.

---

## Regenerating Protobuf code (optional)

Only needed if you modify `.proto` files under `proto/`. Requires `protoc` to be installed separately ([https://grpc.io/docs/protoc-installation](https://grpc.io/docs/protoc-installation)).

```bash
make proto
```

---

## Quick reference

| Command | Description |
| --- | --- |
| `make build` | Compile `jackal` binary |
| `make generate` | Regenerate all `moq` mocks |
| `make test` | Generate mocks + run all tests with race detection |
| `make check` | generate + fmt + vet + lint |
| `make install` | Install `jackal` to `$GOPATH/bin` |
| `make installctl` | Install `jackalctl` to `$GOPATH/bin` |
| `make proto` | Regenerate protobuf Go code |

## Common errors and fixes

| Error | Cause | Fix |
| --- | --- | --- |
| `undefined: transportMock` (or any `*Mock` type) | Mock files not generated | `go install github.com/matryer/moq@latest` then `make generate` |
| `make generate` exits with code 2 | `moq` binary not found in `$PATH` | Install `moq` and ensure `$GOPATH/bin` is in `$PATH` |
| `make check` fails on `fmt` | `goimports` not installed | `go install golang.org/x/tools/cmd/goimports@latest` |
| `make check` fails on `lint` | `golint` not installed | `go install golang.org/x/lint/golint@latest` |
| `make proto` fails | `protoc` not installed | Install via [grpc.io/docs/protoc-installation](https://grpc.io/docs/protoc-installation) |
| `go: updates to go.sum needed` | Dependencies out of sync | `go mod tidy` |
