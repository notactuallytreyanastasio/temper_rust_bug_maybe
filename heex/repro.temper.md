# repro

Minimal reproduction of the Temper Rust backend codegen bug.
The pattern `var` + `for` + `when` triggers two Rust compile errors.

```temper
interface Item {
  label(): String;
}

class FooItem(public name: String) extends Item {
  public label(): String { name }
}

class BarItem(public value: Int) extends Item {
  public label(): String { "${value}" }
}

export let renderItems(items: List<Item>): String {
  let out = new StringBuilder();
  out.append("[");
  var first = true;
  for (let item of items) {
    if (!first) { out.append(", "); }
    first = false;
    when (item) {
      is FooItem -> out.append(item.name);
      is BarItem -> do {
        out.append("bar:");
        out.append("${item.value}");
      };
      else -> out.append("?");
    }
  }
  out.append("]");
  out.toString()
}
```
