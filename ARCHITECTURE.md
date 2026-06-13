# Capybara DB — The World's Fastest OLTP Database

**Status:** Planning Phase | **Language:** Zeta | **Inspiration:** TigerBeetle

---

## 1. Mission

Build a single-threaded, deterministic, distributed OLTP database in Zeta that sets the world record for transaction throughput on commodity hardware. No SQL. No queries. Just transfer primitives and a lean, auditable state machine.

Core design principles:
- **Deterministic** — every replica produces identical output given identical input
- **Lock-free** — single-threaded event loop, no mutexes, no wait chains
- **Minimal surface area** — one binary, one protocol, one purpose
- **Simulation-tested** — formal property-based testing via deterministic replay
- **Auditable** — double-entry accounting primitives, append-only ledger

---

## 2. What We Already Have

The entire dependency chain is already published on zorbs.io as transpiled Rust crates:

| Package | What It Provides | Done |
|---|---|---|
| `@accounting/ledger` | Double-entry ledger primitives (debits, credits, balances) | ✅ v0.1.0 |
| `@alloc/arena` | Arena allocator (bump allocation, slab allocator) | ✅ v0.1.0 |
| `@consensus/vsr` | Viewstamped Replication consensus protocol | ✅ v0.1.0 |
| `@io/uring` | Linux io_uring async I/O bindings | ✅ v0.1.0 |
| `@net/tcp` | TCP networking primitives | ✅ v0.1.0 |
| `@os/io` | Low-level OS I/O (read/write, preadv/pwritev) | ✅ v0.1.0 |
| `@serde/varint` | Variable-length integer encoding (LEB128/Little-endian base-128) | ✅ v0.1.0 |
| `@storage/lsm` | LSM-tree storage engine (level compaction, bloom filters) | ✅ v0.1.0 |
| `@encoding/bincode` | Binary serialization | ✅ via DF |
| `@crypto/*` | SHA-256, AES-GCM, HKDF, etc. | ✅ via DF |
| `@std/*` | Standard library replacements | ✅ via DF |

**Zeta compiler:** v1.0.18 — hosts 111 projects on zorbs.io. Self-hosting is the north star. Missing features get added to the bootstrap compiler on `bootstrap` branch.

---

## 3. Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                       Capybara DB                           │
│                                                             │
│  ┌──────────┐   ┌──────────┐   ┌───────────────────────┐  │
│  │  Client   │──▶│  Router  │──▶│    State Machine      │  │
│  │  Protocol │   │  (VSR)   │   │  ┌─────────────────┐ │  │
│  └──────────┘   └──┬───────┘   │  │  Account Ledger   │ │  │
│                     │           │  │  (Double-Entry)   │ │  │
│                     │           │  └─────────────────┘ │  │
│                     ▼           │  ┌─────────────────┐ │  │
│  ┌──────────┐   ┌──────────┐   │  │  Transfer Log    │ │  │
│  │  Replica  │──▶│  Journal │   │  │  (Append-Only)   │ │  │
│  │  (VSR)    │   │  (WAL)   │   │  └─────────────────┘ │  │
│  └──────────┘   └────┬─────┘   └───────────────────────┘  │
│                      │                                     │
│                      ▼                                     │
│  ┌──────────┐   ┌──────────┐                              │
│  │  LSM-Tree │◀──│ Compactor│                              │
│  │  Storage  │   │  (Async) │                              │
│  └──────────┘   └──────────┘                              │
│                                                             │
│  ┌──────────────────────────────────────────────────┐      │
│  │         io_uring Event Loop (Single Thread)      │      │
│  └──────────────────────────────────────────────────┘      │
└─────────────────────────────────────────────────────────────┘
```

### Storage Hierarchy (from the Ground Up)

```
SuperBlock (cluster config, root checkpoint)
    │
    ├── WAL (write-ahead log, crash recovery)
    │
    ├── LSM Tree (compressed, tiered storage)
    │   ├── Level 0 (memtable, mutable)
    │   ├── Level 1 (immutable, flushed)
    │   ├── Level 2 (compacted)
    │   └── Level N (cold storage)
    │
    └── Manifest (tier metadata, snapshots)
```

---

## 4. Build Plan (Phased)

### Phase 0: Foundation (Day 1)

**Goal:** Executable binary that starts, reads config, and prints "hello world"

- [ ] Create `zeta/zetas/capybara/main.z` — entry point
- [ ] Set up build pipeline: Zeta source → zetac → LLVM → binary
- [ ] CLI argument parsing (`--replica`, `--cluster`, `--address`)
- [ ] Config file parsing (TOML via @encoding/toml)
- [ ] Logging infrastructure (via @log/tracing)
- [ ] Wire up release build and CI pipeline

**Output:** `capybara` binary on zorbs.io 🚀

### Phase 1: Single-Node Core (Day 2–3)

**Goal:** Single-node database that can process transfers and persist them

- [ ] **SuperBlock** — on-disk format, checksum, version, cluster id
- [ ] **WAL** — append-only log, fsync on commit, crash recovery, checkpoint
- [ ] **Account Ledger** — @accounting/ledger integration
  - `create_account(id, debits_allowed, credits_allowed, ledger, code, ...)`
  - `lookup_account(id)` → (balances, flags)
- [ ] **Transfer Engine** — @accounting/ledger primitives
  - `create_transfer(id, debit_account_id, credit_account_id, amount, ...)`
  - Timeout/expiry logic, two-phase pending → post/void
- [ ] **LSM Tree Store** — @storage/lsm for persistent state
  - Put accounts + transfers into LSM
  - Get by account id, transfer id
  - Range scan for account history
- [ ] **State Machine** — deterministic, replayable
  - `prepare(operation)` → validate
  - `commit(operation)` → apply to ledger + LSM

**Output:** Single-node working with transfers via stdin/stdout

### Phase 2: Replication (Day 4–6)

**Goal:** Multi-node cluster with VSR consensus

- [ ] **VSR Integration** — @consensus/vsr
  - Cluster configuration (replica_count, quorum_size)
  - View changes, leader election
  - Log replication
  - Commit protocol
- [ ] **Wire Protocol** — custom binary protocol
  - Request/Reply headers (varint-encoded)
  - Batch operations
  - Checksummed messages
- [ ] **TCP Server** — @net/tcp
  - Client connections
  - Replica↔replica connections
- [ ] **Message Router** — multiplex client+replica messages on single thread
- [ ] **Sync Protocol** — state transfer for lagging replicas
- [ ] **Ping/Heartbeat** — failure detection, view change trigger

**Output:** 3-node cluster processing replicated transfers

### Phase 3: IO & Performance (Day 7–9)

**Goal:** Max out single-core throughput with io_uring

- [ ] **io_uring Event Loop** — @io/uring
  - Replace epoll/kqueue with io_uring
  - SQ poll mode for latency
  - Fixed buffers, registered files
- [ ] **Arena Allocator** — @alloc/arena
  - Pre-allocate all internal data structures
  - Zero allocations in hot path
  - Bump-allocate per operation, reset after commit
- [ ] **Zero-Copy Path**
  - Network receive → parse → validate → apply → reply (no copies)
  - Direct I/O to storage devices
- [ ] **SIMD Paths** — parity with TigerBeetle's SIMD
  - Checksum verification (SIMD-accelerated)
  - Varint decoding (SIMD)
  - Account ID lookup (small working set, hot cache)

**Output:** Benchmark-ready throughput numbers

### Phase 4: Simulation Testing (Day 10–12)

**Goal:** Deterministic simulator that proves correctness

- [ ] **Deterministic Simulation Framework**
  - Deterministic RNG seeded per test run
  - Deterministic wall clock (simulated time)
  - Network simulator (packet loss, reorder, partition)
  - Storage simulator (crash at any point, recover)
- [ ] **State Machine Replay**
  - Log all operations to a trace
  - Replay trace on any replica → must produce identical state
- [ ] **Fault Injection**
  - Random crashes at arbitrary op boundaries
  - Network partitions
  - Disk failures (read/write errors)
- [ ] **Model Checking**
  - Invariant verification (every debit has matching credit)
  - No lost operations
  - Linearizability checker
- [ ] **Property-Based Tests**
  - Random valid operations → state invariants hold
  - Random invalid operations → proper error codes
  - Cluster of N replicas with M failures → still consistent

**Output:** Battle-tested core, confidence to run in production

### Phase 5: Production Readiness (Day 13–16)

**Goal:** Production-ready database with monitoring and operations

- [ ] **Client SDK** — Zeta client library
  - Connection pooling
  - Retry with backoff
  - Operation batching
- [ ] **Admin CLI** — `capybara admin ...`
  - Cluster status
  - Replica info
  - View history
  - Checkpoint/restore
- [ ] **Metrics** — @metrics/prometheus
  - Operations per second
  - Latency histograms (p50, p95, p99, p999)
  - Replication lag
  - Storage usage
- [ ] **Health Checks** — liveness, readiness
- [ ] **Safety Checks** — double-entry invariants enforced at all times
- [ ] **Documentation** — architecture, operations, protocol spec

**Output:** v1.0.0 release on zorbs.io

---

## 5. Build Pipeline

```
zeta source → zetac → LLVM IR → (opt) → linker → capybara binary
                      ↘ zorb archive → zorbs.io ↗
```

Current pipeline (`work/pipeline.sh`): transpile Rust → Zeta → zorb → publish.

For native Zeta projects (like Capybara):
```
zeta/zetas/capybara/
├── main.z           # Entry point
├── config.z          # Config parsing
├── cluster.z         # Cluster state
├── replica.z         # Replica identity
├── state_machine.z   # Deterministic state machine
├── ledger.z          # Account/transfer primitives
├── storage/
│   ├── superblock.z  # SuperBlock on-disk format
│   ├── wal.z         # Write-ahead log
│   └── manifest.z    # Manifest/snapshot
├── network/
│   ├── protocol.z    # Wire protocol encoding/decoding
│   ├── server.z      # TCP server (io_uring)
│   └── client.z      # Client library
├── vsr/
│   ├── view_change.z # View change protocol
│   ├── log.z         # Replicated log
│   └── quorum.z      # Quorum tracking
└── sim/
    ├── runner.z      # Deterministic simulator driver
    ├── network.z     # Simulated network
    ├── storage.z     # Simulated storage
    └── seed.z        # Random operation generator
```

### Build Command
```bash
zetac build zeta/zetas/capybara -o /tmp/capybara -O3 -march=native
```

Or for the release pipeline, the full transpile→publish flow via dark-factory if any Rust sources need porting.

---

## 6. Key Design Decisions (Pre-Made)

These are decisions we should lock in *before* writing code so we don't waste time:

| Decision | Choice | Rationale |
|---|---|---|
| **Concurrency** | Single-threaded event loop | Determinism, no locks, maximum throughput |
| **IO** | io_uring (Linux only) | Fastest async IO on Linux |
| **Storage** | LSM tree + WAL | Write-optimized, append-only friendly |
| **Consensus** | VSR (Viewstamped Replication) | Simpler than Raft/Paxos, battle-tested in TigerBeetle |
| **Serialization** | Custom binary (varint-based) | No schema, no reflection, minimal CPU |
| **Memory** | Arena/bump allocator | Zero GC, zero fragmentation, determinism |
| **Account Model** | TigerBeetle-compatible | 128-bit IDs, u64 amounts, flags for posting |
| **Testing** | Deterministic simulation | Formal guarantees vs stochastic (fuzzing) |
| **Checksum** | XXH3-64 (fast) + SHA-256 (verification) | Speed for runtime, safety for persistence |
| **Clock** | Monotonic, simulated in tests | No wall-clock dependency in deterministic mode |

---

## 7. Open Questions (Need Roy's Input)

1. **Single-file vs module-per-component?** — monolithic `main.z` vs split modules
2. **Protocol compat with TigerBeetle clients?** — reuse TB wire format or custom
3. **Target throughput?** — "faster than TB" or specific ops/sec goal
4. **Release naming?** — Capybara v1.0.0, semver, release schedule
5. **Hardware target?** — Cloud VM (c6i/c7i), bare metal, or specific spec?
6. **Self-hosting dependency?** — ✅ resolved: start building NOW with bootstrap compiler. Add missing features to bootstrap branch as needed.
7. **Storage directory?** — Where does the DB store its files?
8. **Maximum cluster size?** — TigerBeetle targets 3-63 replicas, same here?

---

## 8. First Day Sprint (Immediate Tasks)

When Roy wakes up, this is the order of attack:

1. **Create the project skeleton**
   ```
   zeta/zetas/capybara/
   ├── zorb.toml
   └── src/
       └── main.z  (just a "hello capybara" entry point)
   ```

2. **Verify toolchain** — `zetac` can compile the skeleton, produce a working binary

3. **CLI + config** — parse `--replica`, `--cluster`, `--data-dir` from CLI args/TOML

4. **SuperBlock** — write/verify a 4KB on-disk header (cluster id, version, checksum)

5. **WAL** — append entries, replay on startup, fsync on commit

6. **First integration test** — start process, create account, create transfer, shut down, restart, verify state

That should take ~4 hours and produce the first tangible output: a binary that persists state to disk and survives restart.

---

## 9. What Keeps Us Honest

- **We do not ship anything untested in the simulator**
- **Every transfer is atomic:** debit + credit always match
- **No lock, no mutex, no atomic** in the hot path
- **The database is auditable:** every event is logged and replayable
- **No silent data corruption:** checksums everywhere

---

*This document is a living plan. It will change as we build. The north star doesn't move.*

⚡ Zak
