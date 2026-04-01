# c2s — Client-to-Server Package

The `c2s` package implements the XMPP Client-to-Server (C2S) subsystem for jackal. It manages the full lifecycle of incoming client connections — from TCP socket acceptance through TLS negotiation, SASL authentication, resource binding, and stanza routing — and provides both local and cluster-aware stream routing.

## Architecture Overview

```
┌──────────────────────────────────────────────────────────────┐
│                      SocketListener                          │
│  Accepts TCP connections, creates transports & inC2S streams │
└──────────────┬───────────────────────────────────────────────┘
               │ spawns per connection
               ▼
┌──────────────────────────────────────────────────────────────┐
│                         inC2S                                │
│  State machine managing a single client stream               │
│                                                              │
│  Connecting → Connected → Authenticating → Authenticated     │
│      → Binded → Disconnected → Terminated                    │
└──────────────┬───────────────────────────────────────────────┘
               │ routes stanzas via
               ▼
┌──────────────────────────────────────────────────────────────┐
│                       c2sRouter                              │
│  Implements router.C2SRouter                                 │
│  Delegates to LocalRouter (same node) or ClusterRouter (gRPC)│
└──────────┬───────────────────┬───────────────────────────────┘
           │                   │
           ▼                   ▼
   ┌───────────────┐   ┌────────────────┐
   │  LocalRouter  │   │ ClusterRouter  │
   │  (in-process) │   │ (gRPC / etcd)  │
   └───────────────┘   └────────────────┘
```

## Components

### `SocketListener` — Connection Acceptor

**File:** `socket_listener.go`

Listens on a configurable TCP address/port and accepts incoming client connections. For each connection it:

1. Wraps the `net.Conn` in a `transport.SocketTransport` with connect/keep-alive timeouts.
2. Creates SASL authenticators based on the configured mechanisms (SCRAM-SHA-1, SCRAM-SHA-256, SCRAM-SHA-512, SCRAM-SHA3-512, and optionally an external gRPC authenticator).
3. Instantiates an `inC2S` stream and starts its read loop.

Supports **Direct TLS** (TLS on connect) via `tls.NewListener`.

Multiple listeners can be configured and created at once via `NewListeners()`.

### `inC2S` — Incoming C2S Stream (State Machine)

**File:** `in.go`

The core of the package. Each `inC2S` represents a single client-to-server XMPP stream and implements `stream.C2S`. It follows an explicit state machine:

| State | Description |
| --- | --- |
| `Connecting` | Initial state; stream header not yet received. |
| `Connected` | Stream opened; awaiting StartTLS or SASL auth. |
| `Authenticating` | SASL handshake in progress. |
| `Authenticated` | SASL succeeded; awaiting compression or resource bind. |
| `Binded` | Resource bound; full stanza processing active. |
| `Disconnected` | Stream closing; disconnect hooks running. |
| `Terminated` | Fully cleaned up; transport closed. |

**Key behaviors:**

- **Stream features negotiation:** Offers StartTLS (required for socket transports), SASL mechanisms (filtered by channel binding support), zlib compression, resource binding, and session establishment.
- **SASL authentication:** Delegates to pluggable `auth.Authenticator` implementations. Enforces limits on failed attempts (max 5) and aborts (max 1) before disconnecting with a policy violation.
- **Resource binding:** Handles client-requested and server-generated resource names. Enforces max session count via shapers. Supports three resource conflict strategies: `override` (generate new resource), `terminate_old` (disconnect existing), and `disallow` (reject binding).
- **Stanza processing:** After binding, routes `<iq>`, `<presence>`, and `<message>` stanzas. Component-addressed stanzas are forwarded to the component subsystem. Module IQs are delegated to the module subsystem. All other stanzas go through the global router.
- **Hook integration:** Fires hooks at every lifecycle stage (`C2SStreamConnected`, `C2SStreamElementReceived`, `C2SStreamBinded`, `C2SStreamDisconnected`, `C2SStreamTerminated`, etc.) enabling decoupled module reactions.
- **Concurrency:** Uses a `runqueue.RunQueue` to serialize all stream mutations, ensuring thread-safe state transitions without coarse-grained locking.
- **Rate limiting:** Applies per-JID rate limiters from the shaper subsystem on the transport's read path.
- **Graceful disconnect:** On connection timeout for a binded stream, waits up to 5 seconds for the peer to close before forcing termination.

### `LocalRouter` — Node-Local Stream Router

**File:** `local_router.go`

Manages all C2S streams on the current server instance. Maintains two collections:

- **`stms`** — Anonymous (pre-bind) streams, keyed by `stream.C2SID`.
- **`bndRes`** — Bound resources, keyed by username → `resources` (a per-user set of streams).

**Operations:**

| Method | Description |
| --- | --- |
| `Register(stm)` | Adds a stream to the anonymous map. |
| `Bind(id)` | Moves a stream from anonymous to the bound resources map. |
| `Unregister(stm)` | Removes a stream from both maps. |
| `Route(stanza, username, resource)` | Delivers a stanza to a specific bound resource. |
| `Disconnect(username, resource, streamErr)` | Disconnects a specific bound resource. |
| `Stream(username, resource)` | Looks up a bound stream. |
| `Start(ctx)` / `Stop(ctx)` | Lifecycle management; `Stop` gracefully disconnects all streams with `SystemShutdown`. |

Periodically reports total connection count as a Prometheus gauge.

### `c2sRouter` — C2S Router (Local + Cluster)

**File:** `router.go`

Implements `router.C2SRouter` and serves as the bridge between local and cluster routing. For each operation, it checks whether the target resource lives on the current instance (`instance.ID()`) and delegates accordingly:

- **Same node:** Delegates to `LocalRouter`.
- **Different node:** Delegates to `ClusterRouter` (gRPC-based inter-node communication).

**Stanza routing logic** (bare JID vs. full JID):

- **Full JID:** Routes to the exact matching resource. Returns `ErrResourceNotFound` if absent.
- **Bare JID + Message:** Routes to the highest-priority resource(s). Resources with negative priority are skipped. If no positive-priority resource is available, returns `ErrUserNotAvailable`.
- **Bare JID + other stanza types:** Broadcasts to all available resources.

### `flags` — Stream State Flags

**File:** `flags.go`

Thread-safe bitfield tracking stream state: `Secured`, `Authenticated`, `Compressed`, `Binded`, and `SessionStarted`. Used by `inC2S` to gate feature negotiation and stanza processing.

### `resources` — Per-User Resource Set

**File:** `resources.go`

A thread-safe collection of bound `stream.C2S` streams for a single user. Supports `bind`, `unbind`, `route`, `disconnect`, and enumeration. Used internally by `LocalRouter`.

### `ListenerConfig` / `ListenersConfig` — Configuration

**File:** `config.go`

YAML-driven configuration for C2S listeners:

| Field | Default | Description |
| --- | --- | --- |
| `bind_addr` | — | Listen address |
| `port` | `5222` | Listen port |
| `transport` | `socket` | Transport type |
| `direct_tls` | `false` | TLS on connect (no StartTLS) |
| `sasl.mechanisms` | `[scram_sha_1, scram_sha_256, scram_sha_512, scram_sha3_512]` | Enabled SASL mechanisms |
| `sasl.external.address` | — | External authenticator gRPC address |
| `compression_level` | `default` | zlib compression level (`default`, `best`, `speed`, `no_compression`) |
| `resource_conflict` | `terminate_old` | Resource conflict resolution (`override`, `disallow`, `terminate_old`) |
| `max_stanza_size` | `524288` (512 KB) | Max incoming stanza size in bytes |
| `conn_timeout` | `3s` | TCP connection timeout |
| `auth_timeout` | `10s` | Authentication timeout |
| `keep_alive_timeout` | `3m` | Inactive connection timeout |
| `req_timeout` | `15s` | Per-request timeout |

### Metrics

**File:** `metrics.go`

Exposes Prometheus metrics under the `jackal_c2s_` namespace:

| Metric | Type | Description |
| --- | --- | --- |
| `connection_registered` | Counter | Total stream register operations |
| `connection_unregistered` | Counter | Total stream unregister operations |
| `incoming_requests_total` | Counter | Incoming stanza count by name/type |
| `outgoing_requests_total` | Counter | Outgoing stanza count by name/type |
| `incoming_requests_duration_bucket` | Histogram | Incoming stanza processing latency |
| `incoming_total_connections` | Gauge | Current total C2S connections |

### XMPP Namespaces

**File:** `namespace.go`

Constants for XMPP protocol namespaces used during stream negotiation: `stream`, `sasl`, `tls`, `compress`, `bind`, `session`, and `blocking:errors`.

### Protobuf — `pb/resourceinfo.pb.go`

Generated protobuf types for serializing resource information across cluster nodes. Contains `ResourceInfo` with instance ID, domain, info map, and presence data.

## Interfaces & Mocking

**File:** `interface.go`

Defines private interface aliases for all external dependencies (`hosts`, `session`, `localRouter`, `clusterRouter`, `components`, `modules`, `globalRouter`, etc.) with `//go:generate moq` directives. This enables isolated unit testing with generated mock implementations.

## Key Dependencies

| Package | Role |
| --- | --- |
| `pkg/auth` | SASL authenticators (SCRAM, External) |
| `pkg/hook` | Priority-based event hooks |
| `pkg/host` | Local host/domain management, TLS certificates |
| `pkg/router` | Global stanza routing (C2S, S2S) |
| `pkg/cluster/resourcemanager` | Distributed resource tracking |
| `pkg/cluster/router` | Inter-node gRPC stanza routing |
| `pkg/component` | External component (XEP-0114) stanza delegation |
| `pkg/module` | XMPP module IQ processing and stream features |
| `pkg/session` | Low-level XML stream read/write |
| `pkg/shaper` | Rate limiting and max session enforcement |
| `pkg/transport` | Socket transport with TLS and compression |

## Usage

```go
// Create listeners from config
listeners := c2s.NewListeners(cfg, hosts, router, comps, mods, resMng, rep, peppers, shapers, hk, logger)
for _, ln := range listeners {
    _ = ln.Start(ctx)
}

// Create and start the C2S router
c2sRouter := c2s.NewRouter(localRouter, clusterRouter, resMng, rep, hk, logger)
_ = c2sRouter.Start(ctx)
```
