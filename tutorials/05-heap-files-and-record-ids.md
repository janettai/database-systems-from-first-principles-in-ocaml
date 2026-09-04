# 05 — Heap Files and Record IDs

## 1. Why This Exists

One slotted page fills after a handful of rows. TinyDB needs a table that spans pages without pretending rows are ordered by a key. The simplest organization is a heap file: a collection of data pages in no particular key order. “Heap” here means unordered storage, not the priority-queue data structure.

## 2. Mental Model

A record has two coordinates:

```text
record id (page=4, slot=2)
             |       |
             |       +--> slot directory entry
             +----------> fixed file region
```

The page id gets us from file to page; the slot id gets us from page to bytes. A heap file owns allocation and insertion policy. A table adds a row codec and schema meaning. Keeping those layers distinct lets the same heap store other record formats later.

## 3. Storage / Execution Model

Begin with a naive insertion rule: scan data pages from the beginning, read each page, and try `Page.insert`; allocate a new page if none fits. Lookup reads the named page and slot. Deletion marks the slot empty. A full scan visits every page and every slot in physical order.

Reserve page 0 as a heap header containing magic, format version, and data-page count. Data pages begin at id 1. The header prevents “number of pages” from being inferred differently after a partial file extension, though its own crash consistency remains unsolved.

## 4. Representing It in OCaml

Use abstract ids at module boundaries:

```ocaml
type record_id = { page : Page.id; slot : Page.slot_id }

module type HEAP = sig
  type t
  val insert : t -> Bytes.t -> (record_id, string) result
  val get : t -> record_id -> (Bytes.t option, string) result
  val delete : t -> record_id -> (bool, string) result
  val iter : t -> (record_id * Bytes.t) Seq.t
end
```

The `bool` distinguishes deleting a live record from deleting an absent one. Errors distinguish I/O or corruption from absence. Do not use the record id as a user-visible primary key; compaction across pages may eventually change it.

## 5. Build It

A compact heap state can carry the disk manager and metrics:

```ocaml
type heap = {
  disk : Disk.t;
  mutable page_count : int;
  mutable pages_examined_for_insert : int;
}

let insert heap bytes =
  let rec search n =
    if n > heap.page_count then begin
      let pid, page = Disk.allocate_data_page heap.disk in
      match Page.insert page bytes with
      | None -> Error "record larger than an empty page"
      | Some slot -> Disk.write_page heap.disk pid page; Ok { page = pid; slot }
    end else begin
      heap.pages_examined_for_insert <- heap.pages_examined_for_insert + 1;
      let page = Disk.read_page heap.disk n in
      match Page.insert page bytes with
      | Some slot -> Disk.write_page heap.disk n page; Ok { page = n; slot }
      | None -> search (n + 1)
    end
  in
  search 1
```

Adapt the sketch to private `Page.id` constructors and structured disk errors. Update the page-count header only after the new data page is initialized; later WAL will make that ordering durable.

## 6. Run It

Create a `Table` wrapper whose `insert` encodes chapter 2's `row` and whose `get` decodes it. Insert rows until at least five pages exist. Store every returned record id in a list. Close and reopen the heap, then fetch the ids in a shuffled order.

Delete every third id. Scan the table and print `(page, slot, row.name)`. Physical scan order may resemble insertion order today, but write a test that treats order as unspecified. The SQL layer must add sorting explicitly if order matters.

## 7. What Happened?

TinyDB now addresses more data than one page. Record lookup costs one page read plus slot decoding, independent of earlier record lengths. A scan costs every data page, including pages containing only deleted slots. Insertion is surprisingly bad: as early pages fill, every new insert rereads them before reaching usable space. For `p` full pages and repeated inserts, the placement search can approach quadratic page examinations.

Stable ids also constrain maintenance. Moving a live record to another page would invalidate its id and every index entry that stores it.

## 8. Measure It

Record rows inserted, pages allocated, pages read, pages written, slots examined, and pages examined for placement. Insert fixed-size rows in batches of 100 and print cumulative placement reads. Compare the first and last batch.

Then scan after deleting 90 percent of rows. Report `pages_read / rows_produced`. This ratio makes wasted sparse-page I/O visible. Finally, compare 1,000 random `get` operations with one full scan; OS caching may reduce elapsed time but TinyDB's logical page-read count remains informative.

## 9. Break It

Try to insert a record larger than the maximum payload of an empty page. The allocator must reject it without appending an unusable page. Forge a record id pointing to page 0, beyond `page_count`, and to a nonexistent slot. Each case should return a bounded error.

Crash between appending a page and updating the header. On reopen, file length and header count disagree. Decide whether open should repair, ignore, or reject the orphan page. For now, reject and preserve evidence.

## 10. Improve It

Maintain a free-space directory: perhaps a list of page ids grouped into coarse buckets such as 0–31, 32–63, and 64+ free bytes. It may be stale; insertion must still verify the target page and fall back safely. This reduces placement reads without making metadata correctness essential.

Another problem is repeated physical reads. A `get` loop may ask the OS for the same hot page thousands of times. The next chapter introduces a bounded buffer pool that caches page frames, tracks dirty changes, and must decide what to evict.

## 11. Exercises

1. Implement heap open validation against magic, version, file length, and header page count.
2. Add a free-space bucket directory and measure placement reads again.
3. Calculate the maximum encoded row size that fits an empty slotted page.
4. Implement a physical iterator that skips deleted slots and reports corruption instead of ending early.
5. Explain why an index storing record ids makes cross-page compaction expensive.

## 12. Checkpoint

TinyDB has a heap file, stable `(page, slot)` record ids, typed table encoding, insertion, lookup, deletion, and sequential scanning. Its naive placement policy and repeated physical reads are measurable. A buffer pool now becomes necessary rather than ornamental.
