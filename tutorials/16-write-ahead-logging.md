# 16 — Write-Ahead Logging

## 1. Why This Exists

The buffer pool may evict a dirty page before its transaction commits, and forcing every data page at commit performs scattered writes. TinyDB needs freedom to flush pages independently without losing the information required to repair them. Write-ahead logging supplies one ordering rule: describe a change durably before the changed page reaches disk.

## 2. Mental Model

```text
transaction change
       |
       v
  append WAL record
       |
       v
 modify cached page
       |
       v
 flush page only after its WAL prefix is durable
```

Commit appends `COMMIT T7` and forces the log through that record. Data pages may be written later. Sequential log appends turn many random page updates into a small ordered durability stream.

## 3. Storage / Execution Model

Use records:

```text
BEGIN  T7
UPDATE T7 page=5 slot=2 before=A after=B
COMMIT T7
```

Every record has a monotonically increasing log sequence number, or LSN. Each dirty frame stores its latest `page_lsn`. The WAL manager tracks `flushed_lsn`. Before writing a frame, ensure `flushed_lsn >= page_lsn`. This is the WAL rule.

TinyDB logs whole record before/after images. Production systems often log smaller physiological changes and checksums; clarity wins here.

## 4. Representing It in OCaml

```ocaml
type lsn = int64

type log_record =
  | Begin of { lsn : lsn; tx : txn_id }
  | Update of {
      lsn : lsn; tx : txn_id; rid : record_id;
      before : Bytes.t option; after : Bytes.t option;
    }
  | Commit of { lsn : lsn; tx : txn_id }
  | Abort of { lsn : lsn; tx : txn_id }

type wal = {
  fd : Unix.file_descr;
  mutable next_lsn : lsn;
  mutable written_lsn : lsn;
  mutable flushed_lsn : lsn;
}
```

Encode each record as `[length][kind][lsn][tx][payload][checksum]`. Length permits iteration; checksum distinguishes a valid prefix from a torn tail.

## 5. Build It

The update sequence is non-negotiable:

```ocaml
let logged_update db tx rid after =
  let* before = Table.get_bytes db.table rid in
  let* lsn = Wal.append db.wal (fun lsn ->
    Update { lsn; tx = tx.id; rid; before; after = Some after }) in
  let* () = Table.replace_bytes ~page_lsn:lsn db.table rid after in
  Ok ()
```

When the pool flushes a dirty frame:

```ocaml
let flush_frame pool frame =
  let* () = Wal.flush_through pool.wal frame.page_lsn in
  let* () = Disk.write_page pool.disk frame.page_id frame.data in
  frame.dirty <- false;
  Ok ()
```

Commit appends a commit record, calls `fsync` or the platform's durable flush on the WAL, then reports success. Do not mark commit successful before that call returns.

## 6. Run It

Perform a transfer while tracing:

```text
event | written_lsn | flushed_lsn | dirty page_lsn | disk writes
```

Force a dirty-page eviction before commit. The buffer pool should first flush WAL through that page's LSN, then write the page. Commit afterward should flush the commit record, not necessarily the page.

Reopen immediately after commit without explicitly flushing data pages. The file may contain old data, which is allowed because the durable WAL can redo committed changes next chapter.

## 7. What Happened?

WAL decoupled transaction durability from data-page timing. Steal is now possible: the pool may write uncommitted dirty pages because before-images survive for undo. No-force is possible: commit need not write every changed page because after-images survive for redo.

The ordering rule is more important than record syntax. Logging after page modification leaves a window where eviction can write an undescribed change. Reporting commit before the commit log is stable tells the application a promise the engine cannot keep.

## 8. Measure It

Track log records, log bytes, log appends, log flushes, data-page writes, transactions, and commit latency. Compare forcing once per update with group commit, where several transaction commit records share one flush. Even a simple single-process queue can demonstrate fewer flushes per transaction.

Run 1,000 tiny transactions updating one hot page and 1,000 pages. Contrast WAL bytes and page writes under force-at-commit and no-force. Use counters first; filesystem and hardware caches make microbenchmarks easy to misread.

## 9. Break It

Reverse the rule: write a changed page, then append its update record. Terminate between them; recovery has no before-image. A test hook immediately after each I/O makes this deterministic.

Write a partial length or payload at the WAL tail. On restart, scanning must accept the valid prefix and discard or truncate the incomplete tail. Corrupt a checksum in the middle; unlike a torn tail, middle corruption should stop recovery loudly because later record boundaries cannot be trusted.

Reuse an LSN after restart. Page-LSN comparisons can then skip required redo.

## 10. Improve It

Persist the next-LSN basis or derive it from the last valid record. Make log append and scan codecs total, bounded, and checksummed. Store `page_lsn` in each data-page header, not only the volatile frame, so recovery can decide whether an update is already applied.

WAL makes repair information durable but does not itself perform repair. Next, startup scans the log, classifies committed and incomplete transactions, redoes committed effects, and undoes losers. We will use a simplified two-pass algorithm rather than full ARIES.

## 11. Exercises

1. Define and encode bounded WAL records with a checksum.
2. Add `page_lsn` to the slotted-page header and adjust available space.
3. Assert `flushed_lsn >= page_lsn` before every data-page write.
4. Measure group sizes 1, 4, 16, and 64 commits per flush.
5. Explain steal/no-steal and force/no-force using TinyDB's current policy.

## 12. Checkpoint

TinyDB appends versioned, checksummed begin/update/commit records, gives changes LSNs, forces commit records, and enforces WAL-before-data through the buffer pool. It supports steal and no-force in principle. Durable evidence now exists; crash recovery turns that evidence back into a consistent heap.
