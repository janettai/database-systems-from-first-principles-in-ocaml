# 08 — Selection and Projection

## 1. Why This Exists

Users rarely want every column of every row. For:

```sql
SELECT name FROM users WHERE age > 30;
```

TinyDB needs two relational operators. Selection keeps rows satisfying a predicate. Projection transforms each surviving row into fewer columns. Implementing them over the scan exposes an important fact: composition order changes physical work even when the answer is equivalent.

## 2. Mental Model

The query is a flow, read bottom-up:

```text
scan users
    |
filter age > 30
    |
project name
```

Each operator pulls from its child. Filter may consume many input rows before yielding one. Project consumes exactly one input for each output. Neither needs to know pages, slots, or files; their input is a stream of typed rows.

## 3. Storage / Execution Model

To avoid hard-coding the `users` record into every later plan, introduce a small dynamic row representation:

```ocaml
type value = Int of int | Text of string | Null
type tuple = value array
type schema = (string * column_type) array
```

Column positions are resolved against a schema once, not looked up by string for every row. Predicates evaluate with SQL-like typed operations. We postpone three-valued `NULL` logic; for now comparisons with `Null` are false and this limitation is explicit.

## 4. Representing It in OCaml

Separate an expression from its compiled function:

```ocaml
type expr =
  | Column of string
  | Literal of value
  | Gt of expr * expr
  | Eq of expr * expr

type projection = string list

val compile_predicate : schema -> expr ->
  (tuple -> (bool, string) result, string) result
```

Compilation resolves `Column "age"` to an array offset and rejects missing columns or invalid static type combinations before scanning data.

## 5. Build It

The streaming operators are small:

```ocaml
let select metrics predicate input =
  Seq.filter_map (function
    | Error _ as e -> Some e
    | Ok row ->
        metrics.rows_examined <- metrics.rows_examined + 1;
        match predicate row with
        | Ok true -> metrics.rows_selected <- metrics.rows_selected + 1; Some (Ok row)
        | Ok false -> None
        | Error e -> Some (Error (`Expression e))) input

let project positions input =
  Seq.map (Result.map (fun row ->
    Array.of_list (List.map (Array.get row) positions))) input
```

For strict error semantics, `select` should stop after an error rather than allow later rows. A small `Seq_result` module can provide `map`, `filter`, and `fold` with consistent short-circuiting.

Compile the example into positions once: age might be column 2 and name column 1.

## 6. Run It

Populate users with varied ages and execute:

```ocaml
let input = Table.scan_tuples users in
let filtered = select metrics age_over_30 input in
let names = project [1] filtered in
consume names
```

Print the operator pipeline and counters. Then ask only for the first matching name. The filter should stop pulling once project yields it.

Try a nonexistent column and `age > "thirty"`. These should fail during compilation, before the heap reads a page.

## 7. What Happened?

OCaml sequence composition mirrors relational algebra cleanly: filter is selection and map is projection. Yet the storage cost remains in the scan. If one user matches, TinyDB may still inspect the entire table because no access path can find that user directly.

Operator order also affects CPU and allocation. Filtering before projection avoids building projected tuples for rejected rows. But if projection computes a compact key required by a cheap predicate, another order could help. Logical equivalence does not erase physical cost.

## 8. Measure It

Add per-operator counters: input rows, output rows, predicate evaluations, projected values, and expression errors. Run predicates matching approximately 100, 10, 1, and 0 percent of a 100,000-row table. The scan reads should remain similar for full consumption, while produced rows and projection work change with selectivity.

Compare two pipelines: `project after filter` and a deliberately wasteful projection before filter that preserves the age column. Record allocations or elapsed time. Also time name lookup by string on every row versus one compiled integer position.

## 9. Break It

Put `Null` in the age column and ask whether `age > 30` should be false, true, or unknown. Our two-valued shortcut cannot represent SQL's unknown. Insert a `Text` where an `Int` is expected by bypassing the table encoder; runtime evaluation must return a type error, not raise `Match_failure`.

Create an expensive predicate with a visible counter, then place it before a highly selective cheap predicate. Both orders return the same rows, but one performs far more expensive calls. This is the first small optimization problem.

## 10. Improve It

Introduce typed schemas and a validation phase that converts named expressions into resolved expressions such as `RColumn 2`. Preserve source names for good errors. Add conjunction and evaluate the cheapest or most selective conjunct first only when doing so preserves error semantics.

Single-table operators compose well, but combining relations creates a new cost explosion. The simplest join compares every left row with every right row. We implement it next so its poor scaling is observed before introducing a faster algorithm.

## 11. Exercises

1. Implement resolved expressions and reject duplicate or missing column names.
2. Add `And`, `Or`, and `Not`; specify short-circuit and error behavior.
3. Model SQL truth as `True | False | Unknown` and define filtering.
4. Measure predicate ordering with cheap/selective and expensive/unselective pairs.
5. Add output schema calculation for projection and qualify names such as `users.name`.

## 12. Checkpoint

TinyDB now expresses scan, selection, and projection as a streaming pipeline over dynamic tuples. Schema resolution moves name and type errors before execution; counters expose selectivity and wasted expression work. Full scans still dominate access, and relations cannot yet be combined. A nested-loop join supplies the naive baseline.
