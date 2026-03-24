---
name: swift-codable
description: Comprehensive guide to Swift Codable - encoding/decoding JSON and property lists, custom CodingKeys, nested containers, date strategies, error handling, heterogeneous collections, Core Data integration, and custom encoder/decoder implementation
metadata:
  author: Henry Hudson
  version: "1.0"
  source: "Flight School Guide to Swift Codable by Matt"
---

# Swift Codable - Complete Reference

## 1. Fundamentals

### What is Codable?

`Codable` is a composite type consisting of two protocols:

```swift
typealias Codable = Decodable & Encodable
```

- **`Decodable`** requires: `init(from decoder: Decoder) throws`
- **`Encodable`** requires: `func encode(to encoder: Encoder) throws`

You can conform to just one if you only need one direction.

### Automatic Synthesis

Swift automatically synthesizes conformance for `Decodable` and `Encodable` when:

1. The type adopts the protocol **in its declaration** (not in an extension).
2. **Every stored property** conforms to `Codable`.

```swift
struct Plane: Codable {
    var manufacturer: String
    var model: String
    var seats: Int
}
// Compiler synthesizes init(from:) and encode(to:) automatically
```

### JSON Type Mapping

| JSON        | Swift Types                       |
|-------------|-----------------------------------|
| object      | Dictionary, struct/class          |
| array       | Array                             |
| true/false  | Bool                              |
| null        | Optional (nil)                    |
| string      | String                            |
| number      | Int, Double, Float, Decimal, etc. |

### Basic Decoding

```swift
import Foundation

let json = """
{
    "manufacturer": "Cessna",
    "model": "172 Skyhawk",
    "seats": 4
}
""".data(using: .utf8)!

let decoder = JSONDecoder()
let plane = try decoder.decode(Plane.self, from: json)
```

### Basic Encoding

```swift
let encoder = JSONEncoder()
let data = try encoder.encode(plane)
let jsonString = String(data: data, encoding: .utf8)!
```

**Tip:** Set `encoder.outputFormatting = .prettyPrinted` for human-readable output.

### Decoding Arrays

```swift
// Top-level array
let planes = try decoder.decode([Plane].self, from: json)

// Array nested under a key
let keyed = try decoder.decode([String: [Plane]].self, from: json)
let planes = keyed["planes"]

// Or create a wrapper type
struct Fleet: Decodable {
    var planes: [Plane]
}
```

**Conditional conformance:** `Array` conforms to `Codable` when its `Element` conforms. `Dictionary` conforms when both `Key` and `Value` conform.

---

## 2. CodingKeys and Key Strategies

### Custom CodingKeys Enum

When JSON keys differ from Swift property names, define a private `CodingKeys` enum:

```swift
struct FlightPlan: Decodable {
    var aircraft: Aircraft
    var flightRules: FlightRules
    var route: [String]
    private var departureDates: [String: Date]
    var remarks: String?

    private enum CodingKeys: String, CodingKey {
        case aircraft
        case flightRules = "flight_rules"
        case route
        case departureDates = "departure_time"
        case remarks
    }
}
```

Only specify explicit raw values for cases where the name mismatches. Cases with matching names need no raw value.

### Key Decoding Strategy (snake_case)

```swift
let decoder = JSONDecoder()
decoder.keyDecodingStrategy = .convertFromSnakeCase
```

**Caution:** This relies on consistent naming in JSON responses. Explicit `CodingKeys` is more robust and keeps the mapping visible in code.

### Dynamic / Unknown Keys with CodingKey Struct

When keys are not known at compile time, use a struct instead of an enum:

```swift
private struct CodingKeys: CodingKey {
    var stringValue: String
    var intValue: Int? { return nil }

    init?(stringValue: String) {
        self.stringValue = stringValue
    }

    init?(intValue: Int) {
        return nil
    }

    static let points = CodingKeys(stringValue: "points")!
}
```

Then use dynamic keys in `init(from:)`:

```swift
init(from decoder: Decoder) throws {
    let container = try decoder.container(keyedBy: CodingKeys.self)
    let codes = try container.decode([String].self, forKey: .points)
    var points: [Airport] = []
    for code in codes {
        let key = CodingKeys(stringValue: code)!
        let airport = try container.decode(Airport.self, forKey: key)
        points.append(airport)
    }
    self.points = points
}
```

---

## 3. Dates

### Date Decoding Strategies

JSON has no native date type. Use `dateDecodingStrategy` on `JSONDecoder`:

```swift
let decoder = JSONDecoder()

// ISO 8601 timestamps (e.g. "2018-04-20T14:15:00-07:00")
decoder.dateDecodingStrategy = .iso8601

// Unix timestamp (seconds since epoch)
decoder.dateDecodingStrategy = .secondsSince1970

// Milliseconds since epoch
decoder.dateDecodingStrategy = .millisecondsSince1970

// Custom DateFormatter
let formatter = DateFormatter()
formatter.dateFormat = "yyyy-MM-dd"
decoder.dateDecodingStrategy = .formatted(formatter)

// Fully custom
decoder.dateDecodingStrategy = .custom { decoder in
    let container = try decoder.singleValueContainer()
    let string = try container.decode(String.self)
    // parse string into Date
    return date
}
```

### Date Encoding Strategies

Symmetric strategies exist on `JSONEncoder`:

```swift
let encoder = JSONEncoder()
encoder.dateEncodingStrategy = .iso8601
```

### Property Lists Handle Dates Natively

`PropertyListDecoder` and `PropertyListEncoder` have native date support, so no date strategy is needed.

---

## 4. Null Values and Optionals

### Decoding null

Use `Optional` for properties that may be `null` or missing:

```swift
var remarks: String?  // Automatically handles null and missing keys
```

### decodeIfPresent vs decode

- `decode(_:forKey:)` -- throws if key is missing or value is null (for non-optional types)
- `decodeIfPresent(_:forKey:)` -- returns nil if key is missing or value is null

```swift
self.mealPreference = try container.decodeIfPresent(String.self, forKey: .mealPreference)
```

---

## 5. Nested Objects and Computed Properties

### Nested Codable Types

Define a separate type for nested JSON objects:

```swift
struct Aircraft: Decodable {
    var identification: String
    var color: String
}

struct FlightPlan: Decodable {
    var aircraft: Aircraft  // Decoded from nested JSON object
}
```

### Flattening Nested Keys with Computed Properties

When a nested JSON object is just organizational (not a distinct entity), use a dictionary and computed properties:

```swift
struct FlightPlan: Decodable {
    private var departureDates: [String: Date]

    var proposedDepartureDate: Date? {
        return departureDates["proposed"]
    }

    var actualDepartureDate: Date? {
        return departureDates["actual"]
    }
}
```

---

## 6. Enums

### Enum with Raw Value

Enums with a `String` or `Int` raw value automatically synthesize Codable conformance:

```swift
enum FlightRules: String, Decodable {
    case visual = "VFR"
    case instrument = "IFR"
}
```

If the JSON value is anything other than `"VFR"` or `"IFR"`, decoding fails.

### Enum for Filtering

```swift
enum Explicitness: String, Decodable {
    case explicit
    case cleaned
    case notExplicit
}

extension SearchResponse {
    var nonExplicitResults: [SearchResult] {
        return results.filter { $0.trackExplicitness != .explicit }
    }
}
```

---

## 7. Custom Encoding and Decoding

### When You Must Implement Manually

- Dynamic/unknown keys
- Heterogeneous collections (mixed types in an array)
- Inheritance (subclass properties not picked up by synthesis)
- Values that can be objects, arrays, or single values
- Core Data `NSManagedObject` subclasses
- Multiple representation formats for the same concept

### Manual init(from:) Pattern

```swift
init(from decoder: Decoder) throws {
    let container = try decoder.container(keyedBy: CodingKeys.self)
    self.manufacturer = try container.decode(String.self, forKey: .manufacturer)
    self.model = try container.decode(String.self, forKey: .model)
    self.seats = try container.decode(Int.self, forKey: .seats)
}
```

### Manual encode(to:) Pattern

```swift
func encode(to encoder: Encoder) throws {
    var container = encoder.container(keyedBy: CodingKeys.self)
    try container.encode(self.manufacturer, forKey: .manufacturer)
    try container.encode(self.model, forKey: .model)
    try container.encode(self.seats, forKey: .seats)
}
```

**Note:** In `encode(to:)`, the container must be `var` (not `let`) because encoding mutates it.

---

## 8. Heterogeneous / Indeterminate Types

### Either Type for Two Possibilities

```swift
enum Either<T, U> {
    case left(T)
    case right(U)
}

extension Either: Decodable where T: Decodable, U: Decodable {
    init(from decoder: Decoder) throws {
        if let value = try? T(from: decoder) {
            self = .left(value)
        } else if let value = try? U(from: decoder) {
            self = .right(value)
        } else {
            let context = DecodingError.Context(
                codingPath: decoder.codingPath,
                debugDescription: "Cannot decode \(T.self) or \(U.self)"
            )
            throw DecodingError.dataCorrupted(context)
        }
    }
}

// Usage
let objects = try decoder.decode([Either<Bird, Plane>].self, from: json)
```

### AnyDecodable for Arbitrary Types

For `[String: Any]` style metadata, use a type-erased wrapper like `AnyDecodable` (see [flight-school/AnyCodable](https://github.com/flight-school/AnyCodable)):

```swift
struct Report: Decodable {
    var title: String
    var body: String
    var metadata: [String: AnyDecodable]
}
```

---

## 9. Decoding from Multiple Value Representations

When a value can be an object, array, or single value, try containers in order:

```swift
init(from decoder: Decoder) throws {
    // Try keyed container first (JSON object)
    if let container = try? decoder.container(keyedBy: CodingKeys.self) {
        self.latitude = try container.decode(Double.self, forKey: .latitude)
        self.longitude = try container.decode(Double.self, forKey: .longitude)
        self.elevation = try container.decodeIfPresent(Double.self, forKey: .elevation)
    }
    // Try unkeyed container (JSON array)
    else if var container = try? decoder.unkeyedContainer() {
        self.longitude = try container.decode(Double.self)
        self.latitude = try container.decode(Double.self)
        self.elevation = try container.decodeIfPresent(Double.self)
    }
    // Try single value container (string like "37.332, -122.011")
    else if let container = try? decoder.singleValueContainer() {
        let string = try container.decode(String.self)
        // Parse string into components...
    }
    else {
        let context = DecodingError.Context(
            codingPath: decoder.codingPath,
            debugDescription: "Unable to decode Coordinate"
        )
        throw DecodingError.dataCorrupted(context)
    }
}
```

---

## 10. Multiple Data Source Representations

When consuming the same concept from different APIs with different JSON structures:

1. **Create a Decodable type for each representation format.**
2. **Define a canonical type** (struct or protocol) that reconciles differences.
3. **Provide initializers or protocol conformance** to convert between them.

```swift
// Canonical type as protocol
protocol FuelPrice {
    var type: Fuel { get }
    var pricePerLiter: Decimal { get }
    var currency: String { get }
}

// Conform each API-specific Decodable type
extension AmericanFuelPrice: FuelPrice {
    var type: Fuel { return self.fuel }
    var pricePerLiter: Decimal { return self.price / 3.78541 }
    var currency: String { return "USD" }
}
```

**Tip:** Use `Decimal` (not `Double`) for monetary values to avoid precision loss.

---

## 11. Class Inheritance

### The Problem

Subclasses inherit the synthesized `init(from:)` from the superclass. The superclass implementation does not know about subclass properties, so they get default/nil values.

### The Fix

Override `init(from:)` in the subclass, decode subclass properties, then call `super.init(from:)`:

```swift
class EconomySeat: Decodable {
    var number: Int
    var letter: String
}

class PremiumEconomySeat: EconomySeat {
    var mealPreference: String?

    private enum CodingKeys: String, CodingKey {
        case mealPreference
    }

    required init(from decoder: Decoder) throws {
        let container = try decoder.container(keyedBy: CodingKeys.self)
        self.mealPreference = try container.decodeIfPresent(
            String.self, forKey: .mealPreference
        )
        try super.init(from: decoder)
    }
}
```

**Note:** A `final` class can implement `init(from:)` as a convenience initializer in an extension.

---

## 12. userInfo for Configuration

### Passing Options via CodingUserInfoKey

Define a custom key:

```swift
extension CodingUserInfoKey {
    static let colorEncodingStrategy =
        CodingUserInfoKey(rawValue: "colorEncodingStrategy")!
}
```

Set it on the encoder/decoder:

```swift
let encoder = JSONEncoder()
encoder.userInfo[.colorEncodingStrategy] =
    ColorEncodingStrategy.hexadecimal(hash: true)
```

Read it in `encode(to:)` or `init(from:)`:

```swift
func encode(to encoder: Encoder) throws {
    var container = encoder.singleValueContainer()
    switch encoder.userInfo[.colorEncodingStrategy] as? ColorEncodingStrategy {
    case let .hexadecimal(hash)?:
        try container.encode(
            (hash ? "#" : "") +
            String(format: "%02X%02X%02X", red, green, blue)
        )
    default:
        try container.encode(
            String(format: "rgb(%d, %d, %d)", red, green, blue)
        )
    }
}
```

---

## 13. Property Lists and UserDefaults

### PropertyListEncoder / PropertyListDecoder

Codable works identically with property lists. Property lists natively support dates, binary data, and distinguish integer from floating-point numbers (unlike JSON).

```swift
// Encoding
let encoder = PropertyListEncoder()
let data = try encoder.encode(orders)

// Decoding
let decoder = PropertyListDecoder()
let orders = try decoder.decode([Order].self, from: data)
```

### Persisting with UserDefaults

```swift
// Save
let encoder = PropertyListEncoder()
UserDefaults.standard.set(try encoder.encode(orders), forKey: "orders")

// Load
if let data = UserDefaults.standard.object(forKey: "orders") as? Data {
    let decoder = PropertyListDecoder()
    let orders = try decoder.decode([Order].self, from: data)
}

// Delete
UserDefaults.standard.removeObject(forKey: "orders")

// Register defaults (call at app launch)
UserDefaults.standard.register(defaults: ["orders": []])
```

**UserDefaults is best for:** Small data, read frequently, written occasionally. For large object graphs, consider Core Data.

---

## 14. Core Data Integration

### The Challenge

`NSManagedObject` requires `init(entity:insertInto:)` while `Decodable` requires `init(from:)`. They are incompatible designated initializers.

### The Solution: Pass Context via userInfo

```swift
extension CodingUserInfoKey {
    static let context = CodingUserInfoKey(rawValue: "context")!
}

// Set up decoder
let decoder = JSONDecoder()
decoder.userInfo[.context] = managedObjectContext

// In NSManagedObject subclass
extension Luggage: Decodable {
    private enum CodingKeys: String, CodingKey {
        case identifier = "id"
        case weight
    }

    convenience init(from decoder: Decoder) throws {
        guard let context = decoder.userInfo[.context]
            as? NSManagedObjectContext else {
            fatalError("Missing context")
        }

        guard let entity = NSEntityDescription.entity(
            forEntityName: "Luggage", in: context) else {
            fatalError("Unknown entity")
        }

        self.init(entity: entity, insertInto: context)

        let container = try decoder.container(keyedBy: CodingKeys.self)
        self.identifier = try container.decode(UUID.self, forKey: .identifier)
        self.weight = try container.decode(Float.self, forKey: .weight)
    }
}
```

**Note:** Codable conformance cannot be synthesized for `NSManagedObject` subclasses.

### Unkeyed Containers for Arrays

When JSON represents a value as an array (e.g., `["D", "ZMU"]` for a name), use an unkeyed container:

```swift
var container = try decoder.unkeyedContainer()
self.givenName = try container.decode(String.self)
self.familyName = try container.decode(String.self)
```

---

## 15. Containers Deep Dive

### Three Container Types

| Container Type        | Use Case                         | JSON Equivalent |
|-----------------------|----------------------------------|-----------------|
| **KeyedContainer**    | Key-value pairs (object/dict)    | `{}`            |
| **UnkeyedContainer**  | Ordered collection (array)       | `[]`            |
| **SingleValueContainer** | Single primitive value        | `"value"` / `42` |

### Encoder Protocol

```swift
protocol Encoder {
    var codingPath: [CodingKey] { get }
    var userInfo: [CodingUserInfoKey: Any] { get }

    func container<Key>(keyedBy type: Key.Type)
        -> KeyedEncodingContainer<Key> where Key: CodingKey
    func unkeyedContainer() -> UnkeyedEncodingContainer
    func singleValueContainer() -> SingleValueEncodingContainer
}
```

### Decoder Protocol

```swift
protocol Decoder {
    var codingPath: [CodingKey] { get }
    var userInfo: [CodingUserInfoKey: Any] { get }

    func container<Key>(keyedBy type: Key.Type) throws
        -> KeyedDecodingContainer<Key> where Key: CodingKey
    func unkeyedContainer() throws -> UnkeyedDecodingContainer
    func singleValueContainer() throws -> SingleValueDecodingContainer
}
```

### Custom Encoder/Decoder Implementation Structure

A custom encoder (e.g., for MessagePack) typically has:

```
MessagePackEncoder (public API)
  -> _MessagePackEncoder: Encoder (internal, conforms to Encoder protocol)
       -> SingleValueContainer: SingleValueEncodingContainer
       -> UnkeyedContainer: UnkeyedEncodingContainer
       -> KeyedContainer<Key>: KeyedEncodingContainerProtocol
```

Each container is responsible for encoding values into the target format's binary/text representation.

---

## 16. Error Handling

### Error Handling Patterns

```swift
// Full error handling (production code)
do {
    let plane = try decoder.decode(Plane.self, from: json)
} catch {
    print("Error decoding: \(error)")
}

// Optional result (when error details are unneeded)
if let plane = try? decoder.decode(Plane.self, from: json) {
    print(plane.model)
}

// Force (playground/testing only -- crashes on error)
let plane = try! decoder.decode(Plane.self, from: json)
```

### DecodingError Types

- `DecodingError.dataCorrupted(Context)` -- data is corrupted or invalid
- `DecodingError.keyNotFound(CodingKey, Context)` -- expected key is missing
- `DecodingError.typeMismatch(Any.Type, Context)` -- value type does not match
- `DecodingError.valueNotFound(Any.Type, Context)` -- non-optional value is null

### Throwing Custom Errors

```swift
let context = DecodingError.Context(
    codingPath: decoder.codingPath,
    debugDescription: "Unable to decode Coordinate"
)
throw DecodingError.dataCorrupted(context)

// From within a container
throw DecodingError.dataCorruptedError(
    in: container,
    debugDescription: "Invalid coordinate string"
)
```

### EncodingError Types

- `EncodingError.invalidValue(Any, Context)` -- value cannot be encoded

---

## When Reviewing Code Checklist

- [ ] **Prefer synthesized conformance.** Only implement `init(from:)` and `encode(to:)` manually when the compiler cannot synthesize it.
- [ ] **Adopt Codable in the type declaration**, not an extension, for synthesis to work.
- [ ] **All stored properties must be Codable** for synthesis. If one is not, either make it Codable or implement manually.
- [ ] **Use `Codable` when both encoding and decoding are needed**, `Decodable` or `Encodable` when only one direction is required.
- [ ] **Use explicit `CodingKeys`** instead of `keyDecodingStrategy` for robustness and clarity.
- [ ] **Use `decodeIfPresent`** for optional values; use `decode` for required values.
- [ ] **Handle errors properly.** Never use `try!` in production code. Use `do/catch` or `try?`.
- [ ] **Set `dateDecodingStrategy`** when working with JSON dates. Property lists handle dates natively.
- [ ] **Use `Decimal` for money**, not `Double` or `Float`.
- [ ] **Private `CodingKeys`** to avoid leaking internal representation details.
- [ ] **Subclasses must override `init(from:)`** and call `super.init(from: decoder)` to decode inherited properties.
- [ ] **Use `userInfo`** to pass context (e.g., `NSManagedObjectContext`) or configuration to the encoder/decoder.
- [ ] **Avoid `[String: Any]` in Codable types.** Use `AnyDecodable` or strongly-typed alternatives.
- [ ] **Hide inconvenient decoded structures** behind computed properties for a clean API.
- [ ] **Create separate Decodable types** for each external representation, then reconcile them in a canonical type.
- [ ] **Core Data:** Pass the managed object context via `decoder.userInfo`, use a convenience initializer, and call the designated `NSManagedObject` initializer inside it.
- [ ] **Test round-trips.** Encode then decode (or vice versa) to verify nothing is lost in translation.
