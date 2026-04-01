# jackal

A high-performance, scalable XMPP server written in Go.

![CI Status](https://github.com/ortuman/jackal/workflows/CI/badge.svg)
[![Go Report Card](https://goreportcard.com/badge/github.com/ortuman/jackal?style=flat-square)](https://goreportcard.com/report/github.com/ortuman/jackal)
[![Godoc](http://img.shields.io/badge/go-documentation-blue.svg?style=flat-square)](https://godoc.org/github.com/ortuman/jackal)
[![Releases](https://img.shields.io/github/release/ortuman/jackal/all.svg?style=flat-square)](https://github.com/ortuman/jackal/releases)
[![LICENSE](https://img.shields.io/github/license/ortuman/jackal.svg?style=flat-square)](https://github.com/ortuman/jackal/blob/master/LICENSE)
[![Docker Pulls](https://img.shields.io/docker/pulls/ortuman/jackal.svg)](https://hub.docker.com/r/ortuman/jackal/)
[![Join the chat at https://gitter.im/jackal-im/jackal](https://badges.gitter.im/jackal-im/jackal.svg)](https://gitter.im/jackal-im/jackal?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge)

<div align="center">
    <a href="#">
        <img src="./logos/logo-0.png">
    </a>
</div>

---

## Table of Contents

- [About](#about)
- [Features](#features)
- [Architecture](#architecture)
  - [High-Level Overview](#high-level-overview)
  - [Project Structure](#project-structure)
  - [Design Patterns](#design-patterns)
  - [Stream Lifecycle](#stream-lifecycle)
- [Getting Started](#getting-started)
  - [Prerequisites](#prerequisites)
  - [Building from Source](#building-from-source)
  - [Configuration](#configuration)
  - [Database Setup (PostgreSQL)](#database-setup-postgresql)
  - [Database Setup (BoltDB)](#database-setup-boltdb)
  - [Creating Users](#creating-users)
- [Deployment](#deployment)
  - [Docker](#docker)
  - [Docker Compose](#docker-compose)
  - [Kubernetes (Helm)](#kubernetes-helm)
- [Clustering](#clustering)
  - [Overview](#overview)
  - [Configuration](#cluster-configuration)
  - [How It Works](#how-it-works)
- [Modules & XMPP Extensions](#modules--xmpp-extensions)
  - [Module System](#module-system)
  - [Supported Specifications](#supported-specifications)
- [Server Extensibility](#server-extensibility)
- [Security](#security)
- [Monitoring & Observability](#monitoring--observability)
- [Development](#development)
  - [Makefile Targets](#makefile-targets)
  - [Running Tests](#running-tests)
  - [Code Generation](#code-generation)
- [Contributing](#contributing)
- [License](#license)
- [Contact](#contact)

---

## About

**jackal** is a free, open-source XMPP server built for stability, simple configuration, and low resource consumption. It implements the core XMPP standards ([RFC 6120](https://xmpp.org/rfcs/rfc6120.html), [RFC 6121](https://xmpp.org/rfcs/rfc6121.html)) along with 24+ XMPP Extension Protocols (XEPs), making it suitable for production instant messaging deployments.

jackal is written entirely in Go with a clean, modular architecture that emphasises interface-based design, pluggable storage backends, and a hook-driven event system for extensibility.

---

## Features

| Category | Details |
|---|---|
| **Protocol** | Full XMPP Core (RFC 6120) and XMPP IM (RFC 6121) compliance |
| **Security** | Enforced TLS, SCRAM-SHA-1/256/512/3-512 authentication with channel binding, password peppering |
| **Storage** | Pluggable backends — PostgreSQL 9.5+ and BoltDB (embedded) |
| **Caching** | Optional Redis 6.2+ cache layer with transparent read-through |
| **Clustering** | Multi-node clustering via etcd 3.4+ with gRPC inter-node communication |
| **Extensibility** | 11 built-in XEP modules, external component support (XEP-0114), gRPC extension API |
| **Observability** | Prometheus metrics, Grafana dashboard, health checks, Go pprof profiling |
| **Compression** | zlib stream compression (XEP-0138) |
| **Deployment** | Static binary (CGO_ENABLED=0), Docker images, Helm chart for Kubernetes |
| **Platforms** | macOS, Linux |

---

## Architecture

### High-Level Overview

```
┌───────────────────────────────────────────────────────────────────────┐
│                          Client Applications                          │
└─────────┬──────────────────────┬──────────────────────┬───────────────┘
          │ :5222 (C2S)          │ :5269 (S2S)          │ :5275 (Component)
┌─────────▼──────────────────────▼──────────────────────▼───────────────┐
│                        Transport Layer                                 │
│              (TLS · Direct TLS · zlib Compression)                     │
├───────────────────────────────────────────────────────────────────────┤
│                         Session Layer                                  │
│            (XML Stream Parsing · Stanza Handling)                      │
├───────────────────────────────────────────────────────────────────────┤
│                      Authentication Layer                              │
│        (SASL · SCRAM-SHA-1/256/512/3-512 · Channel Binding)           │
├───────────────────────────────────────────────────────────────────────┤
│                         Router Engine                                  │
│            ┌──────────┬──────────────┬──────────────┐                  │
│            │ Local    │   Cluster    │     S2S      │                  │
│            │ Router   │   Router     │   Router     │                  │
│            └──────────┴──────────────┴──────────────┘                  │
├───────────────────────────────────────────────────────────────────────┤
│  Hook System  │  Module Registry (XEPs)  │  Component Registry         │
├───────────────────────────────────────────────────────────────────────┤
│                        Storage Layer                                   │
│        ┌────────────┬────────────────┬─────────────────┐               │
│        │ PostgreSQL │    BoltDB      │  Redis (Cache)  │               │
│        └────────────┴────────────────┴─────────────────┘               │
├───────────────────────────────────────────────────────────────────────┤
│                      Cluster Coordination                              │
│               (etcd KV · gRPC · Resource Manager)                      │
├───────────────────────────────────────────────────────────────────────┤
│                     Observability Layer                                 │
│         (Prometheus :6060 · Grafana · pprof · Health Checks)           │
└───────────────────────────────────────────────────────────────────────┘
```

### Project Structure

```
jackal/
├── cmd/
│   ├── jackal/              # Main server binary entry point
│   └── jackalctl/           # Admin CLI tool (user management, version)
│       └── ctlv1/           # CLI v1 commands
├── pkg/
│   ├── admin/               # gRPC admin API server
│   ├── auth/                # SASL authentication (SCRAM + external)
│   │   └── pepper/          # Password pepper key management
│   ├── c2s/                 # Client-to-Server stream handling
│   ├── s2s/                 # Server-to-Server federation
│   ├── cluster/             # Clustering subsystem
│   │   ├── connmanager/     # Cluster connection management
│   │   ├── instance/        # Cluster instance identity
│   │   ├── kv/              # Key-value store abstraction (etcd)
│   │   ├── memberlist/      # Cluster membership tracking
│   │   ├── resourcemanager/ # Distributed resource registry
│   │   ├── router/          # Cross-node stanza routing
│   │   └── server/          # Cluster gRPC server
│   ├── component/           # External XMPP component registry
│   ├── hook/                # Priority-based event/hook system
│   ├── host/                # Virtual host (domain) management
│   ├── jackal/              # Core server bootstrap & lifecycle
│   ├── log/                 # Structured logging (go-kit/log)
│   ├── model/               # Domain models
│   │   ├── archive/         # Message archive (XEP-0313)
│   │   ├── blocklist/       # Blocked contacts
│   │   ├── c2s/             # C2S resource descriptors
│   │   ├── caps/            # Entity capabilities
│   │   ├── disco/           # Service discovery
│   │   ├── last/            # Last activity
│   │   ├── roster/          # Contact roster
│   │   └── user/            # User accounts
│   ├── module/              # Pluggable XMPP extension modules
│   │   ├── offline/         # Offline message storage (XEP-0160)
│   │   ├── roster/          # Roster management (RFC-6121)
│   │   ├── xep0004/         # Data Forms
│   │   ├── xep0012/         # Last Activity
│   │   ├── xep0030/         # Service Discovery
│   │   ├── xep0049/         # Private XML Storage
│   │   ├── xep0054/         # vCard-temp
│   │   ├── xep0059/         # Result Set Management
│   │   ├── xep0092/         # Software Version
│   │   ├── xep0115/         # Entity Capabilities
│   │   ├── xep0191/         # Blocking Command
│   │   ├── xep0198/         # Stream Management
│   │   ├── xep0199/         # XMPP Ping
│   │   ├── xep0202/         # Entity Time
│   │   ├── xep0280/         # Message Carbons
│   │   └── xep0313/         # Message Archive Management
│   ├── router/              # Core stanza routing engine
│   ├── session/             # XMPP session state machine
│   ├── shaper/              # Rate limiting & traffic shaping
│   ├── storage/             # Storage layer with decorator chain
│   │   ├── boltdb/          # BoltDB implementation
│   │   ├── cached/          # Redis cache decorator
│   │   ├── measured/        # Prometheus metrics decorator
│   │   ├── pgsql/           # PostgreSQL implementation
│   │   └── repository/      # Repository interfaces
│   ├── transport/           # Network transport (socket, TLS)
│   │   └── compress/        # zlib stream compression
│   ├── util/                # Utilities (DNS, TLS, rate limiter, etc.)
│   └── version/             # Version information
├── proto/                   # Protocol Buffer definitions
│   ├── admin/v1/            # Admin API (user management)
│   ├── c2s/v1/              # C2S resource info
│   ├── cluster/v1/          # Cluster inter-node RPC
│   └── model/v1/            # Data models (user, roster, archive, etc.)
├── config/                  # Example YAML configuration files
├── sql/                     # Database schema migrations
├── dockerfiles/             # Dockerfile and docker-compose
├── helm/                    # Kubernetes Helm chart
├── monitoring/              # Grafana dashboard JSON
├── scripts/                 # Build, test, and CI scripts
└── logos/                   # Project logos
```

### Design Patterns

jackal employs several well-established Go design patterns:

#### Interface-Based Dependency Injection

All major components are defined as interfaces, enabling clean dependency boundaries and straightforward mock-based testing:

```go
// Router abstraction
type Router interface {
    Route(ctx context.Context, stanza stravaganza.Stanza) ([]jid.JID, error)
    C2S() C2SRouter
    S2S() S2SRouter
    Start(ctx context.Context) error
    Stop(ctx context.Context) error
}

// Repository abstraction (composed of 10 sub-interfaces)
type Repository interface {
    User
    Roster
    Archive
    BlockList
    // ... and more
    InTransaction(ctx context.Context, f func(ctx context.Context, tx Transaction) error) error
}
```

#### Decorator/Middleware Chain (Storage Layer)

The storage layer uses a decorator pattern to compose functionality:

```
Request → Measured (Prometheus) → Cached (Redis) → Base (PostgreSQL / BoltDB)
```

Each layer wraps the next, transparently adding metrics collection and caching without modifying the base implementation.

#### Priority-Based Hook/Event System

Modules and subsystems communicate through a decoupled event system:

```go
// Register a handler with priority
hooks.AddHook("modulesStarted", myHandler, hook.HighPriority)

// Fire an event — handlers execute in priority order
halted, err := hooks.Run("modulesStarted", executionContext)
```

#### State Machine (C2S Streams)

Each client connection follows a well-defined state machine:

```
Connecting → Connected → Authenticating → Authenticated → Bound → Disconnected → Terminated
```

#### Module Plugin Architecture

XMPP extensions are implemented as pluggable modules conforming to a common interface:

```go
type Module interface {
    Name() string
    StreamFeature(ctx context.Context, domain string) (stravaganza.Element, error)
    ServerFeatures(ctx context.Context) ([]string, error)
    AccountFeatures(ctx context.Context) ([]string, error)
    Start(ctx context.Context) error
    Stop(ctx context.Context) error
}
```

Modules that handle IQ stanzas additionally implement `IQProcessor`.

### Stream Lifecycle

```
1. TCP Connection Accepted
       │
2. TLS Handshake (StartTLS or Direct TLS)
       │
3. XML Stream Opened ──→ Stream Features Advertised
       │
4. SASL Authentication (SCRAM-SHA negotiation)
       │
5. Stream Restart ──→ Resource Binding
       │
6. Session Established ──→ Resource Registered (local + cluster)
       │
7. Stanza Routing Active
   ├── IQ → Module IQ Processors
   ├── Message → Router (local / cluster / S2S)
   └── Presence → Roster Module + Broadcast
       │
8. Disconnect / Error ──→ Resource Unregistered ──→ Stream Terminated
```

---

## Getting Started

### Prerequisites

| Dependency | Version | Required | Purpose |
|---|---|---|---|
| **Go** | 1.19+ | Yes | Build from source |
| **PostgreSQL** | 9.5+ | Yes* | Primary persistent storage |
| **BoltDB** | — | Yes* | Alternative embedded storage (single-node only) |
| **Redis** | 6.2+ | No | Optional caching layer |
| **etcd** | 3.4+ | No | Required for clustering |

\* One of PostgreSQL or BoltDB is required.

### Building from Source

```bash
# Clone the repository
git clone git@github.com:ortuman/jackal.git
cd jackal

# Build and install both binaries
make install installctl
```

This installs `jackal` (server) and `jackalctl` (admin CLI) into your `$GOPATH/bin`.

### Configuration

jackal uses a YAML configuration file. By default it looks for `config.yaml` in the working directory. You can override this:

```bash
# Via command-line flag
jackal --config=/path/to/config.yaml

# Via environment variable
export JACKAL_CONFIG_FILE=/path/to/config.yaml
jackal
```

Configuration values can also be overridden with environment variables using the `JACKAL_` prefix (e.g., `JACKAL_STORAGE_PGSQL_PASSWORD`).

See [`config/example.config.yaml`](config/example.config.yaml) for a fully documented example covering all options.

#### Key Configuration Sections

```yaml
# Password security — pepper keys for password hashing
peppers:
  keys:
    v1: "your-secret-pepper-key"
  use: v1

# Logging
logger:
  level: info                    # debug | info | warn | error
  output_path: ""                # empty = stdout

# HTTP server for metrics and health checks
http:
  port: 6060

# gRPC admin API
admin:
  port: 15280

# Virtual hosts (XMPP domains)
hosts:
  - domain: example.com
    tls:
      cert_file: /path/to/cert.pem
      privkey_file: /path/to/key.pem

# Storage backend
storage:
  type: pgsql                    # pgsql | boltdb
  pgsql:
    host: 127.0.0.1:5432
    user: jackal
    password: password
    database: jackal
    max_open_conns: 16
  cache:
    type: redis                  # none | redis
    redis:
      addresses:
        - localhost:6379

# Client-to-Server listeners
c2s:
  listeners:
    - port: 5222
      transport: socket
      req_timeout: 60s
      direct_tls: false
      sasl:
        mechanisms:
          - scram_sha_1
          - scram_sha_256
          - scram_sha_512
          - scram_sha3_512

# Server-to-Server federation
s2s:
  listeners:
    - port: 5269
      req_timeout: 60s
      max_stanza_size: 131072
  out:
    dialback_secret: "your-dialback-secret"
    dial_timeout: 5s

# Rate limiting
shapers:
  - name: default
    max_sessions: 10
    rate:
      limit: 65536
      burst: 32768

# Enabled XMPP extension modules
modules:
  enabled:
    - roster
    - offline
    - last
    - disco
    - private
    - vcard
    - version
    - caps
    - blocklist
    - stream_mgmt
    - ping
    - time
    - carbons
    - mam
```

### Database Setup (PostgreSQL)

1. **Create the database and user:**

```sql
CREATE ROLE jackal WITH LOGIN PASSWORD 'password';
CREATE DATABASE jackal;
GRANT ALL PRIVILEGES ON DATABASE jackal TO jackal;
```

2. **Apply the schema:**

```bash
psql --user jackal --password -f sql/postgres.up.psql
```

The schema creates tables for users, roster items, offline messages, message archives, vCards, blocklists, private storage, capabilities, and last activity — all with appropriate indexes and automatic timestamp triggers.

3. **Configure jackal** to use PostgreSQL (see [Configuration](#configuration) above).

### Database Setup (BoltDB)

For single-node deployments, BoltDB provides an embedded alternative that requires no external database:

```yaml
storage:
  type: boltdb
  boltdb:
    path: /path/to/jackal.db
```

> **Note:** BoltDB does not support clustering. Use PostgreSQL for multi-node deployments.

### Creating Users

After starting jackal, register users via the admin CLI:

```bash
# Create a user
jackalctl user add username:password

# Delete a user
jackalctl user delete username
```

The CLI connects to the jackal gRPC admin API (default port `15280`).

---

## Deployment

### Docker

Pull the official image:

```bash
docker pull ortuman/jackal:latest
```

Run with a custom configuration:

```bash
docker run --name jackal \
  --mount type=bind,src=/path/to/config.yaml,dst=/jackal/config.yaml \
  -p 5222:5222 \
  -p 15280:15280 \
  -d ortuman/jackal:latest
```

| Port | Protocol | Purpose |
|---|---|---|
| `5222` | TCP | C2S (client connections) |
| `5223` | TCP | C2S (direct TLS) |
| `5269` | TCP | S2S (server federation) |
| `5270` | TCP | S2S (direct TLS) |
| `5275` | TCP | External components |
| `6060` | HTTP | Metrics, health, pprof |
| `14369` | gRPC | Cluster inter-node |
| `15280` | gRPC | Admin API |

### Docker Compose

Spin up jackal with all dependencies (PostgreSQL + etcd):

```bash
docker-compose -f dockerfiles/docker-compose.yml up
```

This starts:
- **jackal** — XMPP server (ports 5222, 15280)
- **PostgreSQL 13.3** — persistent storage (initialised with schema)
- **etcd 3.4** — cluster coordination

The compose setup uses `wait-for-it.sh` to ensure dependencies are ready before jackal starts.

### Kubernetes (Helm)

jackal includes a production-ready Helm chart with high-availability defaults:

```bash
# Install
sh ./helm/scripts/install.sh your-values.yaml

# Upgrade
sh ./helm/scripts/upgrade.sh your-values.yaml

# Uninstall
sh ./helm/scripts/uninstall.sh
```

**Default HA topology:**

| Component | Replicas | Purpose |
|---|---|---|
| jackal | 2 | XMPP server instances |
| etcd | 3 | Cluster coordination |
| PostgreSQL-HA | 2 + pgpool | Persistent storage |
| Redis | 2 | Cache layer |

The Helm chart includes:
- Init containers that wait for dependency readiness
- ConfigMap-based configuration injection
- Health and readiness probes (`/healthz`)
- Resource requests/limits
- Security contexts (non-root)

Customise the deployment by editing [`helm/values.yaml`](helm/values.yaml).

---

## Clustering

### Overview

jackal supports horizontal scaling through a clustering architecture built on [etcd](https://etcd.io/) for distributed state and gRPC for inter-node communication.

### Cluster Configuration

Add a `cluster` section to each node's configuration:

```yaml
cluster:
  type: kv
  kv:
    type: etcd
    etcd:
      endpoints:
        - http://etcd-host1:2379
        - http://etcd-host2:2379
        - http://etcd-host3:2379
  server:
    port: 14369              # Must be reachable by other cluster nodes
```

### How It Works

1. **Member Discovery** — Each jackal instance registers itself in etcd on startup. The member list is kept in sync across the cluster.

2. **Resource Tracking** — When a user binds a resource (connects), the resource descriptor (JID, presence, instance ID) is stored in etcd. Any node can look up which instance owns a given user session.

3. **Stanza Routing** — When a stanza targets a user connected to a different node:
   - The local C2S router queries the distributed resource manager
   - Identifies the target instance ID
   - Forwards the stanza via gRPC to that node's `LocalRouter` service

4. **Stream Management Failover (XEP-0198)** — When a node goes down, stream management queues can be transferred to another node via the `TransferQueue` gRPC call, allowing clients to resume sessions on a different instance.

5. **Component Routing** — External components registered on one node are accessible cluster-wide through the `ComponentRouter` gRPC service.

```
 Node A                        Node B
┌──────────────┐              ┌──────────────┐
│  C2S Router  │───gRPC──────▶│  C2S Router  │
│  S2S Router  │              │  S2S Router  │
│  Components  │              │  Components  │
└──────┬───────┘              └──────┬───────┘
       │                             │
       └──────────┬──────────────────┘
                  │
           ┌──────▼──────┐
           │    etcd      │
           │  (KV Store)  │
           └─────────────┘
```

---

## Modules & XMPP Extensions

### Module System

jackal's module system provides a plugin-style architecture for XMPP extensions. Each module is independently configurable and can be enabled or disabled:

```yaml
modules:
  enabled:
    - roster
    - offline
    - last
    - disco
    - private
    - vcard
    - version
    - caps
    - blocklist
    - stream_mgmt
    - ping
    - time
    - carbons
    - mam

  # Per-module configuration
  offline:
    queue_size: 300

  ping:
    ack_timeout: 90s
    interval: 3m
    send_pings: true
    timeout_action: kill      # kill | none

  mam:
    queue_size: 1500
```

### Supported Specifications

#### Core Standards

| Specification | Description |
|---|---|
| [RFC 6120](https://xmpp.org/rfcs/rfc6120.html) | XMPP Core |
| [RFC 6121](https://xmpp.org/rfcs/rfc6121.html) | XMPP Instant Messaging and Presence |

#### XMPP Extension Protocols (XEPs)

| XEP | Version | Name | Description |
|---|---|---|---|
| [XEP-0004](https://xmpp.org/extensions/xep-0004.html) | 2.9 | Data Forms | Structured form exchange |
| [XEP-0012](https://xmpp.org/extensions/xep-0012.html) | 2.0 | Last Activity | Query user's last activity time |
| [XEP-0030](https://xmpp.org/extensions/xep-0030.html) | 2.5rc3 | Service Discovery | Discover server and user features |
| [XEP-0049](https://xmpp.org/extensions/xep-0049.html) | 1.2 | Private XML Storage | Store private data on the server |
| [XEP-0054](https://xmpp.org/extensions/xep-0054.html) | 1.2 | vcard-temp | User profile (vCard) storage |
| [XEP-0059](https://xmpp.org/extensions/xep-0059.html) | 1.0 | Result Set Management | Paginate large result sets |
| [XEP-0092](https://xmpp.org/extensions/xep-0092.html) | 1.1 | Software Version | Query server software version |
| [XEP-0114](https://xmpp.org/extensions/xep-0114.html) | 1.6 | Jabber Component Protocol | External component connections |
| [XEP-0115](https://xmpp.org/extensions/xep-0115.html) | 1.5.2 | Entity Capabilities | Cache and advertise features |
| [XEP-0122](https://xmpp.org/extensions/xep-0122.html) | 1.0.2 | Data Forms Validation | Validate form field data types |
| [XEP-0138](https://xmpp.org/extensions/xep-0138.html) | 2.0 | Stream Compression | zlib stream compression |
| [XEP-0160](https://xmpp.org/extensions/xep-0160.html) | 1.0.1 | Offline Messages | Store messages for offline users |
| [XEP-0190](https://xmpp.org/extensions/xep-0190.html) | 1.1 | Closing Idle Streams | Best practice for idle connection handling |
| [XEP-0191](https://xmpp.org/extensions/xep-0191.html) | 1.3 | Blocking Command | Block/unblock contacts |
| [XEP-0198](https://xmpp.org/extensions/xep-0198.html) | 1.6 | Stream Management | Reliable delivery with session resumption |
| [XEP-0199](https://xmpp.org/extensions/xep-0199.html) | 2.0 | XMPP Ping | Application-level keepalive |
| [XEP-0202](https://xmpp.org/extensions/xep-0202.html) | 2.0 | Entity Time | Query entity's current time |
| [XEP-0220](https://xmpp.org/extensions/xep-0220.html) | 1.1.1 | Server Dialback | S2S identity verification |
| [XEP-0237](https://xmpp.org/extensions/xep-0237.html) | 1.3 | Roster Versioning | Incremental roster synchronisation |
| [XEP-0280](https://xmpp.org/extensions/xep-0280.html) | 0.13.3 | Message Carbons | Multi-device message sync |
| [XEP-0297](https://xmpp.org/extensions/xep-0297.html) | 1.0 | Stanza Forwarding | Forward stanzas between entities |
| [XEP-0313](https://xmpp.org/extensions/xep-0313.html) | 1.0.1 | Message Archive Management | Server-side message history |
| [XEP-0368](https://xmpp.org/extensions/xep-0368.html) | 1.1.0 | SRV Records for XMPP over TLS | Direct TLS via SRV records |

---

## Server Extensibility

jackal supports extending server functionality through two mechanisms:

### External Components (XEP-0114)

Connect external services to jackal using the [Jabber Component Protocol](https://xmpp.org/extensions/xep-0114.html):

```yaml
components:
  secret: "component-shared-secret"
  listeners:
    - port: 5275
```

### gRPC Extension API

jackal exposes gRPC APIs for programmatic integration:

- **Authenticators** — Implement custom authentication backends via the [authenticator proto](https://github.com/jackal-xmpp/jackal-proto/blob/master/jackal/proto/authenticator/v1/authenticator.proto)
- **Admin API** — User management (create, delete, change password) on port `15280`

Proto definitions for the extension APIs are maintained in the [jackal-proto repository](https://github.com/jackal-xmpp/jackal-proto).

---

## Security

jackal implements multiple layers of security:

| Layer | Implementation |
|---|---|
| **Transport** | Mandatory TLS (StartTLS or Direct TLS), TLS 1.3 support |
| **Authentication** | SASL with SCRAM-SHA-1, SCRAM-SHA-256, SCRAM-SHA-512, SCRAM-SHA3-512 |
| **Channel Binding** | `tls-unique` and `tls-exporter` mechanisms to prevent MITM attacks |
| **Password Storage** | SCRAM-derived hashes with configurable iteration count, per-user salt, and versioned pepper keys |
| **Rate Limiting** | Per-user/domain traffic shapers with token bucket algorithm |
| **Session Limits** | Configurable maximum sessions per user |

---

## Monitoring & Observability

### Prometheus Metrics

jackal exposes metrics on the HTTP port (default `6060`):

| Endpoint | Purpose |
|---|---|
| `GET /metrics` | Prometheus metrics (gRPC latencies, storage operations, connection counts, etc.) |
| `GET /healthz` | Health check (returns 200 when healthy) |
| `GET /debug/pprof` | Go pprof profiling endpoints |

### Grafana Dashboard

A pre-built Grafana dashboard is included at [`monitoring/grafana-dashboards/jackal.json`](monitoring/grafana-dashboards/jackal.json). Import it into your Grafana instance to visualise:

- Client-to-Server (C2S) connection metrics
- Server-to-Server (S2S) federation metrics
- Storage operation latencies
- Memory and CPU usage
- Error rates

---

## Development

### Makefile Targets

| Target | Description |
|---|---|
| `make build` | Compile the jackal server binary |
| `make install` | Build and install `jackal` to `$GOPATH/bin` |
| `make installctl` | Build and install `jackalctl` to `$GOPATH/bin` |
| `make test` | Run the full test suite |
| `make check` | Run all quality checks (fmt, vet, lint) |
| `make fmt` | Check Go formatting |
| `make vet` | Run `go vet` for common issues |
| `make lint` | Run the linter |
| `make generate` | Generate mock files via `go generate` |
| `make proto` | Regenerate protobuf Go code |

### Running Tests

```bash
# Run the full test suite
make test

# Run tests for a specific package
go test ./pkg/c2s/... -v

# Run tests with race detection
go test -race ./pkg/...
```

jackal uses [moq](https://github.com/matryer/moq) for generating interface mocks and [testify](https://github.com/stretchr/testify) for assertions. Mocks are generated via `//go:generate` directives.

### Code Generation

```bash
# Generate mocks
make generate

# Generate protobuf code (requires protoc, protoc-gen-go, protoc-gen-go-grpc)
make proto
```

---

## Contributing

The [jackal developer community](https://gitter.im/jackal-im/jackal?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=readme.md) is vital to improving jackal.

Contributions of all kinds are welcome:

- 🐛 Reporting issues
- 📖 Updating documentation
- 🔧 Fixing bugs
- ✅ Improving test coverage
- 💡 Sharing ideas and feature requests

Please read and follow our [Code of Conduct](CODE_OF_CONDUCT.md).

---

## License

jackal is licensed under the **Apache License 2.0**. See [LICENSE](LICENSE) for the full license text.

---

## Contact

If you have any suggestion or question:

**Miguel Ángel Ortuño**
- JID: ortuman@jackal.im
- Email: [ortuman@pm.me](mailto:ortuman@pm.me)
