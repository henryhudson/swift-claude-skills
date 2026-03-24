---
name: swift-fp
description: Apply functional programming principles to Swift code. Prefer immutability, pure functions, value types, map/flatMap/apply patterns, and higher-order functions. Use when writing, reviewing, or refactoring Swift code with a functional style.
metadata:
  author: Henry Hudson
  version: "1.0"
  source: "A Functional Programming Kickstart by Daniel Steinberg"
---

Apply functional programming principles when writing, reviewing, or refactoring Swift code. These principles come from "A Functional Programming Kickstart" and represent a practical, Swift-native approach to FP.

## Core Principles

### 1. Prefer Immutability

- Use `let` over `var` by default. Only use `var` when mutation is genuinely needed, and limit its scope.
- Use structs and enums (value types) over classes (reference types) unless reference semantics are specifically required.
- Make properties `let` not `var`. If external read access is needed but mutation should be internal, use `public private(set) var`.
- When mutation is needed inside a function, create a local mutable copy (`var copy = param`) rather than using `inout`.

```swift
// Prefer this
struct Card: Equatable {
    let rank: Rank
    let suit: Suit
}

// Over this
class MutableCard {
    var rank: Rank
    var suit: Suit
}
```

### 2. Write Pure Functions

A pure function:
- Returns the same output for the same input, every time
- Depends only on its explicit parameters (and `self` for methods)
- Has no side effects (no printing, no network calls, no modifying external state)

```swift
// Pure - output depends only on input
func changingRank(of card: Card, to rank: Rank) -> Card {
    Card(rank, of: card.suit)
}

// Impure - depends on external mutable state
func incrementRank() -> Rank {
    // uses and modifies external `yourCard` - avoid this
}
```

Pure functions enable **referential transparency**: any call like `changingRank(of: threeOfClubs, to: .queen)` can always be replaced with its result `Card(.queen, of: .clubs)`.

Gather impure code (I/O, UI state changes, network) in clearly identified boundaries. Keep the core logic pure.

### 3. Non-Mutating Over Mutating

Instead of changing a value in place, return a new value with the changes applied.

- Use imperative names for mutating methods: `sort()`, `remove()`
- Use "-ed"/"-ing" suffixes for non-mutating methods: `sorted()`, `removing()`

```swift
// Non-mutating style - preferred
extension Deck {
    func topped(with card: Card) -> Deck {
        [card] + self
    }

    func selectCard(at index: Int) -> (Card, Deck) {
        var deck = self
        let card = deck.remove(at: index)
        return (card, deck)
    }
}
```

When a non-mutating function needs to return multiple results, use tuples with descriptive names:
```swift
let (yourCard, remainingCards) = freshDeck.selectCard(at: 2)
```

### 4. Functions Are First-Class

Functions can be stored in properties, passed as parameters, and returned from other functions - just like `String` or `Int`.

```swift
// Storing a function
struct SimpleButton: View {
    let title: String
    let action: () -> Void
}

// Passing a function
freshDeck.moveToTheTop(the: queenOfHearts)
         .moveToTheTop(the: fourOfSpades)
```

### 5. Higher-Order Functions

Functions that accept or return other functions are called higher-order functions. Use them to:

**Return functions (partial application / currying):**
```swift
func line(slope m: Int, intercept b: Int) -> (Int) -> Int {
    { x in m * x + b }
}
let threeXPlusTwo = line(slope: 3, intercept: 2)
threeXPlusTwo(4) // 14
```

**Accept functions:**
```swift
func evaluate(_ x: Int, using f: (Int) -> Int) -> Int {
    f(x)
}
evaluate(4, using: threeXPlusTwo)
evaluate(4) { 3 * $0 + 2 } // trailing closure
```

Prefer **point-free style** when passing named functions: `.map(increment)` over `.map { increment($0) }`.

### 6. The Map Pattern (Functors)

`map()` is not just for Arrays. It is a **design pattern** for any type that wraps a value in a context.

**The signature pattern:**
```swift
func map<Output>(_ transform: (Input) -> Output) -> Context<Output>
```

If you have `f: (A) -> B`, then `map(f)` lifts it to `(Context<A>) -> Context<B>`.

**map() works on:**
- `Array` - transforms each element
- `Optional` - transforms the wrapped value if non-nil
- `Result` - transforms the success value
- Custom types (Writer, MagicHat, etc.)

**Functor laws** (any `map()` must satisfy these):
1. **Identity**: `x.map(identity) == identity(x)` - mapping identity changes nothing
2. **Composition**: `x.map(f).map(g) == x.map(f >>> g)` - mapping two functions sequentially equals mapping their composition

```swift
// map is NOT just a for loop - it's a pattern
freshDeck.map(increment)           // Array
someOptional.map(increment)        // Optional
someResult.map(increment)          // Result
// Same `increment` function works in every context!
```

### 7. FlatMap Pattern (Monads)

Use `flatMap()` when your transform returns a value already wrapped in the context, to avoid double-wrapping.

**When to use which:**
- Transform `(A) -> B` --> use `map()`
- Transform `(A) -> B?` --> use `compactMap()` (Arrays only)
- Transform `(A) -> [B]` --> use `flatMap()`
- Transform `(A) -> Context<B>` --> use `flatMap()` (general case)

```swift
// map would give [[Card]] - an Array of Arrays
hand.map(threeInARow)    // [[2S, 3S, 4S], [JH, QH, KH], ...]

// flatMap flattens to [Card]
hand.flatMap(threeInARow) // [2S, 3S, 4S, JH, QH, KH, ...]
```

**Code smell**: If you see `.flatMap { $0 }` after a `.map()`, replace the `.map()` with `.flatMap()` instead.

**Monad laws** (three laws for `just` and `flatMap`):
1. Left identity: `just(a).flatMap(f) == f(a)`
2. Right identity: `m.flatMap(just) == m`
3. Associativity: `m.flatMap(f).flatMap(g) == m.flatMap { x in f(x).flatMap(g) }`

### 8. Apply Pattern (Applicative Functors)

`apply()` sits between `map()` and `flatMap()` in power. It works when the function itself is also wrapped in a context.

```swift
// map:     f: (A) -> B,              value: F<A>  --> F<B>
// flatMap: f: (A) -> M<B>,           value: M<A>  --> M<B>
// apply:   f: Fplus<(A) -> B>,       value: Fplus<A> --> Fplus<B>
```

The key advantage: like `map()`, you use the same simple `(A) -> B` functions. Unlike `flatMap()`, you don't need context-specific transforms. The power comes from having the function itself in a context.

**Currying + apply enables multi-argument functions in context:**
```swift
// Curried add: (Int) -> (Int) -> Int
let curriedAdd = curry(+)

// Apply with Optional
Optional.some(curriedAdd) <*> Optional.some(3) <*> Optional.some(5)
// Optional.some(8)
```

**Hierarchy**: Every Monad is an Applicative Functor, and every Applicative Functor is a Functor.

### 9. Method Chaining

Design methods that return the same type to enable pipelines:

```swift
freshDeck
    .moveToTheTop(the: queenOfHearts)
    .moveToTheTop(the: fourOfSpades)

// Array pipelines
words
    .filter { $0.count > 3 }
    .map(emphasize)
    .sorted()
```

### 10. SwiftUI and FP

SwiftUI embodies FP principles:
- Views are structs (value types) with `let` properties
- `body` is a computed property - views are not mutated, they are recreated
- `@State` moves mutable storage outside the struct, preserving value semantics
- `@State` properties should always be `private`
- Button actions and closures are first-class function values

```swift
struct Counter: View {
    @State private var count = 0

    var body: some View {
        VStack {
            Text("\(count)")
            Button("Increment", action: increment)
        }
    }

    private func increment() { count += 1 }
}
```

## When Reviewing Code

Look for these FP improvement opportunities:

1. **`var` that could be `let`** - can this variable be made immutable?
2. **Class that could be struct** - does this genuinely need reference semantics?
3. **Side effects in functions** - does this function depend on or modify external state?
4. **Mutating where non-mutating would work** - can we return a new value instead?
5. **Manual loops over collections** - can `map`, `filter`, `reduce`, `flatMap`, `compactMap`, or `zip` express this more clearly?
6. **`.flatMap { $0 }`** - replace the preceding `.map()` with `.flatMap()` instead
7. **Repeated parameters** - could currying or partial application simplify the call site?
8. **Functions that could be chained** - does the method return its own type to enable pipelines?

## Philosophy

- FP is not about replacing OOP. Mix and match tools that serve you best.
- Value types are cheap in Swift and getting cheaper with compiler optimizations. Don't stress about "wasted" copies.
- Start small. "Oh look - this is just `map()`, let's use that here."
- Pure functions are easier to understand, combine, and test.
- Pace yourself. Add FP techniques a little at a time.
