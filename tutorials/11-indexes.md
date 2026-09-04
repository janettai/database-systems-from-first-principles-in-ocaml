# 11 — Indexes

## 1. Why This Exists

`SELECT * FROM users WHERE id = 7312` still scans every heap page. The predicate is selective, but the scan has no clue where the matching row lives. An index stores that missing route from a search key to one or more record ids.

## 2. Mental Model

```text
key 7312
    |
    v
index -> [(page=42, slot=3)]
                    |
                    v
                  heap row
```

The heap remains the source of record bytes. The index is redundant derived state that accelerates an access pattern. Redundancy creates maintenance obligations: every insert, delete, and key-changing update must keep heap and index synchronized.

## 3. Storage / Execution Model

Start with an in-memory hash index rebuilt by scanning the table at open. It supports expected constant-time point lookup but no sorted range scan and no durability. A unique index maps one key to at most one record id; a non-unique index maps a key to a list.

A primary index usually enforces or supports identity; a secondary index provides another access path. These are conceptual roles here. TinyDB's heap is not physically ordered by either.

## 4. Representing It in OCaml

```ocaml
type index_key = IInt of int | IText of string

type index = {
  name : string;
  column : int;
  unique : bool;
  entries : (index_key, record_id list) Hashtbl.t;
  metrics : index_metrics;
}

type index_error = Duplicate_key of index_key | Missing_record

val lookup : index -> index_key -> record_id list
val add : index -> index_key -> record_id -> (unit, index_error) result
val remove : index -> index_key -> record_id -> bool
```

Use a list even for unique indexes, then enforce list length at the API boundary. This avoids two separate representations while keeping the invariant explicit.

## 5. Build It

Rebuild by retaining physical ids during a heap scan:

```ocaml
let rebuild table index =
  Hashtbl.clear index.entries;
  Table.scan_with_rid table
  |> Seq.fold_left (fun acc item ->
       match acc, item with
       | Error _ as e, _ -> e
       | Ok (), Error e -> Error (`Scan e)
       | Ok (), Ok (rid, tuple) ->
           match key_at index.column tuple with
           | None -> Ok ()
           | Some key -> Result.map_error (fun e -> `Index e)
                           (add index key rid)) (Ok ())
```

Point lookup obtains candidate ids, fetches heap records through the buffer pool, and rechecks the predicate. Rechecking defends against a stale index and supports more complex predicates above the indexed equality.

For insertion, encode and insert the heap row, then add its key. If `add` rejects a duplicate, deleting the newly inserted heap row is a fragile rollback. This failure deliberately anticipates transactions.

## 6. Run It

Insert 100,000 users, build a unique id index, reset counters, and compare finding id 73,120 by scan and index. Report rows scanned, index lookups, heap page reads, and buffer hits. Repeat for a missing key.

Create a non-unique age index. Look up a common age and a rare age. The index always performs one key lookup, but fetching many scattered record ids may read more pages than a sequential scan.

Close and reopen. The in-memory index is gone; time its rebuild and treat that as startup cost.

## 7. What Happened?

For a selective lookup, the index replaced a table-wide scan with one directory operation and a few heap fetches. For a common value, random record-id fetches can defeat locality. An index is not automatically faster; selectivity and clustering matter.

Writes became more expensive and failure-prone. The index contains no new truth, yet incorrect maintenance can hide a real row or point to a deleted one. Rebuilding repairs derived state but is costly and cannot enforce uniqueness reliably during unavailable periods.

## 8. Measure It

Track index entries, distinct keys, lookups, returned record ids, heap fetches, stale ids, and maintenance operations. Compare point queries at selectivities 0, one row, 1 percent, 10 percent, and 100 percent. Include index-build time and approximate memory using entry and list counts.

Measure insertion throughput with no index, one index, and three indexes. Each extra read access path imposes write amplification. Randomize record insertion and compare fetched pages with a dataset where equal ages were inserted together.

## 9. Break It

Insert a heap row and skip index maintenance; the indexed query misses data. Delete a heap row but retain its index entry; lookup obtains a stale id. Attempt two rows with the same key under a unique index and crash between heap insertion and index rejection.

Modify a row's indexed column in place without moving its entry. Finally, keep record ids across a hypothetical cross-page compaction. These demonstrate that an index and heap must participate in the same atomic change.

## 10. Improve It

Transactions will eventually make heap and index maintenance atomic. Persistence needs an on-disk structure whose nodes fit pages. A sorted index also enables ranges and ordered scans that hashing cannot provide.

The canonical educational structure is a B+ tree: internal pages guide searches, leaf pages store sorted keys and record ids, and linked leaves support ranges. We implement its essential lookup, insertion, and splitting next, while omitting production refinements.

## 11. Exercises

1. Implement unique and non-unique `add` with deterministic duplicate behavior.
2. Recheck heap values after lookup and count stale entries.
3. Find the selectivity at which your age index loses to a scan.
4. Add a covering index containing `id` and `name`; identify which query avoids heap fetches.
5. Write the invariants needed to update a key safely, before implementing transactions.

## 12. Checkpoint

TinyDB can trade memory and write cost for selective point lookup using key-to-record-id indexes. Measurements reveal that selectivity, clustering, and maintenance dominate the choice. The deliberately volatile hash index exposes the need for a persistent, page-shaped, ordered structure: the B+ tree.
