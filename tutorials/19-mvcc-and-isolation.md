# 19 — MVCC and Isolation

## 1. Why This Exists

Strict locking protects correctness, but a long reader can block a writer, or a writer can block every reader of a hot row. The database already keeps before-images for recovery. What if committed historical values remained queryable? Multi-version concurrency control uses transaction visibility rather than mutual exclusion for reads.

## 2. Mental Model

```text
logical row 42
   |
   +-- value A, created T10, deleted T17
   +-- value B, created T17, deleted T24
   +-- value C, created T24, still live
```

A snapshot says which transactions were committed when a reader began. The reader chooses the newest version visible to that snapshot. A writer creates a new tentative version instead of overwriting the old bytes. Readers and writers can therefore overlap.

## 3. Storage / Execution Model

Use monotonically increasing transaction ids and a transaction table. At `BEGIN`, capture a snapshot with a high-water id and the set of transactions active at that instant. A version is visible when its creator committed before the snapshot, was not active in it, and its deleting transaction was not visible.

TinyDB implements snapshot isolation for reads and first-committer-wins on writes to the same logical row. This prevents dirty reads and lost updates. It is not full serializability: write skew remains possible when transactions update different rows after reading a shared predicate.

## 4. Representing It in OCaml

```ocaml
type snapshot = {
  xmax : txn_id;
  active : Txn_id_set.t;
}

type version = {
  value : Bytes.t;
  created_by : txn_id;
  mutable deleted_by : txn_id option;
  previous : version option;
}

type versioned_row = {
  logical_id : int64;
  mutable head : version option;
}

type txn_state = Running | Committed | Aborted
```

This in-memory chain explains visibility. A physical implementation would store version records in heap pages and use record ids as links; vacuum would eventually reclaim unreachable versions.

## 5. Build It

Visibility consults commit state and snapshot membership:

```ocaml
let tx_visible table snapshot tx =
  tx < snapshot.xmax
  && not (Txn_id_set.mem tx snapshot.active)
  && Hashtbl.find_opt table tx = Some Committed

let version_visible table snapshot v =
  tx_visible table snapshot v.created_by
  && match v.deleted_by with
     | None -> true
     | Some tx -> not (tx_visible table snapshot tx)

let rec visible_version table snapshot = function
  | None -> None
  | Some v when version_visible table snapshot v -> Some v
  | Some v -> visible_version table snapshot v.previous
```

A transaction sees its own created version as a special case and not its own deletion. Before commit, validate that no transaction committed a newer version of every row in the write set. Commit state must become durable through WAL before new versions count as visible.

## 6. Run It

Begin reader R, which sees balance 500. Begin writer W, create balance 400, and commit W. R reads again and should still see 500 from its snapshot. A new reader sees 400. Neither read waits on W after its commit, and R could also read the old version while W was tentative.

Start two writers from the same snapshot and have both update one row. Commit the first; the second must detect the newer committed head and abort. Then run a long reader while 100 updates commit and print version-chain length.

## 7. What Happened?

Visibility converted time into data. Readers no longer acquire shared record locks, so they do not block writers. The price is more storage, transaction-status lookups, longer chains, and cleanup rules. A long-running snapshot pins history: removing its old version would change what it sees.

First-committer-wins prevents two writers from overwriting the same logical row. Yet snapshot isolation permits anomalies across rows because each individual write may be conflict-free.

## 8. Measure It

Track versions created, versions traversed per read, active snapshots, write conflicts, aborts, reclaimed versions, and read/write throughput. Compare locking and MVCC with read-heavy, write-heavy, and one-hot-row workloads. Keep WAL policy identical so commit flushing does not confuse the comparison.

Create version chains of length 1, 10, 100, and 1,000. Read from a current and old snapshot and count traversal. Add a simple head cache only after measuring. Show that vacuum effectiveness depends on the oldest active snapshot.

## 9. Break It

Demonstrate write skew with two doctors, each on call. Transactions R1 and R2 both see two available doctors; each turns a different doctor off call. They do not write the same row, so both commit, violating “at least one on call.” Snapshot isolation is stronger than read committed but not serializable.

Remove a version needed by an old snapshot and show a non-repeatable answer. Treat an aborted creator as committed and expose dirty data. Reuse transaction ids after restart and visibility ordering becomes ambiguous. Let version chains grow without vacuum until reads and storage degrade.

## 10. Improve It

Vacuum may reclaim a version only when no active or future snapshot can need it and recovery retention permits removal. Maintain the oldest snapshot boundary. Serializable snapshot isolation would track predicate dependencies and abort dangerous structures; TinyDB documents rather than implements it.

Concurrency has now produced alternative physical designs, just as joins and access paths did. The final chapter puts plan alternatives under a small cost model and assembles the end-to-end database while preserving every declared limitation.

## 11. Exercises

1. Enumerate visibility for creator states running, committed, and aborted.
2. Implement own-write visibility and deletion behavior.
3. Reproduce first-committer-wins and write skew with deterministic schedules.
4. Define a safe vacuum horizon from active snapshots.
5. Compare guarantees of read committed, snapshot isolation, and serializable without implying MVCC selects one automatically.

## 12. Checkpoint

TinyDB has a compact MVCC model with snapshots, version chains, visibility, own writes, write-conflict aborts, and a vacuum horizon. It improves reader/writer concurrency while exposing storage growth and write skew. The final task is to choose among the engine's physical alternatives and integrate the complete query path.
