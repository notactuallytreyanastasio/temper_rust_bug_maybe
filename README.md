# Minimal repro for temperlang/temper#328

## What this is

This is a stripped-down reproduction of the Rust backend codegen bug I hit in
a larger project ([safe_httpeex](https://github.com/notactuallytreyanastasio/safe_httpeex)).

The first version of this repo didn't actually trigger the bug (thanks Tom for
catching that). Turns out it matters *where* the `var` is declared — it has to
be **inside a `when` branch**, not at the function body level. The original
repro had it at function body level so it compiled fine and didn't reproduce
anything lol.

## The actual pattern that breaks

```temper
// DOES trigger the bug — var inside when branch
let f(item: Item, extras: List<String>): String {
  when (item) {
    is FooItem -> do {
      var first = true;          // <-- inside the when (NestingStatement)
      for (let e of extras) {    // <-- for loop becomes a closure, captures first
        first = false;
      }
    };
  }
}

// Does NOT trigger — var at function body level
let g(items: List<Item>): String {
  var first = true;              // <-- this one is fine, varsDeclared finds it
  for (let item of items) {
    first = false;
    when (item) { ... }
  }
}
```

## How to see it

```bash
# these all pass fine (bug is Rust-specific):
temper test -b js
temper test -b py
temper test -b lua

# this fails on unfixed temper, passes on fixed:
temper test -b rust
```

## The errors you get

Two flavors, both from the same root cause:

**E0434** — the generated Rust has a `fn` inside a `ClosureGroup` struct that
tries to assign to `first__27` using the bare var name instead of
`self.first__27`. Rust `fn` items can't capture from enclosing scope:

```
error[E0434]: can't capture dynamic environment in a fn item
   --> src/mod.rs:107:17
    |
107 |                 first__27 = false;
    |                 ^^^^^^^^^
    |
    = help: use the `|| { ... }` closure form instead
```

**E0308** — type mismatch because `first` got declared as plain `bool` but the
closure generates `Arc<RwLock<bool>>` access patterns:

```
error[E0308]: mismatched types
   --> src/mod.rs:104:48
    |
104 |                 if ! temper_core::read_locked( & self.first__27) {
    |                                                ^^^^^^^^^^^^^^^^ expected `&Arc<RwLock<bool>>`, found `&bool`
```

## See also

- [temperlang/temper#328](https://github.com/temperlang/temper/issues/328) — the bug report
- [safe_httpeex](https://github.com/notactuallytreyanastasio/safe_httpeex) — the larger project where I originally hit this
