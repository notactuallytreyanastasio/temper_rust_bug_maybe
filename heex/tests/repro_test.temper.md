# repro_test

```temper
let { renderItems, FooItem, BarItem, Item } = import("..");

test("renderItems formats mixed items") {
  let items = [
    new FooItem("hello") as Item,
    new BarItem(42) as Item,
    new FooItem("world") as Item,
  ];
  assert(renderItems(items) == "[hello, bar:42, world]");
}

test("renderItems handles empty list") {
  let items: List<Item> = [];
  assert(renderItems(items) == "[]");
}

test("renderItems single item no comma") {
  let items = [new FooItem("only") as Item];
  assert(renderItems(items) == "[only]");
}
```
