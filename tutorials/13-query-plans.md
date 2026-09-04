# 13 — Query Plans

## 1. Why This Exists

TinyDB currently executes queries by manually composing OCaml functions. That entangles the user's request with one chosen algorithm. A query representation lets us validate names, display intent, transform equivalent expressions, and only later choose scans, indexes, or join implementations.

## 2. Mental Model

A logical plan says what relations and transformations produce the answer:

```text
Project(users.name)
        |
Filter(users.age > 30)
        |
Scan(users)
```

It does not say which heap pages to visit or which replacement policy to use. A physical plan adds those choices. Separating the two creates a seam where optimization can happen without changing language parsing or operator implementations.

## 3. Storage / Execution Model

Support a tiny surface language: `CREATE TABLE`, `INSERT`, `SELECT ... FROM ... WHERE ...`, `DELETE ... WHERE ...`, and one optional equality join. Parsing yields syntax with unresolved names. Binding consults the catalog and produces qualified columns. Logical planning produces relational operators. Physical planning later selects access paths.

The catalog stores table ids, schemas, heap roots, and index descriptions. It is metadata, not the rows themselves. This tutorial may construct syntax values directly; a hand-written parser can remain small and is not the database lesson.

## 4. Representing It in OCaml

```ocaml
type table_name = string
type column = { relation : string; name : string; position : int }

type predicate =
  | Eq of resolved_expr * resolved_expr
  | Gt of resolved_expr * resolved_expr
  | And of predicate * predicate

type logical_plan =
  | Scan of table_name
  | Filter of predicate * logical_plan
  | Project of column list * logical_plan
  | Join of logical_plan * logical_plan * predicate

type physical_plan =
  | Heap_scan of table_name
  | Index_scan of table_name * index_name * key
  | PFilter of predicate * physical_plan
  | PProject of column list * physical_plan
  | Nested_loop of physical_plan * physical_plan * predicate
  | Hash_join of physical_plan * physical_plan * equijoin_key
```

Separate types prevent the executor from receiving an unresolved logical node by mistake.

## 5. Build It

Write three small passes:

1. `bind : Catalog.t -> syntax_query -> (bound_query, error list) result`
2. `logical : bound_query -> logical_plan`
3. `physical_naive : logical_plan -> physical_plan`

The naive physical pass maps every scan to `Heap_scan`, preserves filters and projections, and maps every join to `Nested_loop`. This intentionally reproduces existing behavior.

Add a pure printer:

```ocaml
let rec pp_plan indent fmt = function
  | Scan name -> Format.fprintf fmt "%sScan(%s)" indent name
  | Filter (p, child) ->
      Format.fprintf fmt "%sFilter(%a)@,%a"
        indent pp_pred p (pp_plan (indent ^ "  ")) child
  | Project (cols, child) -> (* same shape *)
  | Join (left, right, p) -> (* print two children *)
```

Avoid wildcard match cases so adding an operator creates compiler reminders in every pass.

## 6. Run It

Construct the example selection and print it. Then bind an unqualified `id` in a join where both tables have `id`; return an ambiguous-column error with candidates. Bind `users.missing`; fail before opening a heap file.

Create two logically equivalent plans:

```text
Filter(age > 30, Scan(users))
Filter(age > 20, Filter(age > 30, Scan(users)))
```

The second has redundant work. Write a simple rewrite that combines adjacent filters into `And`. Confirm both produce the same multiset on test data.

## 7. What Happened?

The plan made data flow inspectable. Validation errors moved out of the hot row loop. Logical and physical decisions became distinguishable: `Scan(users)` states relation access, while `Index_scan(users, users_id, 7)` commits to an access path.

A tree also makes equivalences possible, but rewrites need laws. Pushing a filter below a join is valid only when its referenced columns all come from that child. Moving a projection too early can remove a later join key. A syntactically simple rewrite can change semantics.

## 8. Measure It

Planning usually processes far fewer nodes than execution processes rows, but measure parse time, bind time, logical node count, physical node count, and execution time separately. Generate plans with 1, 10, 100, and 1,000 nested filters to reveal recursive traversal and printer behavior.

For the example, print estimated and actual rows beside each node manually. Estimates are placeholders today; actual input/output counters come from chapter 8. This annotated tree becomes the basis of an `EXPLAIN ANALYZE`-style view.

## 9. Break It

Use unqualified names in a join and resolve “first match wins”; swapping child order changes meaning. Push `users.age > 30` into the orders child; binding or schema validation should reject it. Project away `user_id` before the join and observe a physical execution error if planning failed to check schemas.

Create a cyclic plan by changing the representation to mutable child references. Recursive printing never ends. Immutable variants prevent this entire class of invalid plan.

## 10. Improve It

Attach an output schema to every validated plan node. Track referenced-column sets to justify pushdown. Preserve source locations from parser tokens for useful errors. Plan ids allow counters and estimates to be associated without mutating the tree.

Plans describe operator structure but do not yet give every physical node one execution interface. The next chapter uses the Volcano iterator model—`open`, `next`, `close`—so a plan tree becomes a pull-based machine.

## 11. Exercises

1. Implement output-schema derivation for every logical node.
2. Bind qualified and unqualified columns with explicit ambiguity errors.
3. Write a safe adjacent-filter merge and test semantic equivalence.
4. Print plan trees with node ids, schemas, and actual row counters.
5. Extend the syntax with `DELETE` while reusing predicate binding.

## 12. Checkpoint

TinyDB represents relational meaning as immutable logical plans and algorithm commitments as a separate physical type. Binding catches schema errors before storage access, and plan trees make rewrites reviewable. Next, one iterator contract turns physical trees into executable pipelines.
