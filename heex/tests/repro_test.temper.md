# repro_test

```temper
import { renderItems, FooItem, BarItem } from "../repro";

export test "renderItems formats mixed items" {
  let items = [
    new FooItem("hello") ~~ Item,
    new BarItem(42) ~~ Item,
    new FooItem("world") ~~ Item,
  ];
  assert renderItems(items) == "[hello, bar:42, world]";
}

export test "renderItems handles empty list" {
  let items: List<Item> = [];
  assert renderItems(items) == "[]";
}

export test "renderItems single item no comma" {
  let items = [new FooItem("only") ~~ Item];
  assert renderItems(items) == "[only]";
}
```
