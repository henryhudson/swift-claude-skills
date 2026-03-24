---
name: swift-kickstart
description: Apply Daniel Steinberg's idiomatic Swift patterns. let over var, type inference, value types, protocol extensions, guard let, named tuples, higher-order functions, and expressive API design. Use when writing or reviewing Swift code.
metadata:
  author: Henry Hudson
  version: "1.0"
  source: "A Swift Kickstart (2nd Edition) by Daniel Steinberg"
---

Apply idiomatic Swift patterns from "A Swift Kickstart" when writing, reviewing, or refactoring Swift code. These represent Daniel Steinberg's opinionated, practical approach to writing "Swifty" code.

## Core Principles

### 1. Express Your Intentions

Swift requires and rewards expressing intent clearly. Every choice — `let` vs `var`, struct vs class, naming — communicates meaning.

### 2. Let Over Var, Always

Start with `let`. Only change to `var` when you're sure the value needs to change.

```swift
// Start here
let count = items.count

// Only if you truly need mutation
var total = 0
for item in items { total += item.price }
```

- If publicly accessible, prefer `let`. Use `private var` internally when mutation is needed.
- Constants are easier to reason about. If something never changes, say so.

### 3. Let Swift Infer Types

If Swift can infer the type, let it. Only declare types explicitly when inference isn't possible.

```swift
// Preferred - inferred
let name = "Swift Programmer"
let primes = [2, 3, 5, 7]
let greetings = [hello, bonjour]

// Only when needed - no initial value
let shouldBeRed: Bool
// or empty collections
var drinkSizes = [String]()
```

### 4. Functions Are First-Class

Functions exist independently of classes. They have types determined by parameters and return values. They can be stored, passed, and returned.

```swift
// Functions have types
func hello(name: String) -> String { "Hello, \(name)!" }
func bonjour(name: String) -> String { "Bonjour, \(name)!" }

// Store in collections
let greetings = [hello, bonjour]  // type: [(String) -> String]

// Pass as parameters
greetings[0]("World")  // "Hello, World!"
```

### 5. Expressive Parameter Labels

Use external parameter names to make call sites read like sentences. Use `_` to suppress labels when they add no clarity.

```swift
// Good - reads naturally at call site
func hello(_ people: String...) -> (count: Int, greeting: String) { ... }
func move(the card: Card, toTopOf deck: Deck) -> Deck { ... }

// Call sites read like English
hello("Thing One", "Thing Two")
move(the: queenOfHearts, toTopOf: freshDeck)
```

### 6. Named Tuple Returns

When a function returns multiple values, always name the tuple components:

```swift
// Preferred - named components
func hello(_ people: String...) -> (count: Int, greeting: String) {
    (people.count,
     people.reduce("") { $0 + "\nHello, " + $1 + "!" })
}

// Access by name, not .0 / .1
hello("Alice", "Bob").count    // 2
hello("Alice", "Bob").greeting // "\nHello, Alice!\nHello, Bob!"
```

### 7. Value Types Over Reference Types

Prefer structs and enums (value types) over classes (reference types).

```swift
// Structs - value semantics, memberwise init, Equatable synthesis
struct Card: Equatable {
    let rank: Rank
    let suit: Suit
}

// Enums - powerful with raw values, associated values, methods
enum Suit: String, CaseIterable {
    case spades = "♠︎"
    case diamonds = "♦︎"
    case clubs = "♣︎"
    case hearts = "♥︎"
}
```

Use classes only when you need:
- Reference semantics (identity, shared mutable state)
- Inheritance
- Interop with Objective-C / UIKit
- Observable objects in SwiftUI

### 8. Enumerations Are Powerful

Swift enums are far more than simple constants:

```swift
// Raw values
enum Rank: String, CaseIterable {
    case ace = "A", two = "2", three = "3"
    // ...
}

// Associated values
enum NetworkResult {
    case success(Data)
    case failure(Error)
}

// Methods and computed properties
extension Rank: CustomStringConvertible {
    var description: String { rawValue }
}

// Switch must be exhaustive
switch card.suit {
case .spades: // ...
case .diamonds: // ...
case .clubs: // ...
case .hearts: // ...
}
```

### 9. Structs Done Right

```swift
struct Model {
    let value: Int           // prefer let properties

    // Non-mutating computed properties
    var decrease: Model { Model(value: value - 1) }
    var increase: Model { Model(value: value + 1) }
}

// Conform to protocols via extensions
extension Model: CustomStringConvertible {
    var description: String { "Model(\(value))" }
}

extension Model: Equatable {}  // synthesized
```

- Separate stored properties from protocol conformances using extensions
- Use `willSet`/`didSet` for observation on `var` properties
- Prefer computed properties over methods when there are no parameters and no side effects

### 10. Safe Optionals

If a value can be `nil`, it must be `Optional`. Never force-unwrap unless you are certain.

**Prefer `guard let` for the happy path:**
```swift
func validate(_ url: URL?) -> String {
    guard let url = url else { return "No file found" }
    return "Found file at: \(url.path)"
}
```

**Use `if let` for conditional logic:**
```swift
if let url = url {
    print("Found: \(url.path)")
} else {
    print("Missing")
}
```

**Nil coalescing for defaults:**
```swift
let name = optionalName ?? "Unknown"
```

**Name shadowing** is idiomatic — use the same name for the unwrapped value:
```swift
guard let url = url else { return }
// `url` is now non-optional
```

### 11. Protocols and Extensions

Protocols define capabilities. Extensions add them. Protocol extensions provide shared defaults.

```swift
// Define a protocol
protocol Describable {
    var shortDescription: String { get }
}

// Conform via extension
extension Card: Describable {
    var shortDescription: String { "\(rank)\(suit)" }
}

// Protocol extension with default implementation
extension Describable {
    var shortDescription: String { "\(self)" }
}

// Generics constrained by protocol
func display<T: Describable>(_ item: T) {
    print(item.shortDescription)
}
```

**Gotcha:** If a method is defined in a protocol extension but NOT required by the protocol, dispatch is static (based on declared type, not runtime type). Only methods declared in the protocol itself get dynamic dispatch.

### 12. Error Handling

Use Swift's error handling rather than returning optionals or booleans for failures:

```swift
enum ValidationError: Error {
    case tooShort
    case invalidCharacter(Character)
}

func validate(_ input: String) throws -> String {
    guard input.count >= 3 else { throw ValidationError.tooShort }
    return input
}

// Catching
do {
    let result = try validate(input)
} catch ValidationError.tooShort {
    print("Too short")
} catch {
    print("Other error: \(error)")
}
```

Use `Result` for async contexts or when you need to store the outcome:
```swift
let result: Result<String, ValidationError> = .success("valid")
```

Use `defer` for cleanup that must happen regardless of how a scope exits.

### 13. Higher-Order Functions

Functions that accept or return other functions. Master these for expressive, concise code:

```swift
// Returning a function (partial application)
func line(slope m: Int, intercept b: Int) -> (Int) -> Int {
    { x in m * x + b }
}
let threeXPlusTwo = line(slope: 3, intercept: 2)
threeXPlusTwo(4)  // 14

// map - transform each element
names.map { $0.uppercased() }

// filter - keep elements matching condition
numbers.filter { $0.isMultiple(of: 2) }

// reduce - accumulate into single value
numbers.reduce(0, +)

// flatMap - transform and flatten
hands.flatMap { $0.cards }

// compactMap - transform, discard nils
strings.compactMap { Int($0) }
```

**Prefer point-free style** when passing named functions:
```swift
deck.map(increment)       // not deck.map { increment($0) }
deck.filter(isRed)        // not deck.filter { isRed($0) }
```

### 14. Collections

```swift
// Arrays - indexed, ordered, may duplicate
let primes = [2, 3, 5, 7]
var evens = [4, 10, 16]
evens.append(8)
evens += [12, 6]

// Dictionaries - key-value, unordered, unique keys
var scores: [String: Int] = ["Alice": 95, "Bob": 87]
scores["Charlie"] = 92

// Sets - unordered, unique elements
var tags: Set<String> = ["swift", "ios"]
tags.insert("swiftui")

// Strings are collections of Characters
for char in "Hello" { print(char) }
"Hello".count  // 5
```

- `Array<Element>` is formally correct but prefer `[Element]`
- `Dictionary<Key, Value>` prefer `[Key: Value]`
- `Optional<Wrapped>` prefer `Wrapped?`

### 15. Access Control

Think about access from the start, not as an afterthought:

- `private` - visible only within the enclosing declaration
- `fileprivate` - visible within the file
- `internal` (default) - visible within the module
- `public` - visible outside the module
- `open` - visible and subclassable outside the module

```swift
struct Counter {
    public private(set) var count = 0  // read public, write private

    mutating func increment() {
        count += 1
    }
}
```

## When Reviewing Swift Code

1. **`var` that should be `let`** - can this be immutable?
2. **Explicit types that could be inferred** - is the type annotation necessary?
3. **Force unwraps (`!`)** - use `guard let`, `if let`, or `??` instead
4. **Unnamed tuple components** - add names for clarity
5. **Class that should be struct** - does this need reference semantics?
6. **Missing `guard let`** - can nested `if let` be flattened with guard?
7. **Manual loops** - can `map`, `filter`, `reduce`, `flatMap`, or `compactMap` express this more clearly?
8. **Unclear parameter labels** - do call sites read naturally?
9. **Conformances in the type declaration** - separate into extensions
10. **Public mutable state** - should this be `public private(set) var`?

## Philosophy

- "I always start by labeling my variables as let values and change them to vars only when I'm sure they need to change."
- "The way we think about our code has changed. We're beginning to understand what we mean when we say 'That code is more Swifty.'"
- Functions can exist on their own without classes. A function is not a method until it lives inside a type.
- FP ideas in Swift are 100 years old mathematically and 60+ years old in programming (Lisp, 1958). They are not new or suspect.
