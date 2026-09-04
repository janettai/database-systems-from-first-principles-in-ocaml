# 10 — Hash Joins

## 1. Why This Exists

The nested-loop join repeatedly asks a question whose answer can be organized: “which right rows have this key?” For equality joins, building that organization once avoids nearly all failed pairwise comparisons. The resulting hash join introduces a recurring database choice: spend memory now to avoid repeated work later.

## 2. Mental Model

```text
smaller input --build--> key -> matching rows
                              ^
larger input ----probe--------|
```

Build consumes one input and stores every row in a hash table keyed by the join expression. Probe streams the other input, computes its key, and retrieves only candidate matches. Duplicate keys map to lists, not single rows.

## 3. Storage / Execution Model

If build has `n` rows and probe has `m`, expected work is roughly `O(n + m + output)`, assuming a reasonable hash distribution. Memory is `O(n)`. Build is a pipeline breaker: no output appears until the entire build side has been consumed. Probe remains streaming.

TinyDB supports `Int` and `Text` equality keys and excludes `Null` from matches. A physical planner should normally build the smaller input, but estimates can be wrong.

## 4. Representing It in OCaml

OCaml's polymorphic hashing would work, but an explicit key keeps semantics visible:

```ocaml
type key = KInt of int | KText of string

module Key = struct
  type t = key
  let equal a b = a = b
  let hash = Hashtbl.hash
end

module H = Hashtbl.Make (Key)

type stats = {
  mutable build_rows : int;
  mutable probe_rows : int;
  mutable lookups : int;
  mutable candidate_rows : int;
  mutable output_rows : int;
}
```

Key extraction returns `key option`; `None` represents null or an unsupported value.

## 5. Build It

Preserve duplicates when building:

```ocaml
let add table key row =
  let rows = Option.value (H.find_opt table key) ~default:[] in
  H.replace table key (row :: rows)

let hash_join ~stats ~build ~probe ~build_key ~probe_key =
  let table = H.create 128 in
  Seq.iter (fun row ->
    stats.build_rows <- stats.build_rows + 1;
    Option.iter (fun key -> add table key row) (build_key row)) build;
  probe
  |> Seq.flat_map (fun row ->
       stats.probe_rows <- stats.probe_rows + 1;
       match probe_key row with
       | None -> Seq.empty
       | Some key ->
           stats.lookups <- stats.lookups + 1;
           H.find_opt table key |> Option.value ~default:[] |> List.to_seq
           |> Seq.map (fun built ->
                stats.candidate_rows <- stats.candidate_rows + 1;
                stats.output_rows <- stats.output_rows + 1;
                combine built row))
```

Copy rows into the table so their bytes do not depend on buffer pins. Reverse per-key lists if stable build order is a chosen contract; relational results otherwise remain unordered.

## 6. Run It

Use the same datasets as chapter 9 and compare outputs as multisets. At 100, 1,000, and 5,000 one-to-one rows, record nested-loop comparisons against hash lookups. The hash join should perform one lookup per non-null probe row.

Repeat with ten orders for every user. Candidate rows and output now grow, because no algorithm can avoid producing required results. Swap build and probe and compare memory estimates and time.

## 7. What Happened?

Hashing replaced repeated unsuccessful comparisons with a one-time classification of the build input. It did not make joins free: hashing variable strings costs bytes, table entries allocate, and high-output joins still do high-output work. First-row latency increased because build must complete before probe begins.

The algorithm is specialized. A predicate like `left.age < right.total` does not identify a single hash bucket. Nested loops still handle it, and sorted inputs could motivate merge join later.

## 8. Measure It

Track build rows, distinct keys, duplicate rows, estimated key bytes, probe rows, hash lookups, candidates, outputs, and elapsed time. Report time to first row separately from time to completion. Hash join can win throughput and lose startup latency.

Generate a skewed build where 90 percent of rows share one key. One probe for that key visits a huge candidate list. Hash lookup remains constant-ish, but join output is not. Compare building 10,000 large tuples with building only `(key, record_id)` pairs and fetching records later.

## 9. Break It

Replace the per-key list with a single row; duplicate-key results disappear. Mutate a `Bytes.t` used inside a hashed key after insertion; its hash/equality behavior may no longer match its bucket. Use immutable strings or copy key bytes.

Try a build side larger than available memory. This implementation either consumes excessive memory or fails. A production engine partitions inputs to disk—grace hash join—but that is outside our core implementation. The planner must retain nested loop as a bounded-memory fallback.

## 10. Improve It

Project build rows to only columns needed above the join. Estimate row width as well as row count when choosing the build side. Pre-size the table from estimated cardinality to reduce resizing. For small build sides, a compact array may outperform hashing despite worse asymptotics.

Hash join accelerates combining whole relations, but a point query still scans a table. Persistent or reusable access paths solve that different problem. We begin with a simple in-memory index from key to record ids.

## 11. Exercises

1. Compare hash-join output with nested loop as multisets on random duplicate-heavy data.
2. Implement composite keys `(user_id, status)`.
3. Add a memory budget that rejects or falls back before building.
4. Measure time to first row and explain the pipeline break.
5. Describe when an index nested-loop join could beat both current algorithms.

## 12. Checkpoint

TinyDB has two physical join algorithms with different domains and costs. Hash join gives expected near-linear equality-join work by materializing one side, while nested loop remains general and bounded-memory. The next subsystem makes point access reusable across queries: an index.
