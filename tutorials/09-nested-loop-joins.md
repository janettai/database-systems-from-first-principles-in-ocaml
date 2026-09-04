# 09 — Nested-Loop Joins

## 1. Why This Exists

Suppose `users(id, name, age)` and `orders(id, user_id, total)` are separate tables. To return each order beside its owner, TinyDB must combine rows whose keys agree. The most direct program is also the definition we can trust as a correctness baseline: compare every left row with every right row.

## 2. Mental Model

```text
for each user
  for each order
    if user.id = order.user_id
      emit combined tuple
```

If the inputs contain `n` and `m` rows, the join performs `n * m` predicate checks. That does not mean it always takes the same time: page reads, buffering, predicate cost, and early termination matter. But the comparison count reveals the scaling law clearly.

## 3. Storage / Execution Model

A naive nested-loop join opens a fresh right scan for every left row. It keeps one left tuple and streams the right. Memory usage is small, but if the right relation exceeds the buffer pool, its pages may be reread for every left row. A materialized variant loads the right side into a list once, trading memory for fewer storage reads.

The output schema concatenates qualified left and right columns. Duplicate keys and repeated matches are valid: joins operate on bags unless a distinct operator says otherwise.

## 4. Representing It in OCaml

```ocaml
type join_stats = {
  mutable left_rows : int;
  mutable right_rows : int;
  mutable comparisons : int;
  mutable output_rows : int;
}

let combine left right = Array.append left right

val nested_loop :
  stats:join_stats ->
  left:tuple Seq.t ->
  right:(unit -> tuple Seq.t) ->
  predicate:(tuple -> tuple -> bool) ->
  tuple Seq.t
```

The right argument is a factory because each outer row needs a new scan. Passing one stateful sequence would exhaust it on the first left row.

## 5. Build It

One clear implementation uses a small flat-map helper:

```ocaml
let rec append a b () =
  match a () with
  | Seq.Nil -> b ()
  | Seq.Cons (x, rest) -> Seq.Cons (x, append rest b)

let rec flat_map f xs () =
  match xs () with
  | Seq.Nil -> Seq.Nil
  | Seq.Cons (x, rest) -> append (f x) (flat_map f rest) ()

let nested_loop ~stats ~left ~right ~predicate =
  flat_map (fun l ->
    stats.left_rows <- stats.left_rows + 1;
    right ()
    |> Seq.filter_map (fun r ->
         stats.right_rows <- stats.right_rows + 1;
         stats.comparisons <- stats.comparisons + 1;
         if predicate l r then begin
           stats.output_rows <- stats.output_rows + 1;
           Some (combine l r)
         end else None)) left
```

Production execution should propagate `result` errors and close child operators. Keep this version visible because its shape explains its costs.

## 6. Run It

Generate one order per user and join on integer equality. Try pairs `(100,100)`, `(1_000,1_000)`, and `(5_000,5_000)` only if the preceding run is tolerable. Consume the complete output and record elapsed time and comparisons.

Then consume only the first result. Change data ordering so the first match appears early, then late. Lazy execution can save comparisons for a limited query, but only if the physical order happens to cooperate.

Compare rescanning the heap on the right with materializing it once as an array.

## 7. What Happened?

For equal input sizes, increasing each by ten increases comparisons by roughly one hundred. Output size stayed linear in the one-to-one dataset, so most work produced nothing. This distinguishes intermediate work from result cardinality.

Materializing reduced page reads but did not reduce predicate comparisons. It may be faster until the right input no longer fits memory. Choosing the smaller relation as the outer input does not change `n * m`, though it changes how often the inner scan restarts and can affect I/O.

## 8. Measure It

Report left rows, right-row visits, comparisons, outputs, heap pages read, buffer hits/misses, and peak materialized rows. Plot or tabulate input size against comparisons. The exact timings may be noisy; the multiplication should be exact.

Create skewed sizes `(10,100_000)` and `(100_000,10)` under a tiny buffer pool. Swap outer and inner inputs. Explain the page-read difference from rescan count and cached working sets. Also run a predicate that never matches to prevent output allocation from dominating.

## 9. Break It

Join duplicate keys: two users with id 7 and three orders with user_id 7 must produce six rows. An implementation that inserts one row per key in a hash table later can accidentally lose duplicates.

Use a right sequence value instead of a factory. Only the first left row sees right rows. Next, make the right table larger than the cache and observe buffer thrashing. Finally, join on a non-equality predicate such as `user.age < order.total`; this will matter when choosing alternatives.

## 10. Improve It

Nested loops remain useful for tiny inputs, arbitrary predicates, and an inner side with an index. But equality joins expose structure: build a mapping from join key to all rows on one side, then probe it once per row on the other. That reduces expected comparisons from a product toward a sum, at the price of memory and hash constraints.

Keep nested loop as a reference implementation. Later optimizer tests can compare faster plans against it on small random data.

## 11. Exercises

1. Implement block nested loop: buffer several left rows, then scan the right once per block.
2. Derive page-read formulas for tuple-at-a-time and block variants.
3. Add a `LIMIT` consumer and measure best- and worst-case first match.
4. Test duplicate keys, empty inputs, and a many-to-many join.
5. Choose an outer input for two differently sized tables and justify using I/O, not only comparisons.

## 12. Checkpoint

TinyDB can join two streams with a correct, general nested-loop operator. It uses bounded memory but performs `O(n * m)` comparisons and may rescan disk pages. Measurements, duplicates, and non-equality predicates establish the baseline. Next, an equality-specific hash join trades memory for expected linear work.
