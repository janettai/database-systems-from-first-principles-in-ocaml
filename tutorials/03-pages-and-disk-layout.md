# 03 — Pages and Disk Layout

## 1. Why This Exists

If records have different sizes, “seek to record 73” has no fixed meaning. Scanning every earlier length prefix would make random access increasingly expensive, and replacing a name with a longer name would shift everything after it. TinyDB needs an address space that stays stable while individual records change.

## 2. Mental Model

Divide one database file into equal-size pages:

```text
database file

+--------+ byte 0
| page 0 |
+--------+ byte 256
| page 1 |
+--------+ byte 512
| page 2 |
+--------+
```

We use 256-byte pages so experiments fill them quickly. Real engines commonly use larger pages. A page is both an on-disk region and an in-memory byte buffer. Its identity is an integer, not a file offset chosen throughout the code.

## 3. Storage / Execution Model

The mapping is one multiplication:

```text
offset = page_id * page_size
```

Reading page 7 means seeking to byte 1,792 and requesting exactly 256 bytes. TinyDB treats a short read as corruption, except when allocating at the current end of file. Page 0 will eventually hold metadata; for now every page is raw. Fixed pages align caching, eviction, logging, and record identifiers around the same unit.

## 4. Representing It in OCaml

Hide constructors that could create negative identifiers or wrong-sized pages:

```ocaml
module Page : sig
  type id = private int
  type t
  val size : int
  val id : int -> (id, string) result
  val empty : unit -> t
  val bytes : t -> Bytes.t
end = struct
  type id = int
  type t = Bytes.t
  let size = 256
  let id n = if n < 0 then Error "negative page id" else Ok n
  let empty () = Bytes.make size '\000'
  let bytes p = p
end
```

Returning the mutable bytes weakens abstraction, but makes the mechanism visible. A later interface can expose checked reads and writes instead.

## 5. Build It

Create a disk manager with exact I/O and counters:

```ocaml
type disk = {
  fd : Unix.file_descr;
  mutable reads : int;
  mutable writes : int;
}

let rec really_read fd b off remaining =
  if remaining = 0 then ()
  else
    let n = Unix.read fd b off remaining in
    if n = 0 then failwith "short page"
    else really_read fd b (off + n) (remaining - n)

let read_page d (pid : Page.id) =
  let b = Bytes.create Page.size in
  ignore (Unix.LargeFile.lseek d.fd
    Int64.(mul (of_int (pid :> int)) (of_int Page.size)) Unix.SEEK_SET);
  really_read d.fd b 0 Page.size;
  d.reads <- d.reads + 1;
  b
```

Implement `really_write` similarly because `Unix.write` may write fewer bytes than requested. `allocate_page` seeks to end, verifies page alignment, appends an all-zero page, and returns its id.

## 6. Run It

Open a disposable file using `O_RDWR`, `O_CREAT`, and binary mode where relevant. Allocate three pages. Put a recognizable byte into each—`A`, `B`, and `C` at offset zero—write them, close the file, reopen it, and read in the order 2, 0, 1.

Use `Unix.LargeFile.fstat` to confirm that size is exactly `3 * Page.size`. Print disk counters. Six logical operations should produce the reads and writes you expect; if helper functions secretly read to allocate, the count will reveal it.

## 7. What Happened?

Page identity remained stable across close and reopen. Reading page 2 did not require parsing pages 0 or 1. We traded some internal fragmentation for predictable addressing and bounded I/O. A five-byte record occupies part of a 256-byte transfer, but several records can share that page later.

The exact-read loop matters even for local files: system calls report work completed, not a promise that the requested count was satisfied. The file size is now an invariant. A length not divisible by page size indicates a torn allocation or unrelated writer.

## 8. Measure It

Write 1,000 page reads with three patterns: repeatedly page 0, sequential pages modulo the file size, and pseudorandom pages. Count system calls and elapsed time. Operating-system caching may make repeated reads fast, but TinyDB still issues every call; that distinction will motivate its own buffer pool.

Also compare reading one encoded row via a full page against reading only its byte length. Small direct reads may look attractive, but they complicate caching and multiply system calls. Record requested bytes, completed bytes, reads, writes, and seeks.

## 9. Break It

Append 17 arbitrary bytes to the file and reopen it. Allocation must refuse the misaligned size rather than silently reinterpret those bytes as part of a page. Truncate the last page and verify that `read_page` fails cleanly. Request a negative page through the public constructor and confirm it is rejected.

Compute an offset using unchecked native integer multiplication for an enormous page id. Overflow can turn a valid-looking id into the wrong location. Production code must use checked `int64` arithmetic and compare the id with file page count.

## 10. Improve It

Return structured errors rather than `failwith`, loop on `EINTR`, validate bounds before seeking, and use `Fun.protect` to close descriptors. Reserve a small header on page 0 for magic bytes, format version, and page count. TinyDB can adopt those refinements incrementally.

Pages solve address stability, not variable-record placement inside a page. Packing records contiguously means deletion creates holes and length changes move bytes. Next we add a slot directory so a stable small integer can name each record within a page.

## 11. Exercises

1. Implement `write_page` and `allocate_page` with complete-write loops.
2. Add `page_count` that rejects a misaligned file.
3. Put magic bytes and a format version in page 0, then reject another file type.
4. Replace multiplication with a checked `int64` helper.
5. Explain why an OS page-cache hit is still counted as a TinyDB disk-manager read.

## 12. Checkpoint

TinyDB has a fixed 256-byte I/O unit, stable page identifiers, exact I/O loops, allocation, and visible read/write counters. We can locate a page in constant time, but not yet a variable-length record inside it. Slotted pages provide that second coordinate.
