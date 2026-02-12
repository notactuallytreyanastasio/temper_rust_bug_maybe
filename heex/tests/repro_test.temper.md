# repro_test

```temper
let { formatItem, FooItem, BarItem, Item } = import("..");

test("formatItem FooItem with extras uses comma separator") {
  let extras = new ListBuilder<String>();
  extras.add("x");
  extras.add("y");
  extras.add("z");
  assert(formatItem(new FooItem("hello"), extras.toList()) == "hellox,y,z");
}

test("formatItem FooItem with no extras") {
  let extras = new ListBuilder<String>();
  assert(formatItem(new FooItem("hello"), extras.toList()) == "hello");
}

test("formatItem BarItem ignores extras") {
  let extras = new ListBuilder<String>();
  extras.add("ignored");
  assert(formatItem(new BarItem(42), extras.toList()) == "bar:42");
}
```
