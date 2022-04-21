+++
title = "Approval testing with Jest Snapshots"
slug = "approval-testing-with-jest-snapshots"
date = "2022-04-21"
category = "tdd"

[extra]
author = "Bob Gregory"

[taxonomies]
tags = ["tdd", "kata", "jest"]
+++

[Approval testing](https://approvaltests.com/) is a powerful technique for working with legacy systems. When refactoring, it's important that we have tests to tell us if we've changed behaviour, but sometimes code can be smelly enough that writing unit tests is hard. Approval tests fill this gap. Instead of writing behavioural unit tests, we capture the output of the system - usually as text - and run a diff tool to check for changes in the output as we change the code.

I've previously struggled to set up approval tests with the common tools and libraries, but I realised today that Jest Snapshots solve _exactly_ the same problem, and are easy to use.
<!-- more -->

Taking the famous [Gilded Rose Kata](https://github.com/emilybache/GildedRose-Refactoring-Kata) as an example, we want to write a snapshot of the output of our system, and to check that the output is unchanged when we refactor.

```ts
import { GildedRose, Item } from "./gilded-rose";


// These are the items as described by the kata

const items = [
  new Item("+5 Dexterity Vest", 10, 20),
  new Item("Aged Brie", 2, 0),
  new Item("Elixir of the Mongoose", 5, 7),
  new Item("Sulfuras, Hand of Ragnaros", 0, 80),
  new Item("Sulfuras, Hand of Ragnaros", -1, 80),
  new Item("Backstage passes to a TAFKAL80ETC concert", 15, 20),
  new Item("Backstage passes to a TAFKAL80ETC concert", 10, 49),
  new Item("Backstage passes to a TAFKAL80ETC concert", 5, 49),

  // This Conjured item does not work properly yet
  new Item("Conjured Mana Cake", 3, 6),
];


// This function prints the state of our items
// to a string

const printState = (day: string) => {
  const result = [`-------- day ${day} --------`];
  result.push("name, sellIn, quality");
  items.forEach((item) =>
    result.push(`${item.name}, ${item.sellIn}, ${item.quality}`)
  );
  return result.join("\n");
};

const gildedRose = new GildedRose(items);
const days = [...Array(20).keys()];

for (const d in days) {

  // run an approval test for this day
  test(`Approval: Day ${d}`, () => {
    expect(printState(d)).toMatchSnapshot(`day-${d}`);
  });

  // and update the quality
  gildedRose.updateQuality();
}
});

```
Running this for the first time will generate our approval files.

```console
bob@bobs-spangly-carbon> npx jest
 PASS  test/acceptance.spec.ts
 › 20 snapshots written.

----------|---------|----------|---------|---------|-------------------
File      | % Stmts | % Branch | % Funcs | % Lines | Uncovered Line #s 
----------|---------|----------|---------|---------|-------------------
All files |     100 |    72.72 |     100 |     100 |                   
 index.ts |     100 |    72.72 |     100 |     100 | 10-12             
----------|---------|----------|---------|---------|-------------------

Snapshot Summary
 › 20 snapshots written from 1 test suite.

Test Suites: 2 passed, 2 total
Tests:       21 passed, 21 total
Snapshots:   20 written, 20 total
Time:        0.48 s, estimated 1 s
Ran all test suites.
```

And now we can confidently make changes to our code. For example, the first if statement in the code reads

```ts
if (
  this.items[i].name != "Aged Brie" &&
  this.items[i].name != "Backstage passes to a TAFKAL80ETC concert"
)
```

If we remove the `name != "Aged Brie"`from this conditional, we'll get 20 failed snapshot tests with helpful diffs on the command line

```console
 ● Gilded rose › Approval: Day 1

    expect(received).toMatchSnapshot(hint)

    Snapshot name: `Gilded rose Approval: Day 1: day-1 1`

    - Snapshot  - 1
    + Received  + 1

    @@ -1,9 +1,9 @@
      -------- day 1 --------
      name, sellIn, quality
      +5 Dexterity Vest, -10, 0
    - Aged Brie, -18, 38
    + Aged Brie, -18, 1
      Elixir of the Mongoose, -15, 0
      Sulfuras, Hand of Ragnaros, 0, 80
      Sulfuras, Hand of Ragnaros, -1, 80
      Backstage passes to a TAFKAL80ETC concert, -5, 0
      Backstage passes to a TAFKAL80ETC concert, -10, 0

      48 |     gildedRose.updateQuality();
      49 |   }
    > 50 | });
         |              ^
      51 |

      at Object.<anonymous> (test/acceptance.spec.ts:50:29)
```

I'm surprised this didn't occur to me before but, as a Jest user, it will definitely save me some time setting up approval tests in the future.
