# 18 — Concurrency Control

## 1. Why This Exists

Two recovered, committed transactions can still produce the wrong answer. If both read balance 500, subtract 100, and write 400, one deduction is lost. Another transaction might read Alice after a debit but before the matching credit and observe a state that later aborts. These are interleaving failures, not crash failures.

## 2. Mental Model

Locks make conflicting operations wait:

```text
shared (S): read
exclusive (X): write

S with S -> compatible
S with X -> conflict
X with S -> conflict
X with X -> conflict
```

Strict two-phase locking gives a transaction the locks it needs and holds write locks until commit or abort. If all conflicting actions follow the rule, the resulting history is equivalent to some serial order and other transactions cannot read uncommitted writes.

## 3. Storage / Execution Model

Lock individual record ids for this tutorial. Before reading, request S; before writing, request X. A transaction may upgrade its own S to X only when no other holder exists. On commit or abort, release all locks. Waiting transactions form FIFO queues per resource.

Record locks do not protect a predicate like “all users aged 30”: another transaction can insert a new matching row, a phantom. Table locks could prevent it at low concurrency; predicate and range locking are beyond the tiny manager.

## 4. Representing It in OCaml

```ocaml
type mode = Shared | Exclusive
type resource = Record of record_id | Table of table_id

type granted = { tx : txn_id; mutable mode : mode }
type request = { tx : txn_id; mode : mode; wake : unit Condition.t }

type entry = {
  mutable holders : granted list;
  waiting : request Queue.t;
}

type lock_manager = {
  mutex : Mutex.t;
  table : (resource, entry) Hashtbl.t;
}
```

One mutex protects lock-manager metadata only. Release it while a transaction waits on a condition, then recheck compatibility in a loop after waking.

## 5. Build It

Define compatibility against other transactions:

```ocaml
let compatible holders tx requested =
  List.for_all (fun held ->
    held.tx = tx ||
    match held.mode, requested with
    | Shared, Shared -> true
    | _ -> false) holders
```

`acquire` locks the manager mutex, creates an entry, and grants immediately only if compatible and no earlier waiter deserves priority. Otherwise it enqueues and waits. `release_all tx` removes that transaction from every holder list and signals queues whose front might now proceed.

Integrate locking above buffer access: acquire before reading or modifying a record, keep it through WAL commit/abort, then release. Never hold the lock-manager mutex during disk I/O.

## 6. Run It

Use deterministic barriers to schedule two increments:

```text
T1 read 500
T2 read 500
T1 write 400
T2 write 400
```

Without locks, final balance is 400. With X locks around read-modify-write, one transaction reads the other's committed 400 and final balance becomes 300.

For dirty read, pause T1 after writing but before abort. T2's S request should wait; after abort restores the row and releases X, T2 reads the original value.

## 7. What Happened?

The lock table constrained histories rather than repairing their results afterward. Strict retention prevented cascading aborts: no transaction consumed a value that might still roll back. Throughput can fall because correct wait time replaces unsafe overlap.

Lock granularity is a tradeoff. Record locks allow unrelated rows to proceed but consume metadata and do not cover phantoms. A table lock is cheap and complete for scans but serializes far more work.

## 8. Measure It

Track lock requests, immediate grants, waits, upgrades, hold time, wait time, transactions per second, and aborts. Compare workloads updating one hot record, ten records, and disjoint records with 1, 2, 4, and 8 workers. The hot record's throughput should saturate while wait time grows.

Compare record and table locking for disjoint updates. Also time read-only workers: shared locks allow concurrency, but manager mutex contention can still appear. Counters should distinguish logical lock waiting from time in storage or WAL flush.

## 9. Break It

Create deadlock:

```text
T1 holds X(A), waits X(B)
T2 holds X(B), waits X(A)
```

Both can wait forever. Implement a timeout as a temporary victim policy, abort one transaction, and release its locks. Timeouts can abort a slow but non-deadlocked transaction; a waits-for graph would detect cycles more precisely.

Release write locks before commit and let a reader observe data whose commit later fails. Forget to lock index maintenance and concurrent uniqueness checks can both succeed. Scan a predicate while another transaction inserts a matching row to observe a phantom.

## 10. Improve It

Track a waits-for graph when blocking: edge `T1 -> T2` means T1 waits for T2. Cycle detection chooses a victim, which aborts through WAL and releases locks. Define a global resource ordering to prevent some deadlocks in internal operations.

Locks make readers wait behind writers even when an older committed version would be useful. Versioning can separate read visibility from overwrite coordination. The next chapter implements a tiny MVCC model and compares its benefits and costs with strict locking.

## 11. Exercises

1. Build compatibility-table tests for S, X, same-owner, and upgrade cases.
2. Reproduce lost update and dirty read with deterministic barriers.
3. Add timeout-based deadlock victim selection and safe abort.
4. Construct a waits-for graph and detect a two- and three-transaction cycle.
5. Demonstrate a phantom and explain what resource would need locking.

## 12. Checkpoint

TinyDB has a simple record/table lock manager, S/X compatibility, strict lock retention, waiting, upgrades, and a basic deadlock victim policy. Lost updates and dirty reads are now reproducibly prevented, while contention and phantoms expose the design's boundaries. MVCC next lets readers select stable committed versions without holding read locks.
