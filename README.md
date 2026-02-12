# Temper Rust Backend Codegen Bug — Minimal Reproduction

## The Bug

When Temper compiles to Rust, mutable variables (`var`) declared **inside a
`when` branch** (a `NestingStatement`) and captured by a `for` loop (which
becomes a closure) within that branch produce invalid Rust code.

### Root Cause

In `TmpLHelpers.kt`, `varsDeclared()` skipped `NestingStatement` nodes (when
blocks, if/else, while loops), but `functionsDeclared()` did NOT skip them.
This asymmetry caused `mutableCaptures()` to:
- **Find** closures inside when branches (via `functionsDeclared()`)
- **Miss** the mutable variables they capture (via `varsDeclared()`)

When a variable isn't detected as a mutable capture, the Rust backend declares
it as a plain type (`bool`), but the closure still generates `Arc<RwLock<bool>>`
access patterns — causing E0308 type mismatches and E0434 fn-item capture errors.

### Key Detail: Where `var` Is Declared Matters

The bug only triggers when `var` is declared **inside** a `when` branch:

```temper
// DOES trigger the bug — var inside when branch
let f(item: Item, extras: List<String>): String {
  when (item) {
    is FooItem -> do {
      var first = true;          // <-- inside NestingStatement
      for (let e of extras) {    // <-- closure captures first
        first = false;
      }
    };
  }
}

// Does NOT trigger — var at function body level
let g(items: List<Item>): String {
  var first = true;              // <-- at function body level (found without nesting)
  for (let item of items) {
    first = false;
    when (item) { ... }
  }
}
```

### Error 1: E0434 — `fn` items capture outer mutable variables

```
error[E0434]: can't capture dynamic environment in a fn item
   --> src/mod.rs:107:17
    |
107 |                 first__27 = false;
    |                 ^^^^^^^^^
    |
    = help: use the `|| { ... }` closure form instead
```

### Error 2: E0308 — Type mismatch with `Arc<RwLock<T>>`

```
error[E0308]: mismatched types
   --> src/mod.rs:104:48
    |
104 |                 if ! temper_core::read_locked( & self.first__27) {
    |                                                ^^^^^^^^^^^^^^^^ expected `&Arc<RwLock<bool>>`, found `&bool`
```

## Fix

The fix in `temperlang/temper` adds an `includeNesting` parameter to
`varsDeclared()` and passes it from `mutableCaptures()`, so mutable variables
inside `when` branches are correctly detected as captures.

See: [temperlang/temper#328](https://github.com/temperlang/temper/issues/328)

## Build Instructions

```bash
# These all work fine (bug is Rust-specific):
temper test -b js
temper test -b py
temper test -b lua

# This fails on unfixed temper, passes on fixed:
temper test -b rust
```

## Files

| File | Purpose |
|------|---------|
| `heex/config.temper.md` | Package configuration |
| `heex/repro.temper.md` | Minimal code triggering the bug |
| `heex/tests/repro_test.temper.md` | Tests (pass on JS/Py/Lua, fail to compile on unfixed Rust) |
