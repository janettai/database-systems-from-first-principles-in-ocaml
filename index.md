---
layout: home
title: Database Systems from First Principles in OCaml
---

Build a deliberately small relational database engine and discover each database abstraction only after a simpler design fails.

```text
bytes -> pages -> records -> heap files -> buffer pool
      -> operators -> indexes -> query plans
      -> transactions -> logging -> recovery
      -> concurrency -> optimization
```

## Start the course

1. [Why Databases Exist](tutorials/01-why-databases-exist.html)
2. [Records and Binary Encoding](tutorials/02-records-and-binary-encoding.html)
3. [Pages and Disk Layout](tutorials/03-pages-and-disk-layout.html)
4. [Slotted Pages](tutorials/04-slotted-pages.html)
5. [Heap Files and Record IDs](tutorials/05-heap-files-and-record-ids.html)
6. [Buffer Pools](tutorials/06-buffer-pools.html)
7. [Table Scans](tutorials/07-table-scans.html)
8. [Selection and Projection](tutorials/08-selection-and-projection.html)
9. [Nested-Loop Joins](tutorials/09-nested-loop-joins.html)
10. [Hash Joins](tutorials/10-hash-joins.html)
11. [Indexes](tutorials/11-indexes.html)
12. [B+ Trees](tutorials/12-b-plus-trees.html)
13. [Query Plans](tutorials/13-query-plans.html)
14. [Iterator Query Execution](tutorials/14-iterator-query-execution.html)
15. [Transactions and ACID](tutorials/15-transactions-and-acid.html)
16. [Write-Ahead Logging](tutorials/16-write-ahead-logging.html)
17. [Crash Recovery](tutorials/17-crash-recovery.html)
18. [Concurrency Control](tutorials/18-concurrency-control.html)
19. [MVCC and Isolation](tutorials/19-mvcc-and-isolation.html)
20. [Query Optimization and the Final Database](tutorials/20-query-optimization-and-final-database.html)

See the [repository guide](README.html) for scope, module boundaries, and study advice.
