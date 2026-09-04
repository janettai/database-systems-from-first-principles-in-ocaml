# 14 — Iterator Query Execution

## 1. Why This Exists

OCaml sequences worked for individual operators, but storage-backed execution needs initialization, deterministic cleanup, and structured failures. The Volcano model gives every physical operator the same lifecycle. A parent asks its child for one row; this pull travels down to storage and one result travels back up.

## 2. Mental Model

```text
Project.next()
      |
      v
Filter.next()
      |
      v
Scan.next() -> buffer pool -> heap
```

Filter may call its child repeatedly. Project calls once. Hash join's first `next` consumes its build child before probing. `close` travels through the tree even when the consumer stops early or an error occurs.

## 3. Storage / Execution Model

Define states: created, open, exhausted, and closed. `open_` allocates operator resources; `next` is valid only when open; `close` is idempotent. Each returned tuple is owned by the caller, so child pages may be unpinned before return.

The query runner opens the root, repeatedly pulls, and closes with `Fun.protect`. It can apply a result limit without teaching every operator about `LIMIT` initially.

## 4. Representing It in OCaml

Objects could express a common method set, but a record is enough:

```ocaml
type exec_error = Storage of string | Expression of string | Invalid_state

type operator = {
  open_ : unit -> (unit, exec_error) result;
  next : unit -> (tuple option, exec_error) result;
  close : unit -> unit;
  stats : unit -> operator_stats;
}

type operator_stats = {
  name : string;
  mutable calls : int;
  mutable rows : int;
}
```

A constructor such as `make_filter predicate child` closes over its cursor state. The physical-plan compiler recursively constructs these records.

## 5. Build It

Filter demonstrates pull behavior:

```ocaml
let make_filter predicate child =
  let opened = ref false in
  let stats = { name = "Filter"; calls = 0; rows = 0 } in
  let rec next () =
    if not !opened then Error Invalid_state
    else begin
      stats.calls <- stats.calls + 1;
      match child.next () with
      | Error _ as e -> e
      | Ok None -> Ok None
      | Ok (Some row) ->
          match predicate row with
          | Error e -> Error (Expression e)
          | Ok false -> next ()
          | Ok true -> stats.rows <- stats.rows + 1; Ok (Some row)
    end
  in
  {
    open_ = (fun () -> opened := true; child.open_ ());
    next;
    close = (fun () -> if !opened then child.close (); opened := false);
    stats = (fun () -> stats);
  }
```

Guard double-open and preserve a terminal error so repeated `next` calls do not resume unpredictably. Scan holds cursor coordinates; projection wraps one child result; nested loop reopens or resets its inner child; hash join builds on first open or first next.

## 6. Run It

Compile and run:

```text
PProject(name)
    PFilter(age > 30)
        Heap_scan(users)
```

Use a runner that prints rows until `Ok None`. Run with no limit and limit one, then display `next` calls and rows at every node. Filter's call count may exceed its output because rejected rows require additional pulls.

Inject an expression error after five rows and verify every operator closes. Repeat with the consumer itself raising while printing.

## 7. What Happened?

The plan became a tree of small state machines. Pull execution naturally pipelines scan, filter, and project: no intermediate relation is materialized. Early termination prevents upstream work. Blocking operators remain visible exceptions; hash join must consume build input, and sorting would consume all input.

Lifecycle complexity also became explicit. A sequence looked like a function, but pages and temporary structures require ownership. `Fun.protect` at the query boundary is part of correctness, not optional polish.

## 8. Measure It

For every node, record open calls, next calls, input rows, output rows, elapsed microseconds, and close calls. Attribute child time carefully: inclusive operator timing double-counts when summed. Exclusive timing requires subtracting child calls or measuring local work separately.

Compare streaming a million rows into a count with materializing them. Track peak live words or resident memory. Measure time to first row for scan/filter/project, nested loop, and hash join. These numbers explain pipelining more clearly than definitions alone.

## 9. Break It

Call `next` before `open_`, open twice, close twice, and call next after close. Each transition must have documented behavior. Stop after one row without close and inspect buffer pins. Create a filter that recursively skips one million rows; non-tail structure would overflow the stack, so verify the recursive call is in tail position or use a loop.

Let a child's close raise while another error is already propagating. TinyDB should make close best-effort and preserve the primary execution error while recording cleanup failure.

## 10. Improve It

Represent lifecycle in private constructors or an internal state variant. Add cancellation checked between pulls. An `Execution_context` can carry the buffer pool, transaction, counters, and memory budget instead of closing over globals. Batch-at-a-time operators could reduce call overhead later, but rows keep the teaching model clear.

The executor now coordinates reads, but updates remain individual physical mutations. A transfer that changes two rows needs a unit of success. Transactions supply that contract before we attempt logging.

## 11. Exercises

1. Implement scan and project constructors with strict lifecycle tests.
2. Compile every `physical_plan` variant into an operator tree.
3. Add `Limit` and prove it closes its child on early exhaustion.
4. Annotate a plan with actual rows and next-call counts.
5. Compare sequence and explicit-iterator implementations for resource safety.

## 12. Checkpoint

TinyDB executes physical plans through uniform pull-based operators with bounded streaming, explicit blocking points, counters, and deterministic cleanup. This completes the basic query path from plan to heap. The next requirement comes from multi-row updates: transaction boundaries and operational ACID guarantees.
