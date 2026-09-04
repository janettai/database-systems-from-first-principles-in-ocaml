# 17 — Crash Recovery

## 1. Why This Exists

After a crash, TinyDB may contain a mixture: old pages for committed transactions, new pages for uncommitted transactions, and a WAL whose tail ends mid-record. WAL preserved enough evidence, but the database is not usable until startup interprets it. Recovery is the procedure that reestablishes transaction guarantees.

## 2. Mental Model

```text
read valid log prefix
        |
        v
classify committed and incomplete transactions
        |
        +--> redo committed after-images
        |
        +--> undo incomplete before-images, newest first
        v
consistent pages
```

Redo is “ensure this committed effect is present,” not “blindly repeat every write.” Undo is “remove effects of transactions that never committed.” Both should be idempotent so another crash during recovery can safely restart recovery.

## 3. Storage / Execution Model

Use a simplified two-pass algorithm:

1. Scan the valid WAL prefix and build each transaction's status plus ordered update list.
2. Redo updates of committed transactions in ascending LSN order.
3. Undo updates of uncommitted transactions in descending LSN order.
4. Flush repaired pages and append abort markers for losers.

A page's stored `page_lsn` lets redo skip an update already reflected on disk. Our whole-record images make replay straightforward. This is not ARIES: there is no dirty-page table, compensation log record design, fuzzy checkpoint, or fine-grained physiological redo.

## 4. Representing It in OCaml

```ocaml
type tx_info = {
  mutable committed : bool;
  mutable updates : log_record list;
}

type recovery_stats = {
  mutable records_scanned : int;
  mutable redone : int;
  mutable redo_skipped : int;
  mutable undone : int;
  mutable torn_tail_bytes : int;
}

type recovery_error =
  | Corrupt_log of lsn * string
  | Corrupt_page of page_id * string
  | Missing_begin of txn_id
```

Keep recovery separate from normal transaction APIs: replay must set a page's bytes and LSN without emitting an identical user update record recursively.

## 5. Build It

Analysis records transaction state:

```ocaml
let analyze records =
  let txs = Hashtbl.create 32 in
  Seq.iter (function
    | Begin { tx; _ } -> Hashtbl.replace txs tx { committed = false; updates = [] }
    | Update ({ tx; _ } as r) ->
        let info = Hashtbl.find txs tx in
        info.updates <- Update r :: info.updates
    | Commit { tx; _ } -> (Hashtbl.find txs tx).committed <- true
    | Abort _ -> ()) records;
  txs
```

Store updates newest-first. For redo, collect committed updates and sort by LSN ascending. Apply an after-image only when the page's LSN is less than the record LSN. For undo, walk each loser's stored list as-is and apply before-images. Assign recovery changes appropriate LSNs or force all repaired pages before declaring recovery complete; document the simplified choice.

## 6. Run It

Add deterministic crash points after:

1. appending an update but before page mutation;
2. flushing an uncommitted changed page;
3. flushing commit but before data pages;
4. writing half a WAL record.

Run the transfer in a child process or terminate at the hook, reopen, recover, and check both balances and their sum. Repeat recovery immediately. The second run should change no logical data.

Print transaction classification and every `REDO`, `SKIP`, and `UNDO` decision with LSN and record id.

## 7. What Happened?

A committed transaction whose pages were old was repaired by redo. An uncommitted transaction whose page escaped the buffer was repaired by undo. A logged but unapplied update was either redone if committed or restored to its already-old before-image if uncommitted. The torn tail was ignored only after its preceding checksum-valid records.

Idempotence depended on page LSNs for redo and on whole before-images for undo. If recovery itself crashes midway, blindly applying an old before-image again is harmless for this simplified record replacement, but more complex operations need compensation logging.

## 8. Measure It

Track startup log bytes read, valid records, transactions classified, redo applied/skipped, undo applied, pages read/written, and recovery time. Create logs with 10, 1,000, and 100,000 historical updates but only a few dirty pages. Full scanning grows with history, even when little repair is needed.

Add a quiescent checkpoint record containing the active transaction ids after flushing all dirty pages. Recovery may start after the checkpoint while retaining earlier records needed by active transactions. Measure the reduced scan, and state why deleting older WAL prematurely would destroy undo information.

## 9. Break It

Apply transaction updates in log order during undo instead of reverse order: two changes to one row restore an intermediate value. Skip page-LSN checks during redo and use a non-idempotent physical operation such as “increment field”; a second recovery increments again.

Treat a checksum failure in the log middle as a torn tail and continue searching for plausible headers. Random bytes can be mistaken for commits. Fail closed on interior corruption. Finally, delete WAL immediately after commit while its data page remains dirty; a crash loses durable work.

## 10. Improve It

Write recovery tests as enumerations of crash points, not probabilistic process killing. Add checksums to pages and atomic replacement rules for the WAL tail. Checkpoints bound recovery work but require careful log-retention calculations.

TinyDB now repairs a single serial transaction history. Real clients overlap. Without a concurrency protocol, even perfect recovery can preserve a committed lost update. Next, a simple lock manager controls interleavings and makes serializability tangible.

## 11. Exercises

1. Implement log-prefix scanning that distinguishes torn tail from middle corruption.
2. Generate crash points around every WAL, page, and flush action in a two-update transaction.
3. Prove experimentally that a second recovery performs zero redo writes when page LSNs are current.
4. Add a quiescent checkpoint and document safe log truncation.
5. Explain one feature missing from this algorithm that full ARIES supplies.

## 12. Checkpoint

TinyDB can classify winners and losers after a crash, redo committed after-images, undo incomplete before-images in reverse order, ignore only a torn WAL tail, and skip already-applied redo using page LSNs. Recovery is intentionally smaller than ARIES. With durability established, concurrent histories become the next source of inconsistency.
