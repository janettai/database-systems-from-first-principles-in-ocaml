# 02 — Records and Binary Encoding

## 1. Why This Exists

Files store bytes, not OCaml records. TinyDB now wants structured rows:

```ocaml
type row = { id : int; name : string; age : int }
```

Writing `Marshal.to_bytes` would persist an OCaml-specific object graph, but it would hide field layout, make corruption hard to diagnose, and couple the database format to runtime details. We will define a tiny format ourselves so every byte has an explained purpose.

## 2. Mental Model

Encoding is a contract between a writer today and a reader later:

```text
row -> [id][age][name length][name bytes] -> row
```

Fixed-width integers make offsets predictable. The string is variable-width, so its length must travel with it. Decoding is not the inverse by magic; it is validation of untrusted bytes followed by construction of a value. A torn file, old format, or incorrect offset must become an error rather than an arbitrary row.

## 3. Storage / Execution Model

Use a deliberately simple version-1 record:

```text
byte 0       format version, unsigned 8-bit
bytes 1..4   id, signed 32-bit big-endian
byte 5       age, unsigned 8-bit
bytes 6..7   UTF-8 name length, unsigned 16-bit big-endian
bytes 8..    name bytes
```

Big-endian means the most significant byte appears first. Endianness does not make data more correct; agreement does. The bounded widths expose limitations: names cannot exceed 65,535 bytes, age cannot exceed 255, and a native OCaml `int` may not fit in signed 32 bits.

## 4. Representing It in OCaml

Make failure explicit:

```ocaml
type decode_error =
  | Too_short of { needed : int; actual : int }
  | Unsupported_version of int
  | Invalid_name_length of int

let set_u16_be b off n =
  Bytes.set_uint8 b off (n lsr 8);
  Bytes.set_uint8 b (off + 1) (n land 0xff)

let get_u16_be b off =
  (Bytes.get_uint8 b off lsl 8) lor Bytes.get_uint8 b (off + 1)
```

For a full implementation, add `set_i32_be` and `get_i32_be` using `Int32` shifts. Convert to and from `int` only after checking bounds.

## 5. Build It

The encoder allocates exactly once:

```ocaml
let encode row =
  let name = Bytes.of_string row.name in
  let n = Bytes.length name in
  if n > 0xffff then invalid_arg "name too long";
  let out = Bytes.create (8 + n) in
  Bytes.set_uint8 out 0 1;
  Bytes.set_int32_be out 1 (Int32.of_int row.id);
  Bytes.set_uint8 out 5 row.age;
  set_u16_be out 6 n;
  Bytes.blit name 0 out 8 n;
  out
```

The decoder checks before slicing:

```ocaml
let decode b =
  let len = Bytes.length b in
  if len < 8 then Error (Too_short { needed = 8; actual = len })
  else if Bytes.get_uint8 b 0 <> 1 then
    Error (Unsupported_version (Bytes.get_uint8 b 0))
  else
    let name_len = get_u16_be b 6 in
    if name_len > len - 8 then Error (Invalid_name_length name_len)
    else Ok {
      id = Int32.to_int (Bytes.get_int32_be b 1);
      age = Bytes.get_uint8 b 5;
      name = Bytes.sub_string b 8 name_len;
    }
```

Notice that decoding ignores no trailing bytes by accident. Decide whether they represent another record or corruption; for this chapter, require `len = 8 + name_len`.

## 6. Run It

Round-trip several values:

```ocaml
let ada = { id = 7; name = "Ada"; age = 36 }
let bytes = encode ada
let restored = decode bytes
```

Print each byte as two hexadecimal digits. Manually locate version `01`, id `00000007`, age `24`, length `0003`, and UTF-8 bytes `41 64 61`. Repeat with an empty name and a multibyte name such as `"Zoë"`; string character count and byte length need not agree.

## 7. What Happened?

The layout converts a semantic record into an addressable byte sequence. Fixed-width fields can be read at known offsets. A length prefix permits the decoder to find the end of a variable field. The validation order matters: reading the advertised length before confirming the header exists would itself access outside the buffer.

We also acquired a file-format commitment. Reordering fields later would silently corrupt old rows unless the version changes or the decoder supports both layouts.

## 8. Measure It

Add counters for bytes encoded, bytes decoded, successful decodes, and errors. Generate rows with name lengths 0, 8, 128, and 4,096, then time 100,000 round trips. Compare encoded size with a textual representation such as `Printf.sprintf "%d,%s,%d"`. Text may be smaller for tiny integers and larger for large values; the important advantage here is deterministic parsing, not a universal compression win.

Check allocations with your preferred OCaml profiling method or compare `Gc.quick_stat ()` before and after a loop. `Bytes.sub_string` copies the name, a cost we accept for ownership safety.

## 9. Break It

Truncate a valid record at every possible byte and call `decode`. None should raise. Flip the version byte. Change the name length to `0xffff`. Encode age `300` and observe that `Bytes.set_uint8` rejects it. Try an id larger than signed 32-bit range; a careless `Int32.of_int` wraps.

Finally, decode bytes written in little-endian order. The record may look structurally valid while the id is nonsense. This is why a documented byte order is part of the format.

## 10. Improve It

Return encoding errors instead of `invalid_arg`, add exact range checks, and distinguish malformed UTF-8 if TinyDB promises valid text. A schema-aware codec could store nullable fields with a bitmap and several strings with an offset table. We postpone those features.

The urgent limitation is placement. Concatenating variable records into a file makes update and lookup offsets unstable. TinyDB next divides the file into fixed-size pages, the unit that disk and memory managers will exchange.

## 11. Exercises

1. Implement checked signed 32-bit conversion without relying on machine word size.
2. Add a nullable `email : string option` using one flag byte.
3. Write `decode_many` for length-prefixed records and reject a partial final record.
4. State whether duplicate field data, trailing bytes, and invalid UTF-8 are errors in your format.
5. Add a property test: every accepted row satisfies `decode (encode row) = Ok row`.

## 12. Checkpoint

TinyDB owns a versioned binary row format with fixed fields, one variable field, explicit endianness, and total decoding errors. Rows can survive process memory once stored, but we still lack stable placement and an I/O unit. The next abstraction is the page.
