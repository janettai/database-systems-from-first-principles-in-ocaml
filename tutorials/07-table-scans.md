# 07 — Table Scans

## 1. Why This Exists

The heap iterator can expose physical records, but a query needs typed rows and should not load the whole table into a list. `SELECT * FROM users` is our first relational operation. It will also establish the execution rule used throughout TinyDB: produce one row at a time and keep memory bounded.

## 2. Mental Model

A scan is a pipeline from physical coordinates to values:

```text
heap pages -> live slots -> record bytes -> decoded rows
```

It owns a cursor `(next_page, next_slot)`. Asking for the next row advances until it finds a live slot, decodes it, and yields. End of page moves to the next page; end of heap ends the sequence. The consumer controls how far execution proceeds.

## 3. Storage / Execution Model

Only one page should be pinned while inspecting its slots, and no page should remain pinned while the consumer processes a yielded row. That implies copying record bytes or decoding the row before unpinning. A scan that returns borrowed page slices would either pin many pages indefinitely or expose invalid memory after eviction.

Physical order remains unspecified. Deletions create skipped slots; corrupt records must become errors rather than silently terminating the scan.

## 4. Representing It in OCaml

OCaml's standard `Seq.t` is:

```ocaml
type 'a node = Nil | Cons of 'a * 'a t
and 'a t = unit -> 'a node
```

Use results in the element type:

```ocaml
type scan_error =
  | Storage of string
  | Decode of record_id * decode_error

val scan : table -> (row, scan_error) result Seq.t
```

A scan error is data delivered at the exact physical position. Decide whether the consumer stops or continues; the default query runner should stop to avoid returning an apparently complete result.

## 5. Build It

Represent mutable cursor state inside the closure for clarity:

```ocaml
let scan table =
  let page = ref 1 in
  let slot = ref 0 in
  let rec next () =
    if !page > Heap.page_count table.heap then Seq.Nil
    else
      match Heap.read_slot table.heap ~page:!page ~slot:!slot with
      | Error e -> Seq.Cons (Error (Storage e), fun () -> Seq.Nil)
      | Ok `End_of_page -> incr page; slot := 0; next ()
      | Ok `Deleted -> incr slot; next ()
      | Ok (`Record bytes) ->
          let rid = { page = Page.id_exn !page; slot = !slot } in
          incr slot;
          Seq.Cons (Result.map_error (fun e -> Decode (rid, e))
                      (Codec.decode bytes), next)
  in
  next
```

In the real heap API, inspect a whole page under one `with_page` callback rather than fetching it once per slot. Store a copied list of decoded results for that page or implement an explicit iterator whose `next` manages the pin precisely.

## 6. Run It

Insert 1,000 rows, delete every seventh, reset metrics, and consume the scan:

```ocaml
Table.scan users
|> Seq.fold_left (fun count -> function
     | Ok _ -> count + 1
     | Error e -> raise (Scan_error e)) 0
```

Confirm the count. Then consume only the first ten rows with a bounded sequence helper. Compare disk reads and decoded rows with a full scan. Laziness should prevent reading the remainder.

Run with a one-frame buffer pool; if pin discipline is correct, the scan still succeeds.

## 7. What Happened?

Execution became demand-driven. Producing ten rows did work only until the tenth live slot. A full scan read each page once when the implementation processed slots together; a per-slot heap API would have produced many pool lookups, perhaps hits, but unnecessary overhead.

The sequence owns mutable progress and is single-use in spirit. Calling the same closure from multiple consumers interleaves one cursor. Persistence is a property of the table; reusability is not automatically a property of a stateful scan value.

## 8. Measure It

Track pages visited, slots visited, records decoded, rows produced, decode errors, buffer hits, and disk reads. Compare:

```text
take 1 | take 10 | take 100 | consume all
```

Repeat with dense pages and pages where 90 percent of slots are deleted. Sparse storage should increase slots and pages per produced row. Also compare materializing `List.of_seq` with folding the sequence; measure peak live memory on a table with large names.

## 9. Break It

Create a page with one malformed record between valid rows. A runner that ignores errors can return an incomplete or misleading result. Confirm TinyDB reports the offending record id.

Write a scan that pins a page, yields a borrowed `Bytes.t`, and unpins only when asked for the next item. Stop after one row and abandon the iterator. The frame stays pinned forever. This is a resource-leak pattern hidden by laziness.

Mutate the table during a scan. Depending on cursor position, the new row may appear, disappear, or reuse a deleted slot already passed. We have no snapshot semantics yet.

## 10. Improve It

Use an explicit closeable cursor for resource-bearing operators, or guarantee that each `next` releases all resources before returning. TinyDB chooses the latter and returns owned rows. Include a table generation if structural changes must invalidate scans, though transactions later provide better semantics.

A scan only reproduces the table. Queries become useful when operators transform its stream. Next we implement selection and projection as composable row functions and make the cost of predicate placement observable.

## 11. Exercises

1. Implement `Seq.take` without evaluating the `(n+1)`th element.
2. Refactor scan so one buffer-pool fetch serves all slots on a page.
3. Choose and document stop-on-error versus continue-on-error behavior.
4. Add a cancellable scan that checks a flag between pages.
5. Explain why result ordering cannot rely on record ids after future page reorganization.

## 12. Checkpoint

TinyDB can stream decoded table rows with bounded memory and explicit errors. Counters show early termination, sparse-page cost, and the difference between buffer hits and disk reads. The scan is the leaf of our future operator tree; selection and projection will wrap it.
