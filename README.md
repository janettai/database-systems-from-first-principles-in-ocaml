# Database Systems from First Principles in OCaml

This is a 20-part tutorial about building a deliberately small relational database engine. It begins with bytes and ends with a cost-based choice between scans, indexes, and join algorithms. The point is not SQL coverage. The point is to make every familiar database abstraction appear only after a simpler design has failed in an observable way.

The tutorials evolve one implementation, called **TinyDB**. Code fragments are intentionally small enough to type into `utop` or gather into modules of your own. Each chapter ends with a checkpoint that states the new invariant TinyDB has earned and the limitation that motivates the next chapter.

The path is:

```text
bytes -> pages -> records -> heap files -> buffer pool
      -> operators -> indexes -> query plans
      -> transactions -> logging -> recovery
      -> concurrency -> optimization
```

Start at [01 — Why Databases Exist](tutorials/01-why-databases-exist.md) and proceed in order. Keep a notebook of the counters requested by each experiment: disk reads, cache hits, rows scanned, join comparisons, log flushes, and transaction throughput. The numbers are part of the implementation, not decoration.

## Curriculum

1. [Why Databases Exist](tutorials/01-why-databases-exist.md) — break an in-memory key/value store.
2. [Records and Binary Encoding](tutorials/02-records-and-binary-encoding.md) — give structured rows a checked byte format.
3. [Pages and Disk Layout](tutorials/03-pages-and-disk-layout.md) — introduce fixed-size I/O and stable page addresses.
4. [Slotted Pages](tutorials/04-slotted-pages.md) — place variable-length records behind stable slots.
5. [Heap Files and Record IDs](tutorials/05-heap-files-and-record-ids.md) — assemble pages into an unordered table store.
6. [Buffer Pools](tutorials/06-buffer-pools.md) — cache, pin, evict, dirty, and flush pages.
7. [Table Scans](tutorials/07-table-scans.md) — stream physical records as typed rows.
8. [Selection and Projection](tutorials/08-selection-and-projection.md) — compose the first relational operators.
9. [Nested-Loop Joins](tutorials/09-nested-loop-joins.md) — establish the correct quadratic baseline.
10. [Hash Joins](tutorials/10-hash-joins.md) — trade build memory for expected linear equality joins.
11. [Indexes](tutorials/11-indexes.md) — replace whole-table point lookups with record-ID routes.
12. [B+ Trees](tutorials/12-b-plus-trees.md) — make an ordered, persistent, page-shaped index.
13. [Query Plans](tutorials/13-query-plans.md) — separate relational intent from algorithm choice.
14. [Iterator Query Execution](tutorials/14-iterator-query-execution.md) — run plans as pull-based operator trees.
15. [Transactions and ACID](tutorials/15-transactions-and-acid.md) — define the unit that succeeds or fails.
16. [Write-Ahead Logging](tutorials/16-write-ahead-logging.md) — make recovery evidence durable before data.
17. [Crash Recovery](tutorials/17-crash-recovery.md) — redo winners and undo losers after restart.
18. [Concurrency Control](tutorials/18-concurrency-control.md) — prevent unsafe interleavings with locks.
19. [MVCC and Isolation](tutorials/19-mvcc-and-isolation.md) — serve snapshots from version chains.
20. [Query Optimization and the Final Database](tutorials/20-query-optimization-and-final-database.md) — cost alternatives and integrate TinyDB.

TinyDB's final language is deliberately narrow:

```sql
CREATE TABLE users (id INT, name TEXT, age INT);
INSERT INTO users VALUES (1, 'Ada', 36);
SELECT name FROM users WHERE age > 30;
DELETE FROM users WHERE id = 1;
```

One equality join is enough to compare physical strategies. There is no PostgreSQL compatibility, distributed protocol, production MVCC, full ARIES, replication, sharding, or complete SQL grammar. Those systems are easier to appreciate once the small mechanisms here are familiar.

Suggested module boundaries are `Codec`, `Page`, `Heap`, `Buffer_pool`, `Operator`, `Index`, `Plan`, `Txn`, `Wal`, and `Recovery`. Mutation is used where the machine being modeled mutates. Interfaces should expose invariants—valid page sizes, stable record identifiers, pin discipline, and WAL ordering—rather than merely repeat implementation types.

All examples assume OCaml 5 and the Unix library. Timing snippets may use `Unix.gettimeofday`; persistence snippets use binary file descriptors. Run destructive crash experiments only against a disposable database file.
