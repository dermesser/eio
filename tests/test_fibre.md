# Setting up the environment

```ocaml
# #require "eio_main";;
```

```ocaml
open Eio.Std

let run fn =
  Eio_main.run @@ fun _ ->
  traceln "%s" (fn ())
```

# Fibre.first

First finishes, second is cancelled:

```ocaml
# run @@ fun () ->
  let p, r = Promise.create () in
  Fibre.first
    (fun () -> "a")
    (fun () -> Promise.await p);;
+a
- : unit = ()
```

Second finishes, first is cancelled:

```ocaml
# run @@ fun () ->
  let p, r = Promise.create () in
  Fibre.first
    (fun () -> Promise.await p)
    (fun () -> "b");;
+b
- : unit = ()
```

If both succeed, we pick the first one:

```ocaml
# run @@ fun () ->
  Fibre.first
    (fun () -> "a")
    (fun () -> "b");;
+a
- : unit = ()
```

One crashes - report it:

```ocaml
# run @@ fun () ->
  Fibre.first
    (fun () -> "a")
    (fun () -> failwith "b crashed");;
Exception: Failure "b crashed".
```

```ocaml
# run @@ fun () ->
  Fibre.first
    (fun () -> failwith "a crashed")
    (fun () -> "b");;
Exception: Failure "a crashed".
```

Both crash - report both:

```ocaml
# run @@ fun () ->
  Fibre.first
    (fun () -> failwith "a crashed")
    (fun () -> failwith "b crashed");;
Exception: Multiple exceptions:
Failure("a crashed")
and
Failure("b crashed")
```

Cancelled before it can crash:

```ocaml
# run @@ fun () ->
  Fibre.first
    (fun () -> "a")
    (fun () -> Fibre.yield (); failwith "b crashed");;
+a
- : unit = ()
```

One claims to be cancelled (for some reason other than the other fibre finishing):

```ocaml
# run @@ fun () ->
  Fibre.first
    (fun () -> raise (Eio.Cancel.Cancelled (Failure "cancel-a")))
    (fun () -> "b");;
Exception: Cancelled: Failure("cancel-a")
```

```ocaml
# run @@ fun () ->
  Fibre.first
    (fun () -> Fibre.yield (); "a")
    (fun () -> raise (Eio.Cancel.Cancelled (Failure "cancel-b")));;
Exception: Cancelled: Failure("cancel-b")
```

Cancelled from parent:

```ocaml
# run @@ fun () ->
  let p, r = Promise.create () in
  Fibre.both
    (fun () ->
      failwith @@ Fibre.first
        (fun () -> Promise.await p)
        (fun () -> Promise.await p)
    )
    (fun () -> failwith "Parent cancel");
  "not-reached";;
Exception: Failure "Parent cancel".
```

Cancelled from parent while already cancelling:

```ocaml
# run @@ fun () ->
  Fibre.both
    (fun () ->
      let _ = Fibre.first
        (fun () -> "a")
        (fun () -> Fibre.yield (); failwith "cancel-b")
      in
      traceln "Parent cancel failed"
    )
    (fun () -> traceln "Cancelling parent"; failwith "Parent cancel");
  "not-reached";;
+Cancelling parent
Exception: Failure "Parent cancel".
```

Cancelling in a sub-switch. We see the exception as `Cancelled Exit` when we're being asked to cancel,
but just as plain `Exit` after we leave the context in which the cancellation started:

```ocaml
# run @@ fun () ->
  let p, r = Promise.create () in
  Fibre.both
    (fun () ->
      try
        Switch.run (fun _ ->
          try Promise.await p
          with ex -> traceln "Nested exception: %a" Fmt.exn ex; raise ex
        )
      with ex -> traceln "Parent exception: %a" Fmt.exn ex; raise ex
    )
    (fun () -> raise Exit);
  failwith "not-reached";;
+Nested exception: Cancelled: Stdlib.Exit
+Parent exception: Cancelled: Stdlib.Exit
Exception: Stdlib.Exit.
```

# Fibre.pair

```ocaml
# run @@ fun () ->
  let x, y = Fibre.pair (fun () -> "a") (fun () -> "b") in
  x ^ y
+ab
- : unit = ()
```

# Fibre.all

```ocaml
# run @@ fun () ->
  Fibre.all [];
  Fibre.all (List.init 3 (fun x () -> traceln "fibre %d" x));
  "done"
+fibre 0
+fibre 1
+fibre 2
+done
- : unit = ()
```

# Fibre.any

```ocaml
# run @@ fun () ->
  string_of_int @@
  Fibre.any (List.init 3 (fun x () -> traceln "%d" x; Fibre.yield (); x));
+0
+1
+2
+0
- : unit = ()
```

# Fibre.await_cancel

```ocaml
# run @@ fun () ->
  Fibre.both
    (fun () ->
       try Fibre.await_cancel ()
       with Eio.Cancel.Cancelled _ as ex ->
         traceln "Caught: %a" Fmt.exn ex;
         raise ex
    )
    (fun () -> failwith "simulated error");
  "not reached"
+Caught: Cancelled: Failure("simulated error")
Exception: Failure "simulated error".
```