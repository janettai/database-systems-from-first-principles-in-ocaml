# 15 — Transactions and ACID

## 1. Why This Exists

Consider transferring 100 units:

```text
Alice -= 100
Bob   += 100
```

If TinyDB stops after the first page change, money disappears. If another query observes that halfway state, it sees a fact no completed operation intended. Grouping statements inside `BEGIN` and `COMMIT` names the desired unit, but those words matter only if the engine gives them operational meaning.

## 2. Mental Model

A transaction is a boundary around tentative effects:

```text
BEGIN T7
  read Alice
  write Alice
  read Bob
  write Bob
COMMIT T7
```

Before commit, failure should allow the whole unit to disappear. After a successful commit, restart should preserve it. Concurrent transactions should behave according to a declared isolation rule. Application constraints define valid states; the engine provides mechanisms that let the application preserve them.

## 3. Storage / Execution Model

ACID is useful when translated into mechanisms:

- Atomicity: all writes become committed or none do; undo/logging supplies this later.
- Consistency: a correct transaction moves between states satisfying declared invariants; the engine enforces types and chosen constraints, not business truth by itself.
- Isolation: concurrent histories appear according to an isolation contract; locks or versions supply this.
- Durability: once commit reports success, required log bytes have reached stable storage.

For this chapter, implement a single-active-transaction undo buffer in memory. It solves explicit abort but intentionally not process crash.

## 4. Representing It in OCaml

```ocaml
type txn_id = int64
type status = Active | Committed | Aborted

type undo = {
  rid : record_id;
  before : Bytes.t option;
}

type txn = {
  id : txn_id;
  mutable status : status;
  mutable undo : undo list;
  mutable writes : int;
}

type txn_error = Not_active | Conflict | Storage of string
```

`before = None` means the transaction inserted the record, so undo deletes it. A delete stores the old bytes so undo can restore them. Keep transaction ids monotonic within a process; persistent uniqueness comes with log metadata.

## 5. Build It

All mutation enters through transaction-aware functions:

```ocaml
let update tx table rid after =
  if tx.status <> Active then Error Not_active
  else
    match Table.get_bytes table rid with
    | Error e -> Error (Storage e)
    | Ok None -> Error (Storage "missing record")
    | Ok (Some before) ->
        tx.undo <- { rid; before = Some (Bytes.copy before) } :: tx.undo;
        tx.writes <- tx.writes + 1;
        Table.replace_bytes table rid after

let abort tx table =
  if tx.status <> Active then Error Not_active
  else begin
    List.iter (apply_undo table) tx.undo; (* newest first *)
    tx.status <- Aborted;
    Ok ()
  end
```

Commit currently clears undo and marks committed. This is deliberately not durable. The transaction manager must prevent bypassing its update API; otherwise some page changes cannot be undone.

## 6. Run It

Create Alice with 500 and Bob with 200. Begin a transaction, subtract from Alice, inject an error before Bob, and abort. Both balances should return to their original values. Then perform both updates and commit; the total stays 700.

Add checks after each phase:

```text
status undo_records dirty_pages total_balance
```

Repeat with two updates to Alice inside one transaction. Reverse-order undo should restore the value before the transaction, not the value after its first update.

## 7. What Happened?

In-memory undo gave explicit abort atomicity while the process remained alive. It did not give durability: buffer frames may still be dirty at commit. Flushing all pages would make committed bytes durable, but pages from an uncommitted transaction could also be evicted earlier. Preventing every such flush would couple transaction size to buffer capacity.

Isolation is also absent. Two transaction objects can read and overwrite the same row. “Transaction” is not a magic type; each ACID letter needs a concrete protocol.

## 8. Measure It

Track transactions begun, committed, aborted, undo bytes copied, rows updated, dirty pages, and commit latency. Run transactions with 1, 10, 100, and 1,000 updates. In-memory undo cost grows with before-images and retains memory until completion.

Compare a hypothetical force-at-commit policy—flush every modified page—with simply marking status. Count page writes for many transactions updating the same hot page. Forcing creates repeated data writes; logging can make commit require only sequential log I/O.

## 9. Break It

Terminate the process after the Alice page is flushed but before Bob changes. The undo list vanishes, leaving an inconsistent durable database. Evict an uncommitted dirty page and show that “do not flush at commit” does not mean “uncommitted data never reaches disk.”

Update an indexed key, fail after heap change, and omit index undo. Abort restores one structure but not the other. Make undo itself fail halfway; recovery needs an idempotent, durable source of truth rather than a fragile callback chain.

## 10. Improve It

Write change descriptions to an append-only log before allowing changed data pages to reach disk. At commit, force the small sequential log rather than every scattered page. Before-images permit undo; after-images permit redo. The buffer pool must know the newest log sequence number affecting each dirty page.

Concurrency control and durability are separable protocols that meet at transaction state. We tackle durability first with write-ahead logging, still using one transaction at a time.

## 11. Exercises

1. Add transactional insert and delete with correct reverse-order undo.
2. Optimize repeated updates to one record by storing only its first before-image; discuss redo consequences.
3. State exactly what this chapter's `commit` guarantees and does not guarantee.
4. Include index changes in the undo model.
5. Write three crash points for the transfer and the required state after restart.

## 12. Checkpoint

TinyDB has explicit transaction ids, states, mutation routing, commit, and in-process abort using before-images. The transfer experiment reveals why API boundaries alone cannot provide atomicity or durability across crashes. Next, WAL makes recovery information durable before data pages.
