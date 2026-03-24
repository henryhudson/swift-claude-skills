---
name: swift-numbers
description: Swift numeric types, floating-point precision, integer overflow, Decimal for money, NSNumber bridging, NumberFormatter locale-aware formatting, Foundation Measurement/Units for dimensional quantities
metadata:
  author: Henry Hudson
  version: "1.0"
  source: "Flight School Guide to Swift Numbers by Matt"
---

# Swift Numbers

## Integer Types

### Signed Integers
- `Int8`, `Int16`, `Int32`, `Int64` are fixed-width signed types.
- `Int` is platform-dependent: same as `Int64` on 64-bit, `Int32` on 32-bit.
- **Default choice**: Use `Int` for whole numbers unless you need an explicit size.

### Unsigned Integers
- `UInt8`, `UInt16`, `UInt32`, `UInt64` are fixed-width unsigned types.
- `UInt` is platform-dependent, same pattern as `Int`.
- Use unsigned types primarily for bit patterns and low-level API interop.
- **Prefer `Int` even for nonnegative values** to avoid unnecessary type conversions.

### Ranges

```swift
UInt8.min  // 0
UInt8.max  // 255
Int8.min   // -128
Int8.max   // 127
```

A fixed-width integer with *n* bits can represent 2^n distinct values. `UInt8` covers 0...255, `Int8` covers -128...127.

### Integer Overflow

By default, Swift traps on overflow. Use overflow operators when you explicitly want wrap-around behavior:

```swift
let u = UInt8.max    // 255
// u + 1            // runtime error!
u &+ 1              // 0 (wraps around)
u &- 1              // 254
```

Reporting overflow without trapping:

```swift
let (result, overflow) = UInt8.max.addingReportingOverflow(1)
// result == 0, overflow == true
```

Unsafe variants (`unsafeAdding(_:)`, `unsafeSubtracting(_:)`, `unsafeMultiplied(by:)`, `unsafeDivided(by:)`) skip overflow checks entirely. The compiler flag `-Ounchecked` disables all overflow checking.

### Integer Conversions

Four strategies for converting between integer types:

```swift
// 1. Range-checked (traps on failure)
let i16 = Int16(747)     // OK
// let i8 = Int8(747)    // runtime error

// 2. Exact (returns nil on failure)
UInt(exactly: 57)        // Optional(57)
UInt(exactly: -33)       // nil
Int8(exactly: 123.0)     // Optional(123)
Int8(exactly: 123.456)   // nil (fractional component)

// 3. Clamping (saturates at min/max)
Int8(clamping: 0xA320)   // 127
UInt8(clamping: -747)    // 0

// 4. Bit pattern (truncates/extends bits)
let u: UInt8 = 0b11110000               // 240
Int8(truncatingIfNeeded: u)              // -16
UInt16(truncatingIfNeeded: u)            // 240
```

### Two's Complement

Swift uses two's complement for signed integers. The most significant bit is the sign bit (0 = positive, 1 = negative). Arithmetic works identically for signed and unsigned at the bit level.

### Bitwise Operations

```swift
let a: UInt8 = 0b11001010
let b: UInt8 = 0b10110101

a & b   // AND:  0b10000000
a | b   // OR:   0b11111111
a ^ b   // XOR:  0b01111111
~a      // NOT:  0b00110101
a << 2  // Left shift
a >> 2  // Right shift
```

### Byte Order

- **Big-endian**: most significant byte first.
- **Little-endian**: least significant byte first.
- Use `.bigEndian` / `.littleEndian` properties when interfacing with systems using a specific byte order.
- High-level frameworks like Foundation handle byte order for you.

### Integer Literal Bases

```swift
0b11100001  // Binary (base 2)
0o341       // Octal (base 8)
225         // Decimal (base 10)
0xe1        // Hexadecimal (base 16)
```

Use underscores to improve readability of long literals:

```swift
let bitPattern: UInt16 = 0b00110011_10101010
let million = 1_000_000
```

## Floating-Point Types

### Built-In Types

| Type | Alias | Significand Bits | Decimal Digits |
|------|-------|-----------------|----------------|
| `Float` | `Float32` | 23 | ~7 |
| `Double` | `Float64` | 52 | ~16 |
| `Float80` | - | 63 | ~19 |

- **Default choice**: Use `Double` unless you have a specific reason for another type.
- Integer literals infer `Int`; floating-point literals infer `Double`.

### IEEE 754 Representation

Every floating-point number has four components:
- **Sign**: positive (+) or negative (-)
- **Radix** (base): 2 for binary floating-point
- **Biased exponent**: exponent + bias (bias = 127 for Float, 1023 for Double)
- **Significand**: the significant digits

A normalized float has a significand in range [1, 2). The leading 1 is implicit and not stored.

```swift
Float.exponentBitCount      // 8
Float.significandBitCount   // 23
Double.exponentBitCount     // 11
Double.significandBitCount  // 52
```

### The 0.1 + 0.2 Problem

Binary floating-point cannot exactly represent most base-10 fractions. Fractions terminate in binary only if the denominator's sole prime factor is 2.

```swift
0.1 + 0.2 == 0.3  // false!
// 0.1 is actually 0.10000000000000001
// 0.2 is actually 0.20000000000000001
// sum is          0.30000000000000004
```

### Comparing Floating-Point Numbers

Never use `==` for floating-point comparisons where rounding may occur. Use ULP-based comparison:

```swift
// ULP = Unit of Least Precision
Float.ulpOfOne                    // 0.00000011920929
Float.greatestFiniteMagnitude.ulp // 2.02824096E+31

// Approximate equality operator
infix operator ==~ : ComparisonPrecedence
func ==~ <T>(lhs: T, rhs: T) -> Bool where T: FloatingPoint {
    return lhs == rhs || lhs.nextDown == rhs || lhs.nextUp == rhs
}

0.1 + 0.2 ==~ 0.3  // true
```

More robust version with margin and ULP tolerance:

```swift
extension FloatingPoint {
    func isApproximatelyEqual(to other: Self,
                              within margin: Self? = nil,
                              maximumULPs ulps: Int = 1) -> Bool {
        precondition(margin?.sign != .minus && ulps > 0)
        guard self != other else { return true }
        let difference = abs(self - other)
        if let margin = margin, difference > margin {
            return false
        }
        return difference < (self.ulp * Self(ulps))
    }
}
```

### Special Values

```swift
// Positive and Negative Zero
-0.0 == 0.0       // true
0.0.isZero         // true
-0.0.isZero        // true

// Infinity
Double.infinity.isFinite    // false
(-Double.infinity).floatingPointClass == .negativeInfinity

// NaN (Not a Number)
Double.nan == Double.nan    // false (NaN != NaN!)
Double.nan.isNaN            // true

// Detecting floating-point exceptions
import Darwin
feclearexcept(FE_INVALID)
Double.signalingNaN + 1
fetestexcept(FE_INVALID) == 0  // false -> invalid operation occurred
```

Available exception flags: `FE_OVERFLOW`, `FE_UNDERFLOW`, `FE_INEXACT`, `FE_DIVBYZERO`, `FE_INVALID`.

## Decimal Type (Foundation)

`Decimal` (bridged to `NSDecimalNumber`) precisely represents base-10 numbers. Unlike `Float`/`Double`, it does not suffer from binary representation errors for decimal fractions.

```swift
import Foundation

let a: Decimal = 0.1
let b: Decimal = 0.2
a + b == 0.3  // true (!)
```

### When to Use Decimal
- **Money and financial calculations**: Always use `Decimal`, never `Float` or `Double`.
- **Any domain where base-10 exactness matters**: tax rates, percentages displayed to users.

### Decimal vs. Integer for Money
- Integers (cents) work but cause problems with fractional multiplication (tax calculations) due to intermediate rounding.
- `Decimal` avoids this while also being more natural to use than cent-based integers.

## Currency and Money

### Type-Safe Money Pattern (Protocol-Oriented)

Define currencies as types using a protocol for compile-time safety:

```swift
protocol CurrencyType {
    static var code: String { get }
    static var symbol: String { get }
    static var name: String { get }
}

// Uninhabitable enum -- cannot be instantiated
enum USD: CurrencyType {
    static var code: String { return "USD" }
    static var symbol: String { return "$" }
    static var name: String { return "US Dollar" }
}

struct Money<C: CurrencyType>: Equatable {
    typealias Currency = C
    var amount: Decimal

    init(_ amount: Decimal) {
        self.amount = amount
    }
}

// Type system prevents mixing currencies at compile time:
let usd = Money<USD>(14.00)
let cny = Money<CNY>(29.9)
// usd + cny  // Compiler error!
// usd == cny // Compiler error!
```

### Making Money Expressible from Literals

```swift
extension Money: ExpressibleByIntegerLiteral {
    public init(integerLiteral value: Int) {
        self.init(Decimal(integerLiteral: value))
    }
}

extension Money: ExpressibleByFloatLiteral {
    public init(floatLiteral value: Double) {
        self.init(Decimal(floatLiteral: value))
    }
}
```

### Comparable Conformance

```swift
extension Money: Comparable {
    static func < (lhs: Money, rhs: Money) -> Bool {
        return lhs.amount < rhs.amount
    }
}
```

## Number Formatting (NumberFormatter)

### Basic Usage

```swift
let formatter = NumberFormatter()
formatter.numberStyle = .ordinal
formatter.string(for: 2)  // "2nd"
```

### Reuse Formatters -- Creating Them Is Expensive

```swift
class ViewController: UIViewController {
    lazy var numberFormatter: NumberFormatter = {
        let formatter = NumberFormatter()
        formatter.numberStyle = .decimal
        return formatter
    }()
}
```

Prefer `string(for:)` over `string(from:)` to avoid explicit `as NSNumber` casts.

### Number Styles

| Style | Example (en-US) | Notes |
|-------|-----------------|-------|
| `.none` | `4` | Cardinal numeral |
| `.spellOut` | `four` | Cardinal words |
| `.ordinal` | `1st` | Rank/position |
| `.decimal` | `1,234,567.89` | Locale-aware separators |
| `.scientific` | `1.23456789E4` | Scientific notation |
| `.percent` | `12%` | Input 0.12 becomes "12%" |
| `.currency` | `$1,234.57` | Uses locale currency symbol |
| `.currencyAccounting` | `($1,234.57)` | Negative in parentheses |
| `.currencyISOCode` | `USD1,234.57` | Three-letter ISO code |
| `.currencyPlural` | `1,234.57 US dollars` | Full currency name |

### Configuring Precision

**Significant digits**:

```swift
formatter.usesSignificantDigits = true
formatter.maximumSignificantDigits = 2
formatter.string(from: 123.456)  // "120"
formatter.string(from: 0.00123)  // "0.0012"
```

**Fixed-width digits**:

```swift
formatter.usesSignificantDigits = false
formatter.minimumIntegerDigits = 4
formatter.minimumFractionDigits = 2
formatter.string(from: 123)  // "0123.00"
```

If minimum > maximum for any digit property, the maximum takes precedence.

### Rounding Modes

| Mode | 1.25 | -1.25 |
|------|------|-------|
| `.ceiling` | 1.3 | -1.2 |
| `.floor` | 1.2 | -1.3 |
| `.up` | 1.3 | -1.3 |
| `.down` | 1.2 | -1.2 |
| `.halfUp` | 1.3 | -1.3 |
| `.halfDown` | 1.2 | -1.2 |
| `.halfEven` | 1.2 | -1.2 |

### Locale-Aware Separators

```swift
formatter.locale = Locale(identifier: "en-GB")
formatter.string(for: 1234567.89)  // "1,234,567.89"

formatter.locale = Locale(identifier: "fr-FR")
formatter.string(for: 1234567.89)  // "1 234 567,89"

formatter.locale = Locale(identifier: "hi-IN")
formatter.string(for: 1234567.89)  // "12,34,567.89" (groups of 2 after thousands)
```

### Non-Hindu-Arabic Numerals

```swift
formatter.locale = Locale(identifier: "ar-SA")
formatter.string(for: 1234567890)  // Arabic-Indic numerals
```

Do not assume user input will always be ASCII 0-9.

### Custom Format Patterns

```swift
formatter.format = "#,##0.5"
formatter.string(for: 1234.567)  // "1,234.5" (rounds to nearest 0.5)
```

Pattern symbols: `0` = digit, `#` = digit (zero absent), `.` = decimal separator, `,` = grouping separator, `@` = significant digit.

### Currency Formatting Pitfall

Setting `numberStyle` to `.currency` without explicitly setting `currencySymbol` and/or `currencyCode` is **wrong**. It uses the locale's currency, fundamentally changing the value's meaning:

```swift
let price: Decimal = 300.00  // USD

formatter.numberStyle = .currency
formatter.locale = Locale(identifier: "ja-JP")
formatter.string(for: price)  // "Y300" -- this is NOT $300!
```

**Always set `currencyCode` explicitly** when formatting monetary amounts.

## Units and Measurements (Foundation)

### Core Types

- `Unit`: abstract base class for all units.
- `Dimension`: abstract subclass of `Unit` for dimensional quantities (has a converter).
- `Measurement<UnitType>`: generic struct storing a value for a particular unit type.

Each dimension type has a **base unit** from which all other units derive via `UnitConverterLinear`.

### Built-In Unit Types

| Unit Class | Base Unit |
|-----------|-----------|
| `UnitLength` | meters (m) |
| `UnitMass` | kilograms (kg) |
| `UnitSpeed` | meters per second (m/s) |
| `UnitTemperature` | kelvin (K) |
| `UnitVolume` | liters (L) |
| `UnitArea` | square meters (m^2) |
| `UnitDuration` | seconds (s) |
| `UnitAngle` | degrees |
| `UnitPressure` | N/m^2 |
| `UnitEnergy` | joules (J) |
| `UnitFrequency` | hertz (Hz) |
| `UnitFuelEfficiency` | L/100km |
| `UnitAcceleration` | m/s^2 |
| `UnitElectricCurrent` | amperes (A) |
| `UnitPower` | watts (W) |

### Creating and Converting Measurements

```swift
let airspeed = Measurement<UnitSpeed>(value: 123, unit: .knots)
airspeed.converted(to: .kilometersPerHour)  // 227.80 km/h
```

### Adding Measurements with Different Units

Same dimension, different units -- automatically converted:

```swift
let planeWeight = Measurement<UnitMass>(value: 660, unit: .kilograms)
let pilotWeight = Measurement<UnitMass>(value: 11, unit: .stones)
let fuelWeight  = Measurement<UnitMass>(value: 300, unit: .pounds)

let total = (planeWeight + pilotWeight + fuelWeight)
    .converted(to: .metricTons)  // 0.866 t
```

Different dimensions produce a compile-time error:

```swift
let length = Measurement<UnitLength>(value: 10, unit: .millimeters)
let volume = Measurement<UnitVolume>(value: 10, unit: .milliliters)
// length + volume  // Compiler error!
```

### Defining Custom Units

```swift
extension UnitSpeed {
    class var feetPerMinute: UnitSpeed {
        let converter = UnitConverterLinear(coefficient: 0.00508)
        return .init(symbol: "ft/min", converter: converter)
    }
}
```

### Subclassing Dimension for New Unit Types

```swift
class UnitForce: Dimension {
    class var newtons: UnitForce {
        return .init(symbol: "N",
                     converter: UnitConverterLinear(coefficient: 1))
    }
    class var kiloNewtons: UnitForce {
        return .init(symbol: "kN",
                     converter: UnitConverterLinear(coefficient: 1000))
    }
    class var pounds: UnitForce {
        return .init(symbol: "lbf",
                     converter: UnitConverterLinear(coefficient: 4.44822))
    }
    override class func baseUnit() -> UnitForce {
        return .newtons
    }
}

let thrust = Measurement<UnitForce>(value: 64_500, unit: .pounds)
thrust.converted(to: .kiloNewtons)  // 286.91 kN
```

### Compound Unit Operators

Foundation does not natively support compound unit composition, but you can define operators:

```swift
// Length * Length = Area
func *(lhs: Measurement<UnitLength>,
       rhs: Measurement<UnitLength>) -> Measurement<UnitArea> {
    return .init(value: lhs.converted(to: .meters).value *
                        rhs.converted(to: .meters).value,
                 unit: .squareMeters)
}

// Length / Speed = Duration (unit cancellation)
func /(lhs: Measurement<UnitLength>,
       rhs: Measurement<UnitSpeed>) -> Measurement<UnitDuration> {
    // implementation
}

// Same-dimension division yields scalar
extension Measurement {
    static func / <T>(lhs: Measurement<T>,
                      rhs: Measurement<T>) -> Double
        where T: Dimension {
        return lhs.value / rhs.converted(to: lhs.unit).value
    }
}
```

### Generic Rate Type for Compound Conversions

```swift
struct Rate<Numerator, Denominator>
    where Numerator: Unit, Denominator: Unit
{
    var value: Double
    var numeratorUnit: Numerator
    var denominatorUnit: Denominator
}

// Rate * Measurement = Measurement (denominator cancels)
func * <T, U>(lhs: Rate<T, U>,
              rhs: Measurement<U>) -> Measurement<T>
    where T: Dimension, U: Dimension
{
    let converted = Measurement(value: lhs.value, unit: rhs.unit)
        .converted(to: lhs.denominatorUnit)
    return .init(value: converted.value * rhs.value,
                 unit: lhs.numeratorUnit)
}

// Usage: fuel consumption
let fuelRate = Rate<UnitVolume, UnitDuration>(
    value: 50, numeratorUnit: .gallons, denominatorUnit: .hours)
let fuelNeeded = fuelRate * flightDuration  // gallons
```

### MeasurementFormatter

```swift
let formatter = MeasurementFormatter()
formatter.string(from: airspeed)  // Localized, e.g. "141.546 mph"
```

**Natural scale** -- picks the best unit for the locale and magnitude:

```swift
formatter.unitOptions = .naturalScale
formatter.string(from: roomLength)       // "26.247 ft"
formatter.string(from: distanceToCity)   // "9.942 mi"
```

**Provided unit** -- keep the original unit:

```swift
formatter.unitOptions = .providedUnit
formatter.string(from: ingotMass)  // "400 oz t"
```

**Precision** via the underlying NumberFormatter:

```swift
formatter.numberFormatter.usesSignificantDigits = true
formatter.numberFormatter.maximumSignificantDigits = 3
formatter.string(from: pressureInMillibars)  // "1,010 mbar"
```

**Temperature pitfall**: `.temperatureWithoutUnit` implicitly sets `.providedUnit`, so temperatures are NOT converted to the locale's preferred scale:

```swift
formatter.unitOptions = .temperatureWithoutUnit
// Shows the original unit's value, NOT the locale-preferred scale!
// Always convert explicitly if using this option.
```

### Bridging with Foundation Date/Time APIs

```swift
// TimeInterval -> Measurement
extension Measurement where UnitType == UnitDuration {
    init(timeInterval: TimeInterval) {
        self.init(value: timeInterval, unit: .seconds)
    }
}

// DateInterval -> Measurement
extension Measurement where UnitType == UnitDuration {
    init(_ dateInterval: DateInterval) {
        self.init(timeInterval: dateInterval.duration)
    }
}
```

### Bridging with Core Location

```swift
extension Measurement where UnitType == UnitSpeed {
    init(clLocationSpeed: CLLocationSpeed) {
        self.init(value: clLocationSpeed, unit: .metersPerSecond)
    }
}
```

## Swift Number Protocols Hierarchy

From most general to most specific:

- `Equatable` / `Comparable`
- `ExpressibleByIntegerLiteral` / `ExpressibleByFloatLiteral`
- `AdditiveArithmetic` -- supports `+`, `-`
- `Numeric` -- adds `*`
- `SignedNumeric` -- adds negation
- `BinaryInteger` -- integer bit operations, conversion strategies
  - `FixedWidthInteger` -- fixed bit count, overflow ops, bitwise ops
    - `SignedInteger` / `UnsignedInteger`
- `FloatingPoint` -- IEEE 754 semantics, special values
  - `BinaryFloatingPoint` -- binary radix, sign/exponent/significand access

Any type can opt into `ExpressibleByIntegerLiteral` or `ExpressibleByFloatLiteral` for literal initialization.

## Common Pitfalls and How to Avoid Them

### 1. Floating-Point Equality
**Problem**: `0.1 + 0.2 != 0.3` due to binary representation.
**Solution**: Use ULP-based comparison or `Decimal` when exact base-10 arithmetic is needed.

### 2. Money as Float/Double
**Problem**: Accumulated rounding errors in financial calculations.
**Solution**: Always use `Decimal` for monetary amounts. Pair with a `Currency` type for type safety.

### 3. Integer Overflow
**Problem**: Arithmetic on `Int8`, `UInt8`, etc. can silently overflow in `-Ounchecked` mode or crash in normal mode.
**Solution**: Use `addingReportingOverflow` when overflow is possible. Use overflow operators `&+`, `&-`, `&*` only when wrap-around is intentional.

### 4. Currency as Locale Preference
**Problem**: `NumberFormatter` with `.currency` style uses the device locale's currency, making `$300` display as `Y300` for Japanese locale users.
**Solution**: Always set `currencyCode` explicitly on the formatter.

### 5. Temperature Without Unit
**Problem**: `MeasurementFormatter` with `.temperatureWithoutUnit` implicitly sets `.providedUnit`, displaying the stored unit's value instead of converting to the user's preferred scale.
**Solution**: Convert to the locale's preferred temperature scale before formatting, or avoid `.temperatureWithoutUnit`.

### 6. Non-ASCII Digit Input
**Problem**: Assuming user-entered numbers use ASCII digits 0-9. Some locales use Eastern Arabic, Devanagari, or other numeral systems.
**Solution**: Use `NumberFormatter` for parsing user input, not manual string-to-int conversion.

### 7. Ignoring Locale Separators
**Problem**: Hardcoding `.` as decimal separator or `,` as thousands separator.
**Solution**: Always use `NumberFormatter` for display. Many locales use `,` as decimal point and `.` or space as group separator.

### 8. Creating Formatters in Loops
**Problem**: `NumberFormatter` and `MeasurementFormatter` are expensive to create.
**Solution**: Create once and reuse (e.g., as a lazy property on a view controller).

## When Reviewing Code Checklist

- [ ] **Money uses `Decimal`**, not `Float` or `Double`
- [ ] **Currency is explicit**, not inferred from locale (check that `currencyCode` is set on any `NumberFormatter` with currency style)
- [ ] **No bare `==` on floating-point** values that result from arithmetic (use ULP-based or epsilon comparison)
- [ ] **Integer overflow is handled**: either the domain guarantees no overflow, or overflow operators / reporting methods are used
- [ ] **NumberFormatter / MeasurementFormatter instances are reused**, not created in tight loops or table view cell configuration
- [ ] **Locale-sensitive formatting** uses `NumberFormatter`, not string interpolation or `String(format:)`
- [ ] **Measurement types are used** for physical quantities instead of raw `Double` with ad-hoc unit comments
- [ ] **Unit conversions use `converted(to:)`** rather than manual multiplication by magic constants
- [ ] **Custom units are exposed as class properties** on the appropriate `Dimension` subclass, not as one-off local constants
- [ ] **Temperature display** either includes the unit or explicitly converts to the user's preferred scale before removing it
- [ ] **Significant digits are configured** when displaying scientific/engineering measurements to avoid false precision
- [ ] **Negative number formatting** accounts for locale differences (hyphen-minus vs. proper minus sign)
- [ ] **Generic numeric code** constrains to the appropriate protocol level (`Numeric`, `FloatingPoint`, `BinaryInteger`, etc.) rather than concrete types
