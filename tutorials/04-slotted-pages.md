# 04 — Slotted Pages

## 1. Why This Exists

Appending encoded rows inside a page works until one is deleted. Moving later bytes would invalidate every stored offset, while leaving holes wastes space. Variable-length updates are worse: a replacement may not fit where the old value lived. We need stable logical names for records even when their physical bytes move.

## 2. Mental Model

A slotted page separates names from placement:

```text
+----------------------+ low addresses
| header               |
| slot directory  ---> |
|                      |
|      free space      |
|                      |
| <--- record bytes    |
+----------------------+ high addresses
```

The slot directory grows upward; record contents grow downward. Each slot stores `(offset, length)`. Slot 3 remains slot 3 if compaction moves its bytes, because only the directory entry changes. The free interval lies between the directory end and record start.

## 3. Storage / Execution Model

For a 256-byte page, use an 8-byte header: two bytes each for magic, slot count, directory end, and record start. Each slot uses four bytes: a two-byte offset and two-byte length. Length zero denotes a deleted slot. An insertion needs `4 + record_length` contiguous free bytes unless it reuses a deleted slot, in which case it needs only the record bytes.

These widths are educational. They cap offsets and lengths at 65,535, which comfortably contains our page.

## 4. Representing It in OCaml

Keep the page byte-oriented, but centralize offsets:

```ocaml
type slot_id = int
type page = Bytes.t

let header_size = 8
let slot_size = 4
let slots p = get_u16_be p 2
let dir_end p = get_u16_be p 4
let data_start p = get_u16_be p 6
let slot_pos slot = header_size + (slot * slot_size)

let init p =
  Bytes.fill p 0 (Bytes.length p) '\000';
  set_u16_be p 0 0x5444;       (* "TD" *)
  set_u16_be p 2 0;
  set_u16_be p 4 header_size;
  set_u16_be p 6 (Bytes.length p)
```

In production, `page` would be abstract so callers cannot violate header invariants.

## 5. Build It

Insertion chooses the first deleted slot or appends a directory entry. It copies record bytes just below `data_start`:

```ocaml
let insert p record =
  let len = Bytes.length record in
  let reusable = find_deleted_slot p in
  let directory_cost = if Option.is_some reusable then 0 else slot_size in
  if data_start p - dir_end p < len + directory_cost then None
  else
    let start = data_start p - len in
    Bytes.blit record 0 p start len;
    let slot = Option.value reusable ~default:(slots p) in
    write_slot p slot ~offset:start ~length:len;
    set_u16_be p 6 start;
    if Option.is_none reusable then begin
      set_u16_be p 2 (slots p + 1);
      set_u16_be p 4 (dir_end p + slot_size)
    end;
    Some slot
```

`read` validates the slot id, treats length zero as absent, validates `offset + length`, and returns `Bytes.sub`. `delete` zeroes the length but preserves the slot. Do not zero payload bytes for correctness; they are unreachable garbage.

## 6. Run It

Initialize a page and insert encoded rows with names of different lengths. Save returned slot ids. Delete the middle record, insert another, and read every original id. If the deleted slot is reused, document that old record identifiers can now alias a new record—an important limitation.

Print after every operation:

```text
slot_count directory_end data_start contiguous_free live_bytes dead_bytes
```

Draw the byte ranges on paper. The drawing should agree with the header, not merely with successful reads.

## 7. What Happened?

Logical identifiers survived movement because callers followed the directory. Deletion was cheap: it changed metadata and did not rewrite neighboring records. But it did not reclaim payload space, so total free bytes and contiguous free bytes diverged. Reusing a slot avoids directory growth yet introduces stale-identifier aliasing unless slots carry generations or deleted identifiers are never exposed beyond a transaction.

Metadata updates are now multi-step. A crash after copying bytes but before publishing the slot leaves harmless garbage; a crash halfway through the slot entry may corrupt the page. Logging will eventually order these changes.

## 8. Measure It

Track insert attempts, successful inserts, bytes copied, reads, deletes, and compactions. Fill a page with alternating 8-byte and 40-byte records. Delete all 40-byte records and attempt a 60-byte insertion. Calculate live bytes and theoretically available bytes before observing whether contiguous allocation succeeds.

Time reading 100,000 random live slots. Directory lookup is constant time; `Bytes.sub` copies proportional to record length. Add a borrowed-slice experiment using `(page, offset, length)` and note the lifetime hazard if the page can be evicted or mutated.

## 9. Break It

Corrupt each header field independently. Set `directory_end` beyond `data_start`, make a slot point into the header, and make `offset + length` cross the page boundary. A safe `validate` function must reject every case before `read` copies bytes.

Create fragmentation: fill, delete alternating records, then request a record smaller than total garbage but larger than the contiguous interval. The naive allocator reports full. Also retain a record id, delete it, reuse its slot, and show that the old id reads new data.

## 10. Improve It

Compaction copies live payloads toward the page end and rewrites their slot offsets; slot ids stay fixed. Run it only when total reclaimable space could satisfy an insertion, because it copies many bytes. A generation number beside each slot can detect stale identifiers, at the cost of space and a wider record id.

TinyDB will initially accept slot reuse and document that record ids are not permanent application keys. The next chapter combines page id and slot id, scans multiple pages, and confronts the problem of choosing a page for insertion.

## 11. Exercises

1. Implement `validate : page -> (unit, string) result` with non-overlap checks.
2. Implement compaction and prove by experiment that every live slot reads identical bytes afterward.
3. Compare first-deleted-slot reuse with never-reuse behavior.
4. Add `free_space` and `garbage_space` functions; test their accounting after each mutation.
5. Decide what `delete` should return when the slot is already deleted.

## 12. Checkpoint

TinyDB can store variable-length records in one fixed page. Slot ids provide indirection, deletion is cheap, and compaction can reclaim holes without changing live ids. We have deliberately exposed fragmentation, stale-id reuse, and crash-sensitive metadata. Next, page and slot become a record id in a heap file.
