+++
title = "Choosing the first test"
slug = "choosing-the-first-test"
date = "2022-02-05"
category = "kata"

[extra]
author = "Bob Gregory"

[taxonomies]
tags = ["tdd", "basics"]
+++

I spend a lot of time teaching TDD to engineers. Some of my most rewarding sessions are spent with first-time TDD practitioners. When we're starting out, I'll choose a simple TDD kata, and we'll pair to solve it together.

Once I've introduced myself and the exercise, I always ask the same opening question: "What's the first test we should write?"

I've found that TDD novices tend to make three common mistakes when starting their test suites.

<!-- more -->

## Asserting the trivial

The most common mistake is to focus on testing things that are necessarily, or trivially, true. For example, in the Tic-Tac-Toe kata, the conversation might go like this:

> What's the first test we should write?

> I think I need to test that we can create a board

```ts
describe("when starting a game", () =>{
    const board = new Board()
    
    it("should exist", () => {
      expect(board).not.toBeUndefined()
    })
})
```

Similarly, sometimes beginners want to assert the type of their newly created object.

```python
def test_can_create_board():
    board = Board()

    assert type(board) is Board
```

### What's wrong with these tests?

These tests don't help us to specify our solution. Instead, they're testing that the programming language works as it should. The only thing that could fail this test is a compiler error.

I think the thought process runs: "I need to create a class, therefore I need to test that I create the class". The engineer is looking for a test that will cause them to write the code they've already decided on.

This is back to front, though. Instead, we want to write a test that _specifies the behaviour of our code_. The test will tell us what the design ought to be.  We want to write the test as though our perfect code already existed, and then make it work.

One step up from testing the abstract concept of constructors is testing the specifics of our constructor. 

```ts
describe("when creating a 3 x 3 board", () => {
    const board = new Board(3, 3);
    
    it("should be the right size", () => {
        expect(board.height).toBe(3);
        expect(board.width).toBe(3);
    })
})
```

This is a little better, but still doesn't help us very much. Again, this test follows from the thought process "I need to create a board, and it needs to be 3x3". The test is focused on verifying data, but we ought to be checking behaviours instead.

### What could we do differently?

When we're looking for our first test, we should focus on a behaviour that we want from the system. That behaviour shouldn't be something that's guaranteed by the programming language (the `new` keyword returns an object) or something trivial (I set fields in the constructor). It should be something that moves us meaningfully forward. For example

```ts
describe("When starting a new game", () => {

    const game = new Game();
    
    it("should alternate moves between the players", () => {
        expect(game.nextMove).toBe(Player.X);
        game.play(0,0);
        expect(game.nextMove).toBe(Player.Y)
    })
})
```

This is _simple_ to implement, but forces us to build a key mechanism of the game - players take it in turns to move. We don't have to build any logic for the win condition, or even for updating the board, we just have to pass play between X and O.

```ts
enum Player {
  X,
  O
}

class Game {
  #nextMove: Player = Player.X;
  
  public play(x: number, y: number) {
    if(this.#nextMove == Player.X) {
      this.#nextMove = Player.Y;
    } else {
      this.#nextMove = Player.X;
    }
  }
}
```

For our next test we might check that the board is updated when we play

```ts
it("should put the player's token on the board", () => {
    expect(game.get(0, 0)).toBe(Player.X);
})
```

And now we're off and away!

## Negative outlook

The second mistake I see in TDD first-timers is starting with an error case. For example, we might choose the Leap Year kata. In this kata, we're asked to build code that can answer, true or false, whether a given year is a leap year.

> What's the first test that we should write?

> I need to make sure that the year is a number.

```ts
describe("When the year is a string", () => {
    expect(() => isLeapYear("poop")).toThrow();
})
```

I think this stems from engineers believing that tests are about "checking correctness". In traditional testing, we spend a lot of time looking for boundary cases - what happens when we use an empty string, a negative number, a string that's too long, and so on. The thought process is "The leap year function takes a number, I need to test that it can only take a number".

### What's wrong with these tests?

There are an infinite number of things that our code does _not_ do. `isLeapYear` does not accept strings, it does not tell you whether a number is prime, it does not accept arbitrary numbers of arguments, it does not generate pictures of chimps, and so on.

We'll never get anywhere by declaring the things our code does _not_ do. Instead, we want to choose a positive behaviour: a thing that our code _does_ do.

```ts
test("a year is a leap year if divisible by 4", () => {
    expect(isLeapYear(4)).toBeTrue();
    expect(isLeapYear(16)).toBeTrue();
    expect(isLeapYear(64)).toBeTrue();
    expect(isLeapYear(4000000004)).toBeTrue();
})

const isLeapYear (n:number) => {
  return n % 4 == 0
}
```

Our next test might introduce one of the exceptions to the rule.
```
test("a year is not a leap year if divisible by 400")
```

Choosing a positive behaviour helps us to move our code in the right direction. We can worry about error cases at the very end, if at all.

## Biting off more than they can chew

The last mistake I see a lot is beginners being too ambitious in their choice of first test. For example, when practising domain modelling techniques, we might use the Poker Hands Kata. This kata asks us to play one round of poker, scoring each player's cards as follows:

{% paper(type="quote") %}
A poker hand consists of 5 cards dealt from the deck. Poker hands are ranked by the following partial order from lowest to highest.

* High Card: Hands which do not fit any higher category are ranked by the value of their highest card. If the highest cards have the same value, the hands are ranked by the next highest, and so on.
* Pair: 2 of the 5 cards in the hand have the same value. Hands which both contain a pair are ranked by the value of the cards forming the pair. If these values are the same, the hands are ranked by the values of the cards not forming the pair, in decreasing order.
* Three of a Kind: Three of the cards in the hand have the same value. Hands which both contain three of a kind are ranked by the value of the 3 cards.
* Flush: Hand contains 5 cards of the same suit. Hands which are both flushes are ranked using the rules for High Card.
* Full House: 3 cards of the same value, with the remaining 2 cards forming a pair. Ranked by the value of the 3 cards.

{% end %}

> What's the first test we should write?

> I think I want to tackle "Full House"

```ts
describe("When a player has a full house", () => {
    const player1 = ["2H", "2C", "2D", "3S", "3D"]
    const player2 = ["5C", "4C", "QC", "KC", "9C"]
    
    it("should beat a Flush", () => {
        const game = new Poker();
        expect(poker.play(player1, player2)).toEqual("Player 1 wins with Full House (3)");
    }
})
```

Here the thinking is "If I tackle the most complex case first, that will flush out all the rest of the behaviour".

### What's wrong with this test?

A test that requires us to build the whole system isn't very helpful. Think of the problem we're trying to solve as a mountain to be climbed. The mountain is far too big for us to walk up it in a single step. We need to break the problem down into manageable pieces.

Our tests are like a staircase that we carve into the mountainside. Each passing test lifts us one tiny step closer to the summit. If we lose our footing, we can always return to the last safe place - the last passing test - and try again.

This test asks us to:

* Parse card values like `9D` or `QC`
* Find groups of cards in a hand, like "pairs", or "triples", or sets of the same suit.
* Identify the highest ranked card in a hand
* Be able to identify that a full house comprises a pair and a triple
*  Be able to identify that a flush comprises 5 cards of the same suit
* Write a rule that decides that a flush is worth less than a full house
* Output a well-formed string describing the winning player's hand.

That's a lot of mountain to be climbed!

### What could we do instead?

Our first test should be the tiniest increment that moves us forward meaningfully. 

```ts

describe("When comparing cards", () => {
  const cards = [
    [5, "spades"],
    [3, "clubs"],
    [9, "diamonds"],
    [4, "hearts"],
  ];
  it("should order them by rank", () => {
    expect(cards.sort(compareCards)).toEqual([
      [3, "clubs"],
      [4, "hearts"],
      [5, "spades"],
      [9, "diamonds"],
    ]);
  });
});

type Rank = number
type Suit = "diamonds" | "hearts" | "clubs" | "spades";
type Card = [rank: Rank, suit: Suit];

function compareCards(a: Card, b: Card) {
  if (rank(a) > rank(b)) return 1;
  if (rank(b) > rank(a)) return -1;
  return 0;
}

const rank (c: Card) => return c[0];
```

This test establishes that cards have a rank and a suit, and that one card can be bigger than another, so that we can have a partial ordering over them.

From here we would probably handle the face cards like `["queen", "hearts"]` or `["ace", "spades"]` and then tackle a test like `the hand with the highest card wins`.

# Summary

The first test of our test suite will often set the direction that our code takes. It's important to choose something that sends us on a useful path.

The first test of any new suite should be _the smallest positive behaviour that moves us meaningfully forward_.
