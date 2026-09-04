# 20 — Query Optimization and the Final Database

## 1. Why This Exists

TinyDB can scan or use an index, and can join with nested loops or hashing. Those choices are semantically equivalent only under their applicability rules, but their costs differ by orders of magnitude. The last subsystem turns one logical request into a cheap enough physical plan and then connects parser, catalog, execution, storage, and transactions into one small engine.

## 2. Mental Model

Filtering before a join can shrink work:

```text
bad:  Join users orders       better: Join filtered-users orders
          |                                |
       Filter age>70                    Filter age>70
```

Optimization is constrained search. Enumerate valid alternatives, estimate cardinality and cost, then choose the lowest estimated plan. Estimates will be wrong; counters let us compare estimates with reality and improve safely.

The integrated path is:

```text
query -> parser -> logical plan -> optimizer -> physical plan -> iterators
                                                           /           \
                                                      B+ index        heap
                                                           \           /
                                                            buffer pool
                                                                 |
                                                               disk
                                                     WAL + transactions
```

## 3. Storage / Execution Model

Collect tiny catalog statistics: table row count, heap page count, distinct count per indexed column, and optional minimum/maximum integer values. Estimate equality selectivity as `1 / distinct`, capped to sensible bounds. Estimate filtered rows as `rows * selectivity`.

Use costs in abstract page-work units:

```text
heap scan       = heap pages
index equality  = tree height + estimated matching heap pages
nested loop     = left cost + left rows * right cost
hash join       = left cost + right cost + left rows + right rows
```

These formulas omit CPU constants, caching, clustering, and memory spills. Their value is that decisions are explicit and testable, not that they predict milliseconds perfectly.

## 4. Representing It in OCaml

```ocaml
type estimate = { rows : float; cost : float }
type candidate = { plan : physical_plan; estimate : estimate }

type database = {
  catalog : Catalog.t;
  disk : Disk.t;
  pool : Buffer_pool.t;
  wal : Wal.t;
  txns : Txn_manager.t;
  locks : Lock_manager.t;
}

val optimize : Catalog.stats -> logical_plan -> candidate
val compile : Execution.context -> physical_plan -> operator
val execute : database -> txn -> statement -> (result_set, error) result
```

Keep estimates beside candidates rather than inside semantic logical nodes. The optimizer can be pure given a catalog snapshot.

## 5. Build It

Implement three decisions:

1. Replace `Filter(id = literal, Scan users)` with `Index_scan` when a compatible index exists and estimated fetch cost is below heap pages.
2. Push a filter below a join only when its referenced columns belong entirely to one child.
3. For equality joins, compare nested-loop candidates in both orientations with hash join building either side; reject hash candidates whose estimated build bytes exceed the memory budget.

The statement runner coordinates modules:

```ocaml
let run_select db tx syntax =
  let* bound = Binder.bind db.catalog syntax in
  let logical = Planner.logical bound in
  let chosen = Optimizer.optimize (Catalog.stats db.catalog) logical in
  let root = Executor.compile (context db tx) chosen.plan in
  Executor.consume root
```

`CREATE TABLE` updates catalog metadata transactionally. `INSERT` encodes a tuple, updates heap and indexes through WAL, and adjusts statistics approximately. `DELETE` plans candidate rows, rechecks its predicate, and creates a tombstone or MVCC deletion. Keep the parser intentionally narrow and report unsupported syntax.

## 6. Run It

Exercise the final language:

```sql
CREATE TABLE users (id INT, name TEXT, age INT);
INSERT INTO users VALUES (1, 'Ada', 36);
SELECT name FROM users WHERE age > 30;
DELETE FROM users WHERE id = 1;
```

Add `orders` and one equality join if you implemented join syntax. Print an explanation before execution:

```text
PProject users.name                 estimated rows=1
  Index_scan users_id key=1         estimated cost=3
```

Then print actual rows, page reads, buffer hits, and operator calls. Close, reopen, run recovery, and repeat the select.

## 7. What Happened?

The final database is composition, not one clever data structure. The parser names intent. Logical plans preserve relational meaning. The optimizer chooses among access and join implementations. Iterators stream rows. Indexes route selective access to record ids. Heap pages give durable placement. The buffer pool controls memory and I/O. Transactions, WAL, and recovery define atomic durable state. Locks or MVCC define which state concurrent queries may observe.

The cost model can still choose poorly. Stale distinct counts underestimate a popular value; scattered index hits cost more pages than predicted; a build input can exceed memory. An optimizer needs feedback and fallbacks.

## 8. Measure It

Build one reproducible benchmark driver with a fixed seed. Report:

```text
workload | rows | pages | plan | estimated rows | actual rows
         | disk reads | hits | rows scanned | comparisons | milliseconds
```

Include point lookup with and without index, selective and unselective filters, nested-loop versus hash equality join, cold versus warm buffer, transaction throughput with one and several workers, and recovery after a committed no-force update. Reset counters and cache state explicitly between comparable runs.

Treat estimation error as `max(actual,1) / max(estimated,1)` or its symmetric counterpart. Large factors identify statistics or independence assumptions worth improving.

## 9. Break It

Run deliberate failures as a graduation suite:

- corrupt a record length and require a bounded decode error;
- pin every frame and require `All_frames_pinned`;
- use a stale index entry and recheck the heap predicate;
- underestimate a hash build beyond budget and fall back or fail clearly;
- crash before and after WAL commit and recover the correct winner/loser state;
- deadlock two writers and abort one victim;
- demonstrate snapshot-isolation write skew as a documented non-goal.

Also make statistics stale by inserting a highly skewed batch without analyze. Compare chosen and best measured plans. Wrong estimates should cause slower execution, never wrong rows.

## 10. Improve It

Improve only where evidence points. Histograms help skew; sampled analyze lowers statistics cost; page-level free-space maps speed insertion; covering indexes avoid heap visits; batching reduces iterator overhead; group commit improves throughput. Each adds state and failure modes, so keep a regression experiment for the problem it solves.

Stop before TinyDB becomes a poor imitation of PostgreSQL. Full SQL, ARIES, production MVCC, distributed consensus, replication, sharding, columnar execution, and LSM trees deserve separate implementations and motivations. The educational engine succeeds when its mechanisms remain inspectable.

## 11. Exercises

1. Implement candidate enumeration and print rejected candidates with reasons.
2. Compare estimated and actual selectivity on uniform and skewed data.
3. Add `ANALYZE table` to refresh counts without changing query semantics.
4. Build the graduation failure suite with deterministic crash and schedule hooks.
5. Draw a trace for one committed insert from syntax through WAL flush and later indexed lookup.
6. Name the invariant owned by every TinyDB module and the counter that reveals its failure.

## 12. Checkpoint

TinyDB supports a minimal relational language through parsing, binding, logical planning, costed physical selection, pull-based operators, heap and indexed access, buffered pages, transactional logging, recovery, and a deliberately small concurrency model. Its measurements expose bad scans, quadratic joins, poor cache behavior, stale estimates, contention, and crash windows. The central lesson is concrete: a database engine is the coordinated composition of physical storage, memory management, indexes, query execution, transactions, recovery, concurrency control, and optimization.
