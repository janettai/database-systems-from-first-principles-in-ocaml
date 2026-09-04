# 01 — Why Databases Exist

## 1. Why This Exists

A database begins as an embarrassingly ordinary program. We need to associate keys with values, so an OCaml hash table looks sufficient:

```ocaml
type db = (string, string) Hashtbl.t

let create () = Hashtbl.create 128
let set db key value = Hashtbl.replace db key value
let get db key = Hashtbl.find_opt db key
let delete db key = Hashtbl.remove db key
```

This already implements `SET`, `GET`, and `DELETE`. Its expected lookup cost is excellent. The purpose of this course is to discover why this is not yet a database engine by making its hidden assumptions fail one by one.

## 2. Mental Model

The hash table is a mapping that lives inside one process:

```text
command -> hash -> bucket -> value
```

Its useful behavior depends on volatile memory, a single address space, and a workload dominated by exact-key lookups. A database must preserve useful behavior when those assumptions disappear. Think of each later subsystem as a contract inserted between the application and an unreliable or expensive resource.

## 3. Storage / Execution Model

For now, commands execute immediately:

```text
SET a 1    mutate memory
GET a      inspect memory
DELETE a   mutate memory
```

There is no durable representation, query plan, transaction boundary, or concurrency policy. A restart creates a new empty table. Listing all entries costs proportional to the number of bindings, and predicates require inspecting every value. Two threads can perform a read-modify-write sequence whose individual calls are safe while the sequence is not.

## 4. Representing It in OCaml

Give commands and results explicit types so the tiny command language cannot be confused with storage:

```ocaml
type command =
  | Set of string * string
  | Get of string
  | Delete of string

type result = Ok | Value of string option

let execute db = function
  | Set (k, v) -> Hashtbl.replace db k v; Ok
  | Get k -> Value (Hashtbl.find_opt db k)
  | Delete k -> Hashtbl.remove db k; Ok
```

This separation survives the course. Parsers will eventually produce commands, planners will transform queries, and storage modules will know nothing about SQL spelling.

## 5. Build It

Add a deliberately expensive scan and a counter:

```ocaml
type metrics = { mutable rows_examined : int }

let find_values metrics predicate db =
  Hashtbl.to_seq db
  |> Seq.filter_map (fun (k, v) ->
       metrics.rows_examined <- metrics.rows_examined + 1;
       if predicate k v then Some (k, v) else None)
  |> List.of_seq
```

Keep `metrics` outside the database. Observability should not change the answer, and resetting counters should not delete data. Seed the table with keys `user:1` through `user:100_000`; make every hundredth value equal to `active`.

## 6. Run It

In `utop`, execute a few commands, stop the session, and start another:

```ocaml
let db = create ();;
execute db (Set ("language", "OCaml"));;
execute db (Get "language");;
```

The second process cannot retrieve the value. Next time a scan for `active` and print both elapsed time and `rows_examined`:

```ocaml
let timed f =
  let before = Unix.gettimeofday () in
  let answer = f () in
  answer, Unix.gettimeofday () -. before
```

Run the scan at 1,000, 10,000, and 100,000 entries.

## 7. What Happened?

Exact lookup followed the hash table's structure, but a content predicate had no useful access path. The scan examined every binding even when only one row matched. Restart exposed a stronger failure: excellent performance is irrelevant if the representation vanishes. Memory also has a finite capacity, while files may be much larger. Finally, `GET`, computation, and `SET` form three operations; no hash-table method makes their combination indivisible.

These are database problems, not SQL problems. SQL is merely one interface for asking the engine to solve them.

## 8. Measure It

Record four observations:

```text
entries | exact lookups | rows examined by scan | scan milliseconds
```

Also inspect process memory with a tool available on your system. Do not chase precise benchmark numbers; look for shape. Exact lookup should remain roughly flat, scan work should grow linearly, and resident memory should grow with data. Add a `commands` counter and verify that a failed lookup still counts as work.

## 9. Break It

Implement an increment as `GET counter`, parse the integer, then `SET counter n+1`. Run two OCaml domains or threads that repeat it. If scheduling does not expose a lost update, insert `Domain.cpu_relax ()` between read and write. The final value can be smaller than the number of increments.

Then store enough large strings to pressure memory. Finally, write every mutation directly to a text file and terminate the process midway through a line. This tempting persistence patch creates partial records and ambiguous recovery.

## 10. Improve It

We now have the course's requirements. Values need a byte representation. Bytes need durable placement. Placement should use fixed-size pages so memory and disk exchange predictable units. Records need stable identifiers. Frequently used pages need caching. Queries need streaming operators and alternative algorithms. Updates need atomic transaction boundaries, logging, recovery, and concurrency rules.

The improvement in the next chapter is deliberately smaller: encode one structured row as bytes and decode it safely. Persistence cannot be correct until bytes have an unambiguous meaning.

## 11. Exercises

1. Add `SCAN prefix` without using a second hash table. Predict and then measure its work.
2. Make command parsing return `('a, string) result` instead of raising on malformed input.
3. Write down the exact instant at which a text-file `SET` becomes durable. Can the current program know?
4. Describe an interleaving in which two increments starting at 10 finish at 11.
5. Separate `Store` behind a module signature without exposing the `Hashtbl.t` representation.

## 12. Checkpoint

TinyDB has a typed command boundary, an in-memory key/value implementation, and visible work counters. It is fast for its favored workload but loses everything on restart, cannot address data larger than memory, scans for non-key predicates, and cannot group updates safely. We have encountered the problems before naming the abstractions that solve them. Next, a row becomes bytes.
