# repro

Minimal reproduction of the Temper Rust backend codegen bug.

The pattern that triggers the bug: `var` declared **inside** a `when` branch,
then captured by a `for` loop (which becomes a closure) within that branch.

The previous repro had `var first` at function-body level, which doesn't
trigger the bug because `varsDeclared()` finds it without traversing into
`NestingStatement` nodes. The bug only manifests when the `var` is nested
inside a `when` (a `NestingStatement`), because `varsDeclared()` skips those
but `functionsDeclared()` traverses them — causing `mutableCaptures()` to
find the closure but miss the variable it captures.

```temper
export interface Item {
  label(): String;
}

export class FooItem(public name: String) extends Item {
  public label(): String { name }
}

export class BarItem(public value: Int) extends Item {
  public label(): String { "${value}" }
}

// Bug trigger: var inside when branch, captured by for loop
export let formatItem(item: Item, extras: List<String>): String {
  let out = new StringBuilder();
  when (item) {
    is FooItem -> do {
      out.append(item.name);
      // var inside when branch + for loop = mutable capture inside NestingStatement
      var first = true;
      for (let extra of extras) {
        if (!first) { out.append(","); }
        first = false;
        out.append(extra);
      }
    };
    is BarItem -> do {
      out.append("bar:");
      out.append("${item.value}");
    };
    else -> out.append("?");
  }
  out.toString()
}
```
