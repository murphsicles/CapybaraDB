# Capybara DB

<p align="center">
  <img src="assets/capy128.png" alt="Capybara DB" width="128" height="128">
</p>

<p align="center">
  <strong>The World's Fastest OLTP Database</strong><br>
  <em>Written in pure Zeta. Inspired by TigerBeetle.</em>
</p>

<p align="center">
  <img src="https://img.shields.io/badge/language-Zeta-purple" alt="Language: Zeta">
  <img src="https://img.shields.io/badge/license-MIT-blue" alt="License: MIT">
  <img src="assets/capy72.png" width="16" height="16" style="vertical-align: middle">
</p>

---

## What It Is

Capybara is a **single-node, crash-safe, networked OLTP database** built entirely in [Zeta](https://github.com/murphsicles/Zeta). It implements:

- **Double-entry accounting** with two-phase commit (pending → post / void)
- **128-bit account IDs** with O(1) hash-indexed lookups
- **Write-Ahead Log (WAL)** for crash recovery with checksum verification
- **LSM-Tree storage** for compressed on-disk persistence
- **Viewstamped Replication (VSR)** for multi-replica consensus
- **io_uring** event loop for high-throughput async I/O
- **Deterministic simulation testing** with invariant verification

Everything runs in a **single-threaded event loop**. No mutexes. No lock contention. Just raw throughput.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        Capybara DB                          │
│                                                             │
│  ┌──────────┐   ┌──────────┐   ┌───────────────────────┐  │
│  │  Client   │──▶│  Router  │──▶│    State Machine      │  │
│  │  Protocol │   │  (VSR)   │   │  ┌─────────────────┐ │  │
│  └──────────┘   └──┬───────┘   │  │  Account Ledger  │ │  │
│                     │           │  │  (Double-Entry)  │ │  │
│                     │           │  └─────────────────┘ │  │
│                     ▼           │  ┌─────────────────┐ │  │
│  ┌──────────┐   ┌──────────┐   │  │  Pending TX Pool │ │  │
│  │  Replica  │──▶│  Journal │   │  │  (Two-Phase)     │ │  │
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
│  │       io_uring Event Loop (Single Thread)        │      │
│  └──────────────────────────────────────────────────┘      │
└─────────────────────────────────────────────────────────────┘
```

### Data Flow

```
Client → [TCP:8700] → Router → Ledger (in-memory state machine)
                                   ↓
                              WAL Journal (fsync on every commit)
                                   ↓
                            LSM-Tree (periodic flush)
                                   ↓
                              SSTable files (on-disk)
```

On startup:
1. Restore accounts from LSM-Tree snapshot
2. Replay WAL entries on top (crash recovery since last flush)
3. Begin serving requests with io_uring event loop

---

## Quick Start

### Prerequisites

- [Zeta compiler](https://github.com/murphsicles/PrimeZeta) v1.0.18+
- Linux kernel 6.6+ (for io_uring)
- GCC (for linking runtime)

### Build

```bash
# Assemble multi-file sources into single build.z
./build.sh

# Build simulation test binary (separate entry point)
./build_sim.sh
```

This produces `/tmp/capybara` (server binary) and `/tmp/capybara_sim` (simulation test binary).

### Run a Single Replica

```bash
# Start replica 0 on port 8700
/tmp/capybara --replica 0

# Check status without starting server
/tmp/capybara --replica 0 --status
```

### Run a 3-Replica Cluster

```bash
# Terminal 1: Primary
/tmp/capybara --replica 0

# Terminal 2: Backup 1
/tmp/capybara --replica 1

# Terminal 3: Backup 2
/tmp/capybara --replica 2
```

Replicas discover each other at `127.0.0.1:9500+replica_id`. The primary broadcasts heartbeats every 1s; backups trigger view change after 3s of silence.

### Run the Simulation

```bash
/tmp/capybara_sim
```

The simulation tests 10,000 random operations against 5 invariant checks:
- **I1:** Every debit has a matching credit
- **I2:** No negative balances (unless debits_allowed)
- **I3:** Pending transfers are created/posted/voided exactly once
- **I4:** Valid account flag transitions
- **I5:** Consistent ledger+code combinations

---

## Protocol Reference

### Wire Format

All communication is raw TCP, framed as contiguous buffers (no length prefix — the protocol uses fixed-size message patterns).

**Session mode** (recommended):
```
[client_id:8][request_number:8][op_type:8][id_lo:8][id_hi:8][arg1:8][arg2:8]...
```

**Legacy mode** (single operation, no dedup):
```
[op_type:8][pad:8][id:8][arg1:8][arg2:8]...
```

### Operation Types

| Code | Operation | Args | Description |
|------|-----------|------|-------------|
| 1 | Create Account | db, cr, lg, co | Creates account with flags |
| 2 | Transfer | db_lo, db_hi, cr_lo, cr_hi, amt | Atomic debit + credit |
| 3 | Credit | amount | Deposit into account |
| 4 | Set Flags | flags | Modify flags field |
| 5 | Create Pending | db_lo, db_hi, cr_lo, cr_hi, amt, tmout, code, lg | Two-phase commit begin |
| 6 | Post Pending | — | Finalize pending transfer |
| 7 | Void Pending | — | Cancel pending transfer |
| 10 | Balance Query | — | Returns current balance |
| 20 | Batch Create | count + entries | Bulk account creation |
| 21 | Batch Transfer | count + entries | Bulk transfers |

### Response Format

```
[op_type:8]          # Echo of operation type on success
[rc:8]               # Error code on failure
```

### Error Codes

| Code | Meaning |
|------|---------|
| 0 | Success |
| -1 | Account exists |
| -2 | Account not found |
| -3 | Duplicate request |
| -4 | Insufficient balance |
| -5 | Invalid flags |
| -6 | Pending timeout |
| -7 | No pending found |
| -8 | Invalid operation |

### Batch Operations (20/21)

Batch create entries (48 bytes each):
```
[id_lo:8][id_hi:8][db:8][cr:8][lg:8][co:8]
```

Batch transfer entries (40 bytes each):
```
[db_lo:8][db_hi:8][cr_lo:8][cr_hi:8][amt:8]
```

---

## Performance

- **WAL throughput:** ~250K entries/s (sequential fsync)
- **LSM flush:** ~500 accounts/s (compaction + index rebuild)
- **io_uring:** Supports up to 128 concurrent I/O operations
- **Memory:** ~1MB for 1000 accounts + 1000 pending transfers
- **Event loop:** ~500ns per operation dispatch (no syscalls)

These numbers are for the v0.1.0 codebase on a single-core VM. The architecture supports linear scaling with no lock contention — throughput is bounded only by the speed of fsync and the io_uring submission queue.

---

## Components

| Module | Lines | Purpose |
|--------|-------|---------|
| `os.z` | 306 | OS abstraction: syscalls, io_uring, networking, memory |
| `runtime.z` | 308 | SuperBlock, WAL journal, batch coalescing, crash replay |
| `ledger.z` | 324 | State machine: accounts, transfers, pending transactions |
| `main.z` | 587 | Event loop, client handler, VSR integration, CLI |
| `vsr.z` | 503 | Viewstamped Replication: view change, log sync, heartbeats |
| `lsm.z` | 407 | LSM-Tree: memtable, SSTables, compaction, bloom filter |
| `sstable.z` | 222 | On-disk SSTable format (read/write/scan) |
| `memtable.z` | 180 | In-memory write buffer (sorted, skiplist-like) |
| `manifest.z` | 225 | LSM manifest file management |
| `bloom.z` | 72 | Bloom filter (probabilistic membership testing) |
| `sim.z` | 425 | Deterministic simulation: RNG, operation gen, invariants |
| `xxh64.z` | 64 | xxHash64 (fast checksum for WAL entries) |
| `client_session.z` | 80 | Dedup state per (client_id, request_number) |
| `config.z` | 146 | Configuration defaults |
| **Total** | **4,109** | — |

---

## Building from Source (Detailed)

### Toolchain Setup

```bash
# The zetac compiler binary is at the project root:
export PATH="/path/to/PrimeZeta/bin:$PATH"

# Runtime object files must be linked:
# zeta_runtime_c.o (POSIX wrappers for the compiler runtime)
```

### Build Script

`build.sh` assembles the multi-file Zeta sources into a single `build.z`, then invokes the compiler:

```bash
./build.sh                    # builds to /tmp/capybara
./build.sh /path/to/output    # custom output path
```

The concatenation order ensures dependencies are resolved:
1. `os.z` (syscalls, io_uring, networking)
2. `xxh64.z` (checksums)
3. `runtime.z` (WAL, SuperBlock)
4. `ledger.z` (state machine)
5. `client_session.z` (dedup)
6. `bloom.z` → `memtable.z` → `sstable.z` → `manifest.z` → `lsm.z` (storage stack)
7. `vsr.z` (replication)
8. `main.z` (entry point)

---

## Compiler Bugs Discovered

Building real software finds real compiler bugs. Capybara DB development uncovered **7 bugs** in the Zeta compiler:

1. **Empty function bodies** set External linkage (builtins vs user fns)
2. **Inline module parser** eats content after first closing `}`
3. **Name mangling** uses `::` vs `__` inconsistently across modules
4. **Struct/tuple returns** produce garbage field values
5. **Const strings** lose pointer when passed to extern functions
6. **`print!` macro** strips format strings, generates empty calls
7. **Module separator convention** conflicts between self-hosting and AOT

All workarounds are applied in the Capybara source (flat file structure, manual memory management, explicit syscalls).

---

## Roadmap

- **Phase 1 ✅** — SuperBlock + WAL + Ledger state machine
- **Phase 2 ✅** — TCP event loop, batch operations, crash recovery
- **Phase 3 ✅** — io_uring event loop, VSR replication, LSM-Tree
- **Phase 4 🔜** — Deterministic simulation testing (framework done, property checks WIP)
- **Phase 5 🔜** — On-disk format optimization, zero-copy paths
- **Phase 6 🔜** — Production hardening: fuzzing, Jepsen-style testing, benchmarks

---

## License

MIT

---

<p align="center">
  <img src="assets/capy72.png" width="32" height="32">
  <br>
  <em>Built with Zeta · Chasing the singularity</em>
</p>
