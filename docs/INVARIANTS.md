# kv-ledger — Safety & Correctness Invariants (v1)

This document specifies invariants that must hold at all times.
If any invariant is violated, the implementation is incorrect.

---

## Log Authority Invariants

1. The replicated log is the only source of truth.
2. No application-visible state may exist outside the log.
3. The in-memory state machine is a pure function of committed log entries.

---

## Application Order Invariants

4. Log entries are applied strictly in increasing index order.
5. An entry at index i is applied iff:
   - i == last_applied + 1
   - i <= commit_index
6. Each committed log entry is applied exactly once.
7. No entry with index > commit_index may affect state.

---

## State Machine Invariants

8. last_applied is monotonically increasing.
9. At all times:
   last_applied <= commit_index
10. The kv map reflects exactly the result of applying log entries
    with indices [1 .. last_applied].

---

## Visibility Invariants

11. GET must never observe the effects of uncommitted log entries.
12. GET must never observe the effects of unapplied log entries.
13. A successful PUT or DELETE is visible to GET only after commitment
    and application.

---

## Delete (Tombstone) Invariants

14. DELETE removes a key from kv if present.
15. DELETE on a non-existent key is a no-op.
16. DELETE is idempotent.
17. After replay, deleted keys are absent from kv.

---

## Crash Safety Invariants

18. After any crash and restart, replaying log entries
    [1 .. commit_index] reconstructs the same kv state.
19. Crash recovery does not require network communication.
20. Uncommitted entries must not affect post-restart state.

---

## Determinism Invariants

21. Applying the same committed log entries in the same order
    always produces the same kv state.
22. State transitions must not depend on:
    - timestamps
    - randomness
    - memory layout
    - map iteration order

---

## Scope Boundaries

23. kv-ledger does not infer commitment.
24. kv-ledger does not repair logs.
25. kv-ledger does not persist derived state.
26. kv-ledger does not provide availability guarantees under leader failure.

---

## Final Rule

If the observed behavior of the system contradicts any invariant
in this document, the implementation is wrong — not the specification.
