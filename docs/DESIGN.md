# kv-ledger — Design (v1)

## Purpose

kv-ledger is a replicated key-value store built on top of ledger v1.

The system demonstrates deterministic state machine replication by deriving
all application-visible state from a replicated, crash-safe append-only log.

The focus is on correctness, crash recovery, and explicit semantics — not on
performance, availability, or feature completeness.

---

## Relationship to ledger

- ledger is treated as a trusted replication and durability layer
- kv-ledger does not modify or depend on ledger internals
- All application-visible state is derived exclusively from the ledger log
- The log is the single source of truth

---

## System Model

- Fixed-size cluster (3–5 nodes)
- Single leader (assumed; no leader election in v1)
- Clients interact only with the leader
- Followers replicate log entries but do not serve reads
- Communication via raw TCP
- Nodes may crash and restart with disk intact
- Network may drop, delay, duplicate, or reorder messages

---

## Client API (v1)

### PUT

PUT(key, value) → success | error

- Sent to the leader
- Encoded as a log entry
- Applied only after commitment
- Success implies durability and future visibility

---

### DELETE

DELETE(key) → success | error

- Sent to the leader
- Encoded as a tombstone log entry
- Applied only after commitment
- Removes the key from the derived state if present

---

### GET

GET(key) → value | not_found

- Served only by the leader
- Reads from the in-memory state machine
- Observes only committed and applied state

There is no distinction between:
- a key that was never written
- a key that was deleted

Both return not_found.

---

## Non-Goals (v1)

- Conditional updates (CAS)
- Range scans
- Transactions
- Reads from followers
- Snapshots or compaction
- Leader election
- High availability under leader failure
- Performance optimizations
- Log garbage collection

---

## State Model

### Source of Truth

The replicated log maintained by ledger is the only authoritative state.

The in-memory key-value map is a derived cache and is never persisted directly.

---

### State Machine State

The state machine maintains the following in-memory variables:

- kv : map[string][]byte
- last_applied : uint64

Initial state:

- kv is empty
- last_applied = 0

---

### Log Payload Format

Each log entry payload represents exactly one state machine operation.

Conceptual format:

[ op_type | key_len | value_len | key | value ]

Where:

- op_type (v1): PUT, DELETE
- key_len, value_len: fixed-width integers
- key, value: raw bytes
- value is omitted for DELETE

Malformed or unknown payloads indicate a fatal bug.

---

## State Machine Semantics

### Inputs

The only inputs to the state machine are committed log entries of the form:

(index, payload)

---

### Application Rule

A log entry at index i may be applied iff:

- i == last_applied + 1
- i <= commit_index

Entries are applied strictly in increasing index order.

Each entry is applied exactly once.

---

### Transition Function

For an entry at index i:

PUT(key, value):
- kv[key] = value
- last_applied = i

DELETE(key):
- remove key from kv if present
- no-op if absent
- last_applied = i

DELETE operations are idempotent.

---

### Visibility Rules

- Only entries with index <= commit_index may affect state
- Uncommitted entries must never affect kv
- A GET observes only the effects of applied entries

---

## Replay & Recovery Semantics

On startup or restart, the state machine is rebuilt solely from the log.

Recovery proceeds as follows:

1. Initialize state:
   - kv = empty
   - last_applied = 0

2. Open the log and read the durable commit_index.

3. For each log index i from 1 to commit_index, in increasing order:
   - Read log entry (i, payload)
   - Apply the decoded operation
   - Set last_applied = i

4. After recovery:
   - last_applied == commit_index
   - kv reflects all committed log entries

Entries with index > commit_index are ignored during recovery.

No network communication is required during recovery.

---

## Determinism Requirement

Applying the same sequence of committed log entries in the same order must always
produce the same kv state.

The state machine must not:
- use timestamps
- use randomness
- depend on map iteration order

---

## Responsibilities of kv-ledger

- Encode and decode KV operations
- Maintain the in-memory state machine
- Apply committed log entries exactly once
- Expose PUT / DELETE / GET client interface
- Handle crash recovery via log replay

---

## What kv-ledger Does Not Do

- Infer commitment
- Persist derived state
- Repair the log
- Manage replication
- Guarantee availability under leader failure

---

## Design Philosophy

- The log is law
- State is derived, not stored
- Correctness over performance
- Explicit semantics over implicit behavior
- If behavior contradicts this document, the code is wrong
