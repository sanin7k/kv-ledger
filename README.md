# kv-ledger

**kv-ledger** is a minimal replicated key-value store built on top of **ledger v1**.

It demonstrates **deterministic state machine replication**, where all
application-visible state is derived exclusively from a replicated,
crash-safe, append-only log.

The project prioritizes **correctness, crash recovery, and explicit semantics**
over performance, availability, or feature completeness.

---

## Core Idea

The replicated log is the **single source of truth**.

The in-memory key-value map is a **derived cache** that is never persisted
directly and can always be rebuilt by replaying committed log entries.

If the log is correct, the system is correct.

---

## Scope (v1)

- Fixed-size cluster with a single assumed leader
- PUT and DELETE operations encoded as log entries
- GET served from the leader’s applied state
- Crash recovery via full log replay
- Deterministic, replayable state machine

---

## Non-Goals

- Leader election
- Reads from followers
- Transactions or CAS
- Snapshots or compaction
- Log garbage collection
- High availability under leader failure
- Performance optimizations

---

## Documentation

State machine behavior, log semantics, and recovery rules are specified in:

- `docs/DESIGN.md`

Safety and correctness properties are specified in:

- `docs/INVARIANTS.md`

If the code contradicts either document, **the code is wrong**.

---

## Status

**Design frozen for v1.**
