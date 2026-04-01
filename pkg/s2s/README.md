# S2S — Server-to-Server Federation

This package implements XMPP Server-to-Server (S2S) federation for jackal, enabling users on different XMPP servers to communicate with each other. A message from `alice@server-a.example` to `bob@server-b.example` is transparently routed through the S2S subsystem.

## Architecture

```
                        ┌───────────────────────────────────────────────┐
                        │               jackal instance                 │
                        │                                               │
                        │  ┌──────────┐                                 │
 Remote XMPP Server ────┼─▶│ Socket   │──▶ inS2S stream ──▶ InHub       │
    (inbound :5269)     │  │ Listener │    (state machine)  (registry)  │
                        │  └──────────┘          │                      │
                        │                        ▼                      │
                        │                 ┌────────────┐                │
                        │                 │   Global   │                │
                        │                 │   Router   │                │
                        │                 └──────┬─────┘                │
                        │                        │                      │
                        │            ┌───────────┼───────────┐          │
                        │            ▼           ▼           ▼          │
                        │        C2S Router  Components  Modules        │
                        │            │                                  │
                        │            ▼                                  │
                        │       Local Users                             │
                        │                                               │
                        │  ┌────────────┐                               │
                        │  │ S2S Router │──▶ OutProvider ──▶ outS2S ────┼──▶ Remote XMPP Server
                        │  └────────────┘   (conn pool)   (state        │     (outbound)
                        │                                  machine)     │
                        └───────────────────────────────────────────────┘
```

### Component Responsibilities

| Component | File | Role |
|---|---|---|
| **SocketListener** | `socket_listener.go` | Accepts incoming TCP/TLS connections on the S2S port |
| **inS2S** | `in.go` | Handles an incoming S2S stream: TLS, authentication, stanza routing |
| **InHub** | `in_hub.go` | Registry of all active inbound streams; handles graceful shutdown |
| **S2S Router** | `router.go` | Routes outbound stanzas by resolving target domain to an outS2S stream |
| **OutProvider** | `out_provider.go` | Connection pool for outbound streams (keyed by sender→target domain pair) |
| **outS2S** | `out.go` | Handles an outgoing S2S stream: TLS, authentication, stanza queuing |
| **Dialer** | `dialer.go` | DNS SRV resolution and TCP/TLS connection establishment |
| **Dialback** | `dialback.go` | XEP-0220 Server Dialback key generation and request tracking |

### Supporting Files

| File | Purpose |
|---|---|
| `interface.go` | All interface definitions (session, transport, hosts, router, etc.) |
| `config.go` | Configuration structs for listeners and outgoing connections |
| `flags.go` | Thread-safe bitmask for stream state (secured, authenticated, dialback-authorised) |
| `namespace.go` | XMPP namespace constants (stream, SASL, TLS, dialback) |
| `metrics.go` | Prometheus counters and gauges for S2S connections and stanzas |

---

## Connection Lifecycle

### Outbound: Sending a Stanza to a Remote Server

When the global router determines the recipient is on a remote domain, the following flow executes:

```
 1. Global Router
    │
    │  stanza.ToJID().Domain() is not a local host
    ▼
 2. S2S Router
    │
    │  outProvider.GetOut(ctx, senderDomain, targetDomain)
    ▼
 3. OutProvider (connection pool)
    │
    │  Stream exists?  ──yes──▶  Return cached outS2S
    │       │no
    ▼
 4. Dialer — resolve remote server
    │
    │  a) DNS SRV: _xmpps-server._tcp.target  →  Direct TLS
    │  b) DNS SRV: _xmpp-server._tcp.target   →  STARTTLS
    │  c) Fallback: target:5269               →  STARTTLS
    ▼
 5. outS2S state machine negotiates the connection
    │
    │  TLS → SASL EXTERNAL or Dialback → Authenticated
    ▼
 6. Stanza delivered via stream
    │
    │  Pending stanzas (queued during negotiation) are flushed
    ▼
 7. Stream cached in OutProvider for reuse
```

### Inbound: Receiving a Stanza from a Remote Server

```
 1. SocketListener accepts TCP connection (:5269 or :5270 for direct TLS)
    │
    ▼
 2. inS2S stream created, registered in InHub
    │
    ▼
 3. inS2S state machine negotiates
    │
    │  Advertise features → TLS → SASL EXTERNAL or Dialback → Authenticated
    ▼
 4. Stanzas received and routed
    │
    │  IQ      → Module processor or Global Router
    │  Message → Global Router (→ C2S Router → local user)
    │  Presence→ Global Router (full JID only)
    ▼
 5. Stream closed → unregistered from InHub
```

---

## State Machines

### Outbound Stream (`outS2S`)

```
                    ┌─────────────────┐
                    │  outConnecting   │  (initial state after dial)
                    └────────┬────────┘
                             │ receive <stream:stream>
                             ▼
                    ┌─────────────────┐
             ┌──── │  outConnected    │ ◀──────────────────┐
             │     └────────┬────────┘                     │
             │              │ receive <stream:features>     │
             │              ▼                               │
             │     ┌──── TLS needed? ────┐                 │
             │     │yes                  │no               │
             │     ▼                     ▼                 │
             │  ┌──────────────┐   Authenticated?          │
             │  │ outSecuring  │      │no    │yes          │
             │  └──────┬───────┘      ▼      ▼             │
             │         │         EXTERNAL? ┌──────────────────┐
             │  <proceed>        │yes  │no │outAuthenticated│
             │  + StartTLS       ▼     ▼   │  (send pending) │
             │  + restart   ┌─────────┐ Dialback?            │
             │    stream    │outAuth- │ │yes  │no            │
             │     │        │enti-    │ ▼     ▼              │
             │     │        │cating   │ ┌──────────────┐     │
             │     │        └────┬────┘ │outVerifying-  │     │
             │     │    <success>│      │DialbackKey   │     │
             │     │     + restart      └──────┬───────┘     │
             │     │       stream    <db:result │             │
             │     │         │       type=valid>│             │
             │     └─────────┴─────────────────┴─────────────┘
             │
             │ (any error or disconnect)
             ▼
    ┌─────────────────┐
    │ outDisconnected  │
    └─────────────────┘
```

**Dialback-only variant** (`dialbackType`): After connecting, sends `<db:verify>`, receives the result, then disconnects. Used exclusively for verifying a remote server's identity on behalf of an inbound stream.

### Inbound Stream (`inS2S`)

```
                    ┌─────────────────┐
                    │  inConnecting    │  (initial state)
                    └────────┬────────┘
                             │ receive <stream:stream>
                             │ send <stream:features>
                             ▼
                    ┌─────────────────┐
             ┌──── │  inConnected     │ ◀──────────────────┐
             │     └────────┬────────┘                     │
             │              │                              │
             │     ┌──── Secured? ────────┐                │
             │     │no                    │yes             │
             │     ▼                      ▼                │
             │  <starttls>        Receive element:         │
             │  → <proceed>       ┌─────┬──────┬────────┐  │
             │  → StartTLS        │     │      │        │  │
             │  → restart         │     │      │        │  │
             │     stream         │     │      │        │  │
             │     │              ▼     ▼      ▼        ▼  │
             │     │          <auth> <db:    <db:   stanza  │
             │     │           │    result> verify>  (if    │
             │     │           ▼      │      │     auth'd) │
             │     │       Verify     ▼      ▼        │    │
             │     │       peer  ┌─────────┐ Verify   │    │
             │     │       cert  │inAuth-  │ key in   │    │
             │     │         │   │orizing- │ KV store │    │
             │     │    valid?   │Dialback │    │     │    │
             │     │    │yes │no │Key      │ send     │    │
             │     │    ▼    ▼   └────┬────┘ db:verify│    │
             │     │ <success> <fail> │ result  result │    │
             │     │ + restart        │ from           │    │
             │     │   stream         │ dialback       │    │
             │     │     │            │ channel        │    │
             │     └─────┴────────────┴────────────────┘    │
             │                                              │
             │ (any error or disconnect)                    │
             ▼                                              │
    ┌─────────────────┐                                     │
    │ inDisconnected   │                                    │
    └─────────────────┘                                     │
```

### Stream Flags (Bitmask)

Both inbound and outbound streams track security/auth state with a thread-safe bitmask:

```
Bit 0: fSecured                 — TLS negotiation completed
Bit 1: fAuthenticated           — SASL EXTERNAL auth succeeded
Bit 2: fDialbackKeyAuthorized   — Dialback verification passed
```

A stream is considered authorised to route stanzas if **either** `fAuthenticated` or `fDialbackKeyAuthorized` is set.

---

## Authentication

### SASL EXTERNAL

Used when the remote server presents a valid TLS client certificate:

```
Connecting Server                          Receiving Server
      │                                          │
      │──── TLS handshake (with client cert) ───▶│
      │                                          │
      │◀── <features><mechanisms>EXTERNAL ──────│
      │                                          │
      │──── <auth mechanism="EXTERNAL">          │
      │      base64(sender-domain)          ────▶│
      │                                          │── Verify peer cert
      │                                          │   DNSNames contains
      │                                          │   sender domain
      │◀── <success/> ─────────────────────────│
      │                                          │
      │──── (stream restart, ready to route) ───▶│
```

### Server Dialback (XEP-0220)

Used as a fallback when client certificates are not available:

```
Server A (sender)                 Server B (target)
      │                                  │
      │──── <stream:stream> ────────────▶│
      │◀── <features> (dialback) ───────│
      │                                  │
      │──── <db:result                   │
      │      from="a" to="b">key ──────▶│  key = HMAC-SHA256(
      │                                  │    SHA256(secret),
      │                                  │    "b a streamID")
      │                                  │
      │     ┌────── Server B opens ──────┤
      │     │       new connection       │
      │     │       to Server A          │
      │     │                            │
      │     │  <db:verify from="b"       │
      │◀────│   to="a" id="streamID">   │
      │     │   key                      │
      │     │                            │
      │── Lookup KV[db://streamID] ──┐   │
      │   Compare key with expected  │   │
      │◀─────────────────────────────┘   │
      │                                  │
      │──── <db:verify type="valid"> ───▶│  (on the verification connection)
      │     (verification conn closes)   │
      │                                  │
      │◀── <db:result type="valid"> ────│  (on the original connection)
      │                                  │
      │  Connection authenticated ✓      │
```

**Dialback key storage:** Pending requests are stored in the distributed KV store as `db://<streamID>` → `"<sender> <target>"`, enabling dialback verification to work across cluster nodes.

---

## DNS Resolution and Connection Establishment

The `Dialer` resolves a remote domain to a network address using this priority:

```
1. _xmpps-server._tcp.<domain>   →  Direct TLS connection (XEP-0368)
2. _xmpp-server._tcp.<domain>    →  Plain TCP, then STARTTLS
3. <domain>:5269                  →  Fallback plain TCP, then STARTTLS
```

Each SRV lookup may return multiple targets with priority and weight. The dialer tries targets in order until a connection succeeds.

---

## Configuration

### Inbound Listeners

```yaml
s2s:
  listeners:
    - port: 5269              # Standard S2S port
      connect_timeout: 3s     # TCP accept timeout
      keep_alive_timeout: 10m # Idle stream timeout
      req_timeout: 15s        # Per-stanza processing timeout
      max_stanza_size: 1048576 # 1 MB
      direct_tls: false       # Set true for port 5270 (XEP-0368)
```

### Outbound Connections

```yaml
s2s:
  out:
    dialback_secret: "shared-secret"  # HMAC key for dialback auth
    dial_timeout: 5s                  # Remote server connect timeout
    keep_alive_timeout: 10m           # Idle stream timeout
    req_timeout: 15s                  # Per-stanza timeout
    max_stanza_size: 131072           # 128 KB
```

---

## Metrics

The package exports Prometheus metrics for monitoring federation health:

| Metric | Type | Description |
|---|---|---|
| `s2s_incoming_connection_registered` | Counter | Inbound streams opened |
| `s2s_incoming_connection_unregistered` | Counter | Inbound streams closed |
| `s2s_outgoing_connection_registered` | Counter | Outbound streams opened (label: `default`/`dialback`) |
| `s2s_outgoing_connection_unregistered` | Counter | Outbound streams closed |
| `s2s_incoming_total_connections` | Gauge | Current inbound stream count (updated every 30s) |
| `s2s_outgoing_total_connections` | Gauge | Current outbound stream count (updated every 30s) |
| `s2s_incoming_requests_total` | Counter | Stanzas received (labels: name, type) |
| `s2s_outgoing_requests_total` | Counter | Stanzas sent (labels: name, type) |
| `s2s_incoming_requests_duration_bucket` | Histogram | Inbound stanza processing latency |

---

## Hooks

The S2S subsystem fires lifecycle events through jackal's hook system, allowing modules and extensions to observe or intercept federation activity:

### Outbound

| Hook | Fired When |
|---|---|
| `S2SOutStreamConnected` | Outbound stream fully authenticated |
| `S2SOutStreamElementSent` | Stanza sent to remote server |
| `S2SOutStreamDisconnected` | Outbound stream closed |

### Inbound

| Hook | Fired When |
|---|---|
| `S2SInStreamRegistered` | New inbound stream accepted |
| `S2SInStreamElementReceived` | Any element received (before processing) |
| `S2SInStreamIQReceived` | IQ stanza received |
| `S2SInStreamIQRouted` | IQ stanza successfully routed |
| `S2SInStreamMessageReceived` | Message stanza received |
| `S2SInStreamMessageRouted` | Message stanza successfully routed |
| `S2SInStreamPresenceReceived` | Presence stanza received |
| `S2SInStreamPresenceRouted` | Presence stanza successfully routed |
| `S2SInStreamWillRouteElement` | Just before routing any stanza |
| `S2SInStreamUnregistered` | Inbound stream closed |

---

## Concurrency Model

- **Run queues:** Each stream uses a dedicated `RunQueue` to serialise operations (send, disconnect, element handling), avoiding lock contention while maintaining ordering.
- **Connection pool:** `OutProvider` uses a read-write mutex with double-check locking — reads are concurrent, writes (new connections) are serialised.
- **Async operations:** `SendElement()` and `Disconnect()` return `<-chan error` channels, allowing callers to optionally wait for completion.
- **Graceful shutdown:** `InHub.Stop()` and `OutProvider.Stop()` disconnect all active streams with a `SystemShutdown` error and wait for completion (or context timeout).

---

## Example: Full Federation Flow

Alice (`alice@server-a.example`) sends a message to Bob (`bob@server-b.example`):

```
  server-a.example                                    server-b.example
 ┌──────────────────┐                                ┌──────────────────┐
 │                  │                                │                  │
 │ Alice's C2S      │                                │                  │
 │ stream sends:    │                                │                  │
 │ <message         │                                │                  │
 │  to="bob@        │                                │                  │
 │  server-b...">   │                                │                  │
 │       │          │                                │                  │
 │       ▼          │                                │                  │
 │ Global Router    │                                │                  │
 │ "server-b is     │                                │                  │
 │  not local"      │                                │                  │
 │       │          │                                │                  │
 │       ▼          │                                │                  │
 │ S2S Router       │                                │                  │
 │       │          │                                │                  │
 │       ▼          │                                │                  │
 │ OutProvider      │                                │                  │
 │ (get or create   │                                │                  │
 │  outS2S stream)  │                                │                  │
 │       │          │    TCP + TLS + Auth             │                  │
 │       ▼          │ ─────────────────────────────▶ │ SocketListener   │
 │ outS2S stream ───┼── <message to="bob@..."> ────▶│       │          │
 │                  │                                │       ▼          │
 │                  │                                │ inS2S stream     │
 │                  │                                │       │          │
 │                  │                                │       ▼          │
 │                  │                                │ Global Router    │
 │                  │                                │ "server-b is     │
 │                  │                                │  local"          │
 │                  │                                │       │          │
 │                  │                                │       ▼          │
 │                  │                                │ C2S Router       │
 │                  │                                │       │          │
 │                  │                                │       ▼          │
 │                  │                                │ Bob's C2S stream │
 │                  │                                │ receives message │
 └──────────────────┘                                └──────────────────┘
```

---

## Related Specifications

| Specification | Relevance |
|---|---|
| [RFC 6120](https://xmpp.org/rfcs/rfc6120.html) | XMPP Core — S2S stream negotiation, TLS, SASL |
| [XEP-0220](https://xmpp.org/extensions/xep-0220.html) | Server Dialback — identity verification fallback |
| [XEP-0368](https://xmpp.org/extensions/xep-0368.html) | SRV Records for XMPP over TLS — direct TLS via `_xmpps-server` SRV |
| [XEP-0138](https://xmpp.org/extensions/xep-0138.html) | Stream Compression — zlib compression for S2S streams |
