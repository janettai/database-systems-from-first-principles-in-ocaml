# 06 — Buffer Pools

## 1. Why This Exists

Fetching the same record 10,000 times currently performs 10,000 disk-manager reads. The operating system may cache those bytes, but TinyDB cannot see hits, coordinate dirty writes, pin pages during use, or enforce a memory budget. The engine needs its own page cache: a buffer pool.

## 2. Mental Model

The buffer pool stands between every storage client and disk:

```text
operators
    |
    v
+---------+  hit   frame already present
| buffer  |  miss  read into a victim frame
+---------+
    |
    v
   disk
```

A frame is a memory slot holding one page. Its page id says what it currently represents. Dirty means memory differs from durable storage. Pin count says a caller is using it, so eviction is forbidden. Replacement chooses among unpinned frames.

## 3. Storage / Execution Model

Use a capacity of three frames to force replacement. On `fetch pid`, return an existing frame and increment its pin count, or select an unpinned victim. Flush a dirty victim before reading the requested page. On `unpin`, decrement the count and optionally mark dirty. `flush_all` writes every dirty page but does not evict it.

Start with FIFO: frames enter a queue on load; the oldest eligible frame loses. FIFO is simple but ignores later reuse. We will compare it with LRU-like recency.

## 4. Representing It in OCaml

```ocaml
type frame = {
  mutable page_id : Page.id option;
  data : Bytes.t;
  mutable dirty : bool;
  mutable pin_count : int;
  mutable touched_at : int;
}

type metrics = {
  mutable hits : int;
  mutable misses : int;
  mutable evictions : int;
  mutable flushes : int;
}

type t = {
  disk : Disk.t;
  frames : frame array;
  table : (Page.id, int) Hashtbl.t;
  mutable clock : int;
  metrics : metrics;
}
```

The page table maps a resident page id to its frame-array index. `touched_at` supports a simple LRU victim scan.

## 5. Build It

Make the safe use pattern scoped:

```ocaml
let with_page pool pid f =
  match fetch pool pid with
  | Error _ as e -> e
  | Ok frame ->
      Fun.protect
        ~finally:(fun () -> unpin pool pid)
        (fun () -> f frame.data)
```

For modifications, return whether the callback changed the page or provide `with_page_write` that always marks dirty. A miss first finds an empty frame, otherwise the unpinned frame with the smallest `touched_at`. Before reuse:

1. If dirty, write its old page.
2. Remove the old mapping.
3. Read the requested page into the existing buffer.
4. Reset metadata and insert the new mapping.

If all frames are pinned, return `Error All_frames_pinned`; never evict a live borrow.

## 6. Run It

With three frames and pages `A`, `B`, `C`, `D`, run reference strings:

```text
A A A A
A B C A B C
A B C D A B C D
A B C A D
```

After each access, print resident ids, pins, dirty flags, and cumulative hits/misses. Modify page A through `with_page_write`, then access enough pages to evict it. Reopen the disk and confirm the change was flushed.

Next, intentionally raise inside a `with_page` callback. `Fun.protect` should still unpin the frame.

## 7. What Happened?

Repeated access to A produced one miss followed by hits. A working set of four pages in three frames thrashed: each next request could evict the page needed soonest. This is not a correctness bug; bounded memory forces a policy choice. LRU helps when recent past predicts near future but can still scan more pages than capacity plus one.

Dirty eviction added an ordering constraint: the old page must be written before its bytes are overwritten. We still lack WAL, so a flush may expose part of a larger transaction.

## 8. Measure It

Report accesses, hits, misses, hit rate, evictions, dirty flushes, disk reads, and disk writes. Compare capacities 1, 3, 10, and 100 for a workload with 80 percent of accesses targeting 20 percent of pages. Repeat a sequential scan larger than each capacity.

Measure FIFO and LRU against the same precomputed reference string. Avoid generating randomness inside the timed region. A replacement policy wins only for a workload; do not conclude that one is universally best from one trace.

## 9. Break It

Fetch every frame without unpinning, then request another page. Correct behavior is a clear `All_frames_pinned`, not forced eviction. Unpin twice and ensure underflow is rejected. Mark a page dirty, forget to flush, then terminate abruptly; the durable file contains the old bytes.

Return `frame.data` from `with_page`, keep it after unpin, and trigger eviction. The borrowed value now refers to different page contents. This demonstrates why buffer ownership cannot be expressed safely by an unrestricted `Bytes.t`.

## 10. Improve It

Use opaque page handles whose operations check that the frame still carries the expected page. Make pin/unpin pairing hard to misuse through scoped callbacks. Background flushing could reduce eviction latency, but it introduces synchronization and WAL ordering, so defer it.

Route all heap operations through the pool and remove direct `Disk.read_page` calls from higher layers. Once pages are cached, a table scan can stream decoded rows while holding at most one page pin. That is our next execution abstraction.

## 11. Exercises

1. Implement FIFO, LRU, and clock replacement behind one policy type.
2. Assert the invariant that each resident page has exactly one page-table entry.
3. Add `flush_page` that refuses or permits pinned frames; justify the choice.
4. Create a trace where FIFO beats LRU and one where LRU beats FIFO.
5. Redesign `with_page` so callers cannot retain the mutable byte buffer.

## 12. Checkpoint

TinyDB owns a bounded cache with hits, misses, pins, dirty tracking, replacement, and flushing. We observed locality, thrashing, pin leaks, and unsafe borrowed bytes. Storage clients now access pages through the pool. Next, a scan turns heap pages into a lazy stream of rows.
