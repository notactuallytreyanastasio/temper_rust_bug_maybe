# Temper Rust Backend Codegen Bug — Minimal Reproduction

## The Bug

When Temper compiles to Rust, mutable variables (`var`) used inside `for`
loops inside `when` (pattern-match) blocks produce invalid Rust code that
fails with two classes of errors:

### Error 1: E0434 — `fn` items capture outer mutable variables

Temper's Rust codegen emits `fn` items inside function bodies for `when`
branches. These `fn` items reference the outer mutable variable `first`,
but Rust `fn` items cannot capture variables from their enclosing scope.
Only closures can do that.

**Generated (wrong):**
```rust
fn when_branch_0(first: &bool, out: &mut StringBuilder, item: &FooItem) {
    // ...references `first` from outer scope...
}
```

**Expected:** Either use closures, or pass the variable explicitly without
generating `fn` items that reference outer scope.

### Error 2: E0308 — Type mismatch with `Arc<RwLock<T>>`

Temper wraps mutable variables in `Arc<RwLock<bool>>` for the outer scope,
but in the `when`-block branch functions it emits `&bool` parameter types
instead of `&Arc<RwLock<bool>>`, causing a type mismatch.

**Generated (wrong):**
```rust
// Outer scope:
let first: Arc<RwLock<bool>> = Arc::new(RwLock::new(true));

// When-branch fn:
fn branch(first: &bool, ...) { ... }
//               ^^^^^ should be &Arc<RwLock<bool>>
```

## Triggering Pattern

The minimal pattern that triggers both bugs is `var` + `for` + `when`:

```temper
export let renderItems(items: List<Item>): String {
  var first = true;           // <-- mutable variable
  for (let item of items) {   // <-- for loop
    if (!first) { ... }
    first = false;
    when (item) {             // <-- when pattern match
      is FooItem -> ...;
      is BarItem -> ...;
      else -> ...;
    }
  }
}
```

Any one of these removed makes the bug disappear:
- Remove `var first` (no mutable variable) — compiles fine
- Remove `when` (no pattern match) — compiles fine
- Remove `for` (no loop) — compiles fine

## Files

| File | Purpose |
|------|---------|
| `heex/config.temper.md` | Package configuration |
| `heex/repro.temper.md` | Minimal code triggering the bug |
| `heex/tests/repro_test.temper.md` | Tests (pass on JS/Py/Lua, fail to compile on Rust) |

## Build Instructions

```bash
# These all work fine:
temper build -b js
temper build -b py
temper build -b lua

# This fails with 13+ errors (E0434, E0308):
JAVA_HOME=/opt/homebrew/opt/openjdk@21 temper build -b rust
```

## Environment

- Temper: latest (as of 2026-02-11)
- Rust: stable
- macOS (Apple Silicon)
