# Capybara DB v1.0.0 — Initial Release

<p align="center">
  <img src="assets/capy128.png" alt="Capybara DB" width="128" height="128">
</p>

**June 13, 2026** · Written in pure Zeta · 4,109 lines of source

---

## What's New

This is the first public release of **Capybara DB** — a single-node, crash-safe, networked OLTP database server built entirely in the Zeta programming language. It started as a Phase 0 sketch on June 7 and shipped as a working event-loop server with replication on June 13.

**6 days from first commit to v1.0.0.**

---

## Features

### 📊 Double-Entry Accounting State Machine
- Create accounts with configurable flags (debits allowed, credits allowed, ledger, code)
- Atomic transfers between any two accounts (128-bit IDs)
- Credit operations (direct deposit)
- Two-phase commit: create pending → post / void with timeout
- Batch operations: bulk create (up to 1000 accounts) and bulk transfer
- O(1) hash-indexed account lookups (1000 accounts, 1000 pending transfers)

### 💾 Crash Recovery
- **SuperBlock:** 4KB header, persistence check on startup
- **WAL:** 128-byte structured entries with xxHash64 checksums
- **Crash recovery:** replay WAL → bit-identical state
- **Fsync** on every commit
- Stalls to minimum `O(N)` replay (just one linear pass over entries)

### 🔄 VSR Replication
- Full Viewstamped Replicated State Machine protocol
- 3-replica cluster (1 primary + 2 backups)
- Heartbeat keepalive (1s interval)
- View change (3s timeout)
- Log sync and catch-up for stale replicas
- Persistent TCP connections with reconnect

### 🔧 io_uring Event Loop
- Native io_uring init, SQE submission, CQE completion
- 128-entry ring buffer
- Accept, read, and write operations
- Graceful fallback to blocking I/O if io_uring unavailable
- Single-threaded: zero mutex contention

### 🌲 LSM-Tree Storage
- Log-structured merge tree with size-tiered compaction
- 4 levels, up to 8 SSTables per level
- Memtable (in-memory write buffer)
- SSTable format (read/write/scan)
- Bloom filter for point lookups
- Periodic flush from ledger (every 5s)
- On-disk manifest management

### 🧪 Deterministic Simulation Testing
- Separated from server binary (`capybara_sim`)
- xoshiro128** deterministic PRNG (seeded, reproducible)
- 10,000 random operations per run
- 5 invariant checks:
  - **I1:** Every debit has a matching credit
  - **I2:** No negative balances unless debits_allowed
  - **I3:** Pending transfers created/posted/voided exactly once
  - **I4:** Valid account flag state transitions
  - **I5:** Consistent ledger+code combinations

### 🌐 Network Protocol
- Raw TCP, framed message format
- Session mode: (client_id, request_number) dedup
- Legacy mode: single operation per connection
- Batch operations for bulk throughput
- Errno-style error codes
- Zero dependencies beyond kernel syscalls

---

## Source Code Metrics

| Module | Lines | Purpose |
|--------|-------|---------|
| `os.z` | 306 | OS abstraction, io_uring, networking |
| `runtime.z` | 308 | SuperBlock, WAL, crash replay |
| `ledger.z` | 324 | State machine, accounts, transfers |
| `main.z` | 587 | Event loop, client handler, CLI |
| `vsr.z` | 503 | VSR consensus protocol |
| `lsm.z` | 407 | LSM-Tree storage engine |
| `sstable.z` | 222 | SSTable format |
| `memtable.z` | 180 | In-memory write buffer |
| `manifest.z` | 225 | Manifest management |
| `bloom.z` | 72 | Bloom filter |
| `sim.z` | 425 | Simulation framework |
| `xxh64.z` | 64 | xxHash64 checksum |
| `client_session.z` | 80 | Dedup state tracking |
| `config.z` | 146 | Configuration defaults |
| **Total** | **4,109** | — |

---

## Quick Start

```bash
# Assemble and compile
./build.sh                          # → /tmp/capybara
./build_sim.sh                      # → /tmp/capybara_sim

# Run single replica
/tmp/capybara --replica 0           # serves on port 8700

# Run simulation
/tmp/capybara_sim                   # 10K random ops, invariant checks
```

---

## Known Limitations

1. **Flat file structure** — workaround for a compiler bug where inline modules eat content after the first closing brace
2. **No TLS/encryption** — intended for trusted networks only
3. **Fixed-size state** — 1000 accounts, 1000 pending transfers (configurable, but set at compile time)
4. **No SQL, no query language** — Capybara is a transaction server, not a relational database
5. **Single-threaded by design** — no cluster-internal parallelization (VSR replication across nodes only)
6. **Data directory hard-coded** to `/tmp/capybara_data` (configurable via source edit for now)

---

## Compiler Bugs Discovered

Building Capybara DB uncovered 7 bugs in the Zeta compiler:

1. Empty function body → External linkage (builtins vs user fns)
2. Inline module parser eats content after first `}`
3. Name mangling `::` vs `__` inconsistent across modules
4. Struct/tuple returns produce garbage field values
5. Const string loses pointer to extern C
6. `print!` macro strips format strings
7. Module separator convention conflicts

All workarounds applied. These will be fixed on the `bootstrap` branch before v1.1.

---

## Credits

- **Roy Murphy** — Creator of Zeta, architecture, logo
- **Zak Murphy** — Implementation, simulation framework, release

---

## Links

- [Zeta Compiler](https://github.com/murphsicles/PrimeZeta)
- [Capybara DB Repository](https://github.com/murphsicles/CapybaraDB)
- [Zeta Language](https://zeta-lang.org) (coming soon)

---

<p align="center">
  <img src="assets/capy72.png" width="32" height="32">
  <br>
  <em>Chasing the singularity, one transaction at a time.</em>
</p>
