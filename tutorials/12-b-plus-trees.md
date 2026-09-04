# 12 — B+ Trees

## 1. Why This Exists

The hash index vanishes on restart and cannot answer `age BETWEEN 30 AND 40` without checking every key. A sorted tree can persist in pages, locate a boundary logarithmically, and then walk neighboring leaves. B+ trees are shaped for storage: high fanout keeps their height small.

## 2. Mental Model

```text
             [20 | 50]             internal root
            /    |     \
 [2,7,13] -> [20,31,44] -> [50,72] leaves
```

Internal keys separate child ranges. Actual `(key, record_id)` entries live in leaves. Leaves are linked left-to-right. Lookup descends from root; range scan descends once to the lower bound and follows leaf links.

## 3. Storage / Execution Model

Use an order of four children so splits happen in tiny experiments. Model nodes in memory first, then state how each node maps to a page. A leaf with four entries overflows, splits roughly in half, and copies the first key of the right leaf into its parent. An internal overflow promotes a separator upward. Splitting the root creates a new root and increases height.

Production trees address variable keys, concurrent splits, underflow merging, checksums, and crash recovery. TinyDB implements integer keys, insertion, lookup, and forward ranges only.

## 4. Representing It in OCaml

Variants make node roles explicit:

```ocaml
type leaf = {
  mutable entries : (int * record_id list) array;
  mutable next : page_id option;
}

type internal = {
  mutable keys : int array;
  mutable children : page_id array;
}

type node = Leaf of leaf | Internal of internal

type tree = {
  pool : Buffer_pool.t;
  mutable root : page_id;
  order : int;
}
```

The array lengths obey `children = keys + 1` for internal nodes. On disk, prefix every page with a node tag, key count, and optional next-leaf id; encode fixed-width keys and record ids after it.

## 5. Build It

Lookup chooses a child using the first separator strictly greater than the key:

```ocaml
let child_index keys key =
  let rec loop i =
    if i = Array.length keys || key < keys.(i) then i
    else loop (i + 1)
  in loop 0

let rec lookup tree pid key =
  match read_node tree.pool pid with
  | Leaf leaf -> find_leaf_entry leaf.entries key
  | Internal n -> lookup tree n.children.(child_index n.keys key) key
```

Insertion descends while recording parent page ids. Insert into a sorted leaf. If it overflows, allocate a right leaf, divide entries, repair `next`, and return `(separator, right_pid)` upward. Parents insert that pair beside the split child. If propagation exits above the old root, allocate `Internal { keys=[|separator|]; children=[|old; right|] }` and set the tree root.

Write a pure array-splitting helper and test it separately before adding page I/O.

## 6. Run It

Insert keys in ascending order `1..20`, printing the tree after every split. Repeat in descending and shuffled order. For every key, compare tree lookup with the chapter 11 hash index. Then scan range `[6, 14]`: locate 6, traverse leaf entries, follow `next`, and stop after 14.

Close and reopen after persisting the root page id in index metadata. Verify lookups without rebuilding from the heap.

## 7. What Happened?

Root growth kept every leaf at the same depth. Even with the intentionally tiny order, lookup visited far fewer nodes than rows. Real page-sized nodes hold many keys, so millions of records often require only a few I/Os.

Ascending inserts repeatedly touched the right edge and produced half-full left pages at split time. Random order distributed activity. Range scan used tree navigation only for its starting point; leaf links made the rest sequential.

Persistence also made update ordering dangerous. A parent must not point to an uninitialized right child, and a new root id must not become durable before the root page exists.

## 8. Measure It

Track node reads, buffer hits, comparisons within nodes, leaf splits, internal splits, root splits, height, and leaf occupancy. Compare lookup counts at 100, 1,000, and 10,000 keys. Measure point lookup cold and warm in the buffer pool.

For ranges of 1, 10, 100, and 1,000 adjacent keys, report internal pages and leaf pages visited. Compare against a heap scan and an in-memory hash index. Hash wins many point cases; it cannot naturally stop after an ordered interval.

## 9. Break It

Use the wrong equality rule in `child_index`: if separators copy the first key of the right child, equal keys must descend right. A boundary test catches the mismatch. Drop duplicate record ids by replacing a leaf entry rather than extending its list.

Crash after writing the new right leaf but before its parent. The page is orphaned. Crash after publishing the parent but before the child and lookup follows garbage. Delete most keys; without merge or redistribution, leaves become sparse. TinyDB accepts deletion underflow but must keep search correct.

## 10. Improve It

Centralize node validation: sorted keys, correct counts, child-range consistency, valid page ids, and acyclic leaf links. Store a root generation and use WAL later to recover structural changes. Prefix compression and binary search within nodes are useful refinements, not requirements here.

We now have alternative ways to access a relation, but queries are still assembled manually as functions. The next chapter represents intent as a logical plan, separating what the query means from how it will run.

## 11. Exercises

1. Implement and property-test sorted insertion and splitting on pure arrays.
2. Verify every leaf appears at one depth after random insertions.
3. Implement inclusive and exclusive range endpoints.
4. Serialize one leaf and internal node into 256-byte pages with bounds checks.
5. Explain the safe write order for a split and why WAL will improve it.

## 12. Checkpoint

TinyDB has a persistent ordered index model with internal routing, linked leaves, lookup, insertion, splitting, root growth, and ranges. It deliberately omits deletion rebalancing and crash-safe structural updates. With scans, two joins, and two index styles available, TinyDB next needs plans that distinguish query meaning from execution choice.
