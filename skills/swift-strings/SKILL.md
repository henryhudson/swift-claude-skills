---
name: swift-strings
description: Swift String mastery covering Unicode model, grapheme clusters, String views (utf8/utf16/unicodeScalars), encoding/decoding, interpolation, formatting, Foundation APIs, localization, regex/Scanner parsing, and NLP
metadata:
  author: Henry Hudson
  version: "1.0"
  source: "Flight School Guide to Swift Strings by Matt"
---

# Swift Strings: Comprehensive Reference

## 1. Unicode Model

### Character Hierarchy
Unicode encodes **characters** (ideas), not **glyphs** (visual representations). The character model has four layers:

1. **Abstract Characters** -- smallest meaningful units of a writing system
2. **Coded Characters** -- each assigned a unique name and code point (e.g., U+0041 LATIN CAPITAL LETTER A)
3. **Character Encoding Forms** -- how code points map to code units (UTF-8, UTF-16, UTF-32)
4. **Character Encoding Schemes** -- byte serialization order (big-endian vs little-endian)

### Code Space
- Range: U+0000 to U+10FFFF (1,114,112 code points across 17 planes)
- **Plane 0 (BMP)**: Most common characters (Latin, CJK, Cyrillic, etc.)
- **Plane 1 (SMP)**: Historical scripts, emoji, musical notation
- **Plane 2 (SIP)**: Additional CJK ideographs
- **Surrogate code points** (U+D800 to U+DFFF): Reserved for UTF-16 encoding, NOT valid scalar values
- **Scalar value**: Any code point that is NOT a surrogate

### Encoding Forms

| Encoding | Code Unit Size | Variable Width | ASCII Compatible |
|----------|---------------|----------------|------------------|
| UTF-8    | 8-bit         | 1-4 units      | Yes              |
| UTF-16   | 16-bit        | 1-2 units      | No               |
| UTF-32   | 32-bit        | Fixed (1 unit)  | No               |

- UTF-8 is the most popular encoding on the web (90%+ of websites)
- UTF-16 is NSString's native encoding
- UTF-32 is memory-inefficient but allows O(1) code-point access

### Composition and Canonical Equivalence
- **Dynamic composition**: Combining marks attach to base characters (e + U+0301 COMBINING ACUTE ACCENT = e)
- **Precomposed characters**: Single code points for common combinations (U+00E9 = e)
- **Canonical equivalence**: Both forms are considered equal in Swift

### Normalization Forms
| Form | Description |
|------|------------|
| NFC  | Canonical Decomposition, then Canonical Composition (precomposed) |
| NFD  | Canonical Decomposition (decomposed) |
| NFKC | Compatibility Decomposition, then Canonical Composition |
| NFKD | Compatibility Decomposition |

### Character Properties
- **Graphic**: letters, numbers, symbols, marks, punctuation, whitespace
- **Format**: invisible but affect neighbors (e.g., ZWJ, directional markers)
- **Control**: inherited from ASCII
- Case mapping, collation, and bidirectionality are locale-dependent

---

## 2. Swift String Fundamentals

### Core Type Relationships
```swift
// String is a Collection of Character values
// Each Character is one grapheme cluster (one or more Unicode.Scalar values)
let family = "\u{1F468}\u{200D}\u{1F469}\u{200D}\u{1F467}\u{200D}\u{1F467}"
family.count          // 1 (single grapheme cluster)
family.unicodeScalars.count  // 7 (four emoji + three ZWJ)
family.utf16.count    // 11
family.utf8.count     // 25
```

### Creating Strings
```swift
// String literal
let s = "Hello"

// Multi-line literal (useful for embedding HTML/JSON)
let multi = """
    Line 1
    Line 2
    """

// Unicode scalar escape
"\u{3030}"  // "~" (WAVY DASH)

// String interpolation
"Result: \(7 + 4)"  // "Result: 11"

// Raw string literals (Swift 5+) -- escape sequences are literal
#"No \n newline here"#           // "No \n newline here"
#"Interpolation: \#(value)"#     // uses \#() for interpolation
```

### String Indexing -- CRITICAL CONCEPT
**Swift strings are NOT indexed by integers.** Accessing the nth character is O(n) because character boundaries require inspecting each scalar value.

```swift
// WRONG -- will not compile
"Hello"[0]

// CORRECT
let s = "Hello"
s[s.startIndex]                              // "H"
s[s.index(after: s.startIndex)]              // "e"
s[s.index(s.startIndex, offsetBy: 4)]        // "o"
s[s.index(s.endIndex, offsetBy: -1)]         // "o"

// Finding indices
s.firstIndex(of: "l")   // String.Index
s.lastIndex(of: "l")    // String.Index

// Ranges and Substrings
let idx = s.index(after: s.startIndex)
s[...idx]    // "He"   (Substring)
s[idx...]    // "ello" (Substring)
```

**Why no integer subscripts?** To prevent accidentally writing O(n^2) code:
```swift
// ACCIDENTALLY QUADRATIC -- each s[i] is O(n)
for i in 0..<string.count {
    let character = string[i]  // Would be O(n) per access
}
```

### Substring
- `Substring` shares a buffer with its originating `String`
- Both conform to `StringProtocol`, making them interchangeable in many contexts
- Convert to `String` when you need to store long-term: `String(substring)`

### Comparing Strings
```swift
// Swift uses canonical equivalence for ==
let precomposed = "expos\u{00E9}"   // e
let decomposed  = "expose\u{0301}"  // e + combining acute
precomposed == decomposed            // true
precomposed.count == decomposed.count // true

// To distinguish canonically equivalent strings, compare encoding views:
precomposed.utf8.elementsEqual(decomposed.utf8) // false
```

### Unicode Encoding Views
```swift
let s = "Tokyo JP"
s.utf8           // UTF8View -- Collection of UInt8
s.utf16          // UTF16View -- Collection of UInt16
s.unicodeScalars // UnicodeScalarView -- Collection of Unicode.Scalar
```

### Unicode Character Properties (Swift 5+)
```swift
// On Unicode.Scalar
scalar.properties.isEmoji
scalar.properties.name     // e.g., "LATIN CAPITAL LETTER A"

// On Character
character.isLetter
character.isNumber
character.isWholeNumber
character.wholeNumberValue  // Int? -- works for all numeral systems
character.isCased
character.isUppercase
character.isLowercase
```

---

## 3. String Protocols and Supporting Types

### Protocol Hierarchy
`String` conforms to: `Sequence`, `Collection`, `BidirectionalCollection`, `RangeReplaceableCollection`, `StringProtocol`, `CustomStringConvertible`, `CustomDebugStringConvertible`, `LosslessStringConvertible`, `TextOutputStreamable`, `ExpressibleByStringLiteral`, `ExpressibleByStringInterpolation`

### Sequence Operations on Strings
```swift
// Functional programming
"Boeing 737".filter { $0.isCased }.map { $0.uppercased() }
// ["B", "O", "E", "I", "N", "G"]

// Splitting
"7A,7B,10F".split(separator: ",")  // ["7A", "7B", "10F"]

// Finding
"pilot".first(where: { "aeiouy".contains($0) })  // "i"
"pilot".contains("i")  // true

// Prefix/suffix extraction
"airport".prefix(3)                         // "air"
"airport".prefix(while: { $0 != "r" })      // "ai"
"airport".suffix(4)                         // "port"

// Dropping
"ticket counter".dropFirst(7)   // "counter"
"ticket counter".dropLast(7)    // "ticket"
```

**Sequence methods to AVOID on String:**
- `shuffled()` -- rarely useful for text
- `sorted()` -- does not use Unicode Collation Algorithm
- `lexicographicallyPrecedes(_:)` -- not locale-aware
- `min()` / `max()` -- unlikely to produce meaningful results
- `enumerated()` -- offsets are NOT `String.Index` values

### Collection Operations
```swift
string.count       // number of Character elements (O(n))
string.isEmpty     // O(1) -- always prefer over count == 0

// Iterate with indices
for (index, character) in zip(string.indices, string) {
    print(index, character)
}

// BidirectionalCollection
string.last(where: isVowel)
String(string.reversed())  // reverse a string
```

### RangeReplaceableCollection
```swift
var s = "plane"
s.append("s")              // "planes"
s.insert("t", at: idx)     // insert at index
s.removeFirst(5)           // remove from start
s.removeLast(3)            // remove from end
s.replaceSubrange(range, with: "new")
s.reserveCapacity(100)     // pre-allocate for performance
```

### StringProtocol -- The Extension Point
Always extend `StringProtocol` (not `String`) to share functionality with `Substring`:
```swift
extension StringProtocol {
    var isPalindrome: Bool {
        let normalized = self.lowercased().filter { $0.isLetter }
        return normalized.elementsEqual(normalized.reversed())
    }
}

// Works on both String and Substring
"racecar".isPalindrome              // true
"racecar"[...].isPalindrome         // true (Substring)
```

### CustomStringConvertible / CustomDebugStringConvertible
```swift
// print() uses String(describing:) -> description
// debugPrint() uses String(reflecting:) -> debugDescription

struct FlightCode: CustomStringConvertible {
    let airlineCode: String
    let flightNumber: Int
    var description: String { "\(airlineCode) \(flightNumber)" }
}
```

### ExpressibleByStringInterpolation (Swift 5)
Allows custom types to parameterize interpolation behavior:
```swift
extension StyledString: ExpressibleByStringInterpolation {
    struct StringInterpolation: StringInterpolationProtocol {
        mutating func appendLiteral(_ literal: String) { ... }
        mutating func appendInterpolation<T>(_ value: T, style: Style) { ... }
    }
}
// Usage:
let styled: StyledString = "Hello, \(name, style: .bold)!"
```

### TextOutputStream
Create custom output targets:
```swift
struct StderrOutputStream: TextOutputStream {
    mutating func write(_ string: String) {
        fputs(string, stderr)
    }
}
var standardError = StderrOutputStream()
print("Error!", to: &standardError)
```

---

## 4. Foundation String APIs

### NSString Bridging -- KEY DIFFERENCE
**NSString uses UTF-16 code units for all positions, ranges, and lengths. Swift String uses extended grapheme clusters.** This is the single most important interop difference.

```swift
import Foundation

// Convert between Range<String.Index> and NSRange
let nsRange = NSRange(string.startIndex..<string.endIndex, in: string)
let range = Range(nsRange, in: string)

// Get UTF-16 length (for NSString compatibility)
let length = string.utf16.count

// Cast between String and NSString
let nsstring = string as NSString
```

**Warning:** Some Foundation methods operate on UTF-16 code units, not grapheme clusters:
```swift
// padding(toLength:) counts UTF-16 code units, not characters!
let emoji = "\u{1F469}\u{200D}\u{1F467}"  // one character
emoji.count  // 1
emoji.padding(toLength: 2, withPad: " ", startingAt: 0)  // BREAKS the emoji
```

### Localized String Operations

#### Collation and Comparison
```swift
// ALWAYS use localizedStandardCompare for user-facing sort order
files.sorted { $0.localizedStandardCompare($1) == .orderedAscending }
// Sorts "File 3", "File 7", "File 36" correctly (numeric-aware)

// Locale-specific comparison
lhs.compare(rhs, options: [], range: nil, locale: Locale(identifier: "sv_SE"))
// Swedish: A < Z < A < O
// English: A ~= A (diacritics ignored)
```

#### Searching
```swift
// Non-localized (character-level exact match)
"Eclair".contains("E")                     // false (e != E)

// Localized (diacritic-insensitive, case-insensitive)
"Eclair".localizedStandardContains("E")    // true
"Eclair".localizedStandardRange(of: "E")   // non-nil
```

#### Case Mapping
```swift
// Non-localized (context-independent)
"hello".uppercased()   // "HELLO"
"HELLO".lowercased()   // "hello"

// Localized -- USE THESE for user-facing text
"hello".localizedUppercase
"HELLO".localizedLowercase

// Turkish: dotted capital I (I) lowercases differently
let turkish = Locale(identifier: "tr")
"I".lowercased(with: turkish)  // "\u{0131}" (dotless i)
```

### Normalization
```swift
import Foundation
let nfc = string.precomposedStringWithCanonicalMapping    // NFC
let nfd = string.decomposedStringWithCanonicalMapping     // NFD
let nfkc = string.precomposedStringWithCompatibilityMapping  // NFKC
let nfkd = string.decomposedStringWithCompatibilityMapping   // NFKD
```

### String-Data Conversion
```swift
// Reading files
let string = try String(contentsOf: url)
let string = String(data: data, encoding: .utf8)

// Writing to Data
let data = string.data(using: .utf8)!

// Legacy encodings
string.data(using: .macOSRoman)
string.canBeConverted(to: .ascii)  // check first
string.data(using: .ascii, allowLossyConversion: true)  // "Avion ??"
```

### String Transforms (ICU)
```swift
import Foundation

// Strip diacritics
"Avion".applyingTransform(.stripDiacritics, reverse: false)  // "Avion"

// Transliterate to Latin
"angul".applyingTransform(.toLatin, reverse: false)  // romanized form

// Other transforms: .toXMLHex, .stripCombiningMarks, .fullwidthToHalfwidth
```

### Formatting with Format Specifiers
```swift
String(format: "%04d", 1)        // "0001" (zero-padded)
String(format: "%02X", 0x0F)     // "0F"   (hex, uppercase)
String(format: "%.2f", 3.14159)  // "3.14" (2 decimal places)
String(format: "%+d", 42)        // "+42"  (explicit sign)
String(format: "%@", "Hello")    // "Hello" (string specifier)
String(format: "%d%%", 100)      // "100%"  (literal percent)

// Locale-aware formatting
String(format: "%f", locale: Locale(identifier: "fr_FR"), 12345.67)
// "12 345,670000" (French: space for thousands, comma for decimal)
```

**Best practice:** Use `NumberFormatter` for user-facing numeric display, not format strings.

---

## 5. Binary-to-Text Encoding

### Radix Conversions
```swift
// Binary (base 2)
String(0xF0 as UInt8, radix: 2)              // "11110000"
UInt8("11110000", radix: 2)!                  // 240

// Hexadecimal (base 16)
String(0xF0 as UInt8, radix: 16, uppercase: true)  // "F0"
UInt8("F0", radix: 16)!                             // 240

// Hex dump of Data
data.map { String($0, radix: 16, uppercase: true) }  // ["C0", "FF", "EE"]
```

### Base64 Encoding
```swift
import Foundation

// Encode
let data = "Hello!".data(using: .utf8)!
let encoded = data.base64EncodedString()  // "SGVsbG8h"

// Decode
let decoded = Data(base64Encoded: encoded)!
String(data: decoded, encoding: .utf8)    // "Hello!"

// With options (e.g., line length for MIME)
data.base64EncodedString(options: .lineLength64Characters)
```

Use cases: email attachments (MIME), embedding assets in HTML/CSS via data URIs, embedding resources in code.

---

## 6. Parsing Data from Text

### Foundation Scanner
`Scanner` scans strings sequentially, extracting substrings and numbers. Its Objective-C API is awkward in Swift (returns NSString by reference), so wrapping it is recommended:

```swift
import Foundation

// Basic Scanner usage
let scanner = Scanner(string: text)
scanner.caseSensitive = true
scanner.charactersToBeSkipped = .whitespacesAndNewlines

// Improved Swift wrapper pattern
extension Scanner {
    struct Error: Swift.Error {
        let location: Int
    }

    func scan(_ characters: CharacterSet) throws -> String {
        var result: NSString?
        guard scanCharacters(from: characters, into: &result),
              let string = result as String?
        else { throw Error(location: scanLocation) }
        return string
    }
}
```

### Regular Expressions (NSRegularExpression)
```swift
import Foundation

let pattern = #"(\w+)\s+(\d+)"#
let regex = try NSRegularExpression(pattern: pattern)
let range = NSRange(string.startIndex..<string.endIndex, in: string)

regex.enumerateMatches(in: string, range: range) { match, _, _ in
    guard let match = match else { return }
    // Access capture groups
    if let groupRange = Range(match.range(at: 1), in: string) {
        let captured = String(string[groupRange])
    }
}
```

### Parsing Approaches (Summary)
1. **Scanner** -- Good for semi-structured text with mixed types
2. **NSRegularExpression** -- Pattern matching with capture groups
3. **Parser generators (ANTLR)** -- Formal grammars for complex formats

---

## 7. Natural Language Processing

### NLTokenizer -- Tokenization
**Never use `split(separator:)` for natural language text.** Use `NLTokenizer` instead -- it handles Chinese (no word spaces), different sentence terminators, and line ending conventions.

```swift
import NaturalLanguage

let tokenizer = NLTokenizer(unit: .word)  // .document, .paragraph, .sentence, .word
tokenizer.string = text

tokenizer.enumerateTokens(in: text.startIndex..<text.endIndex) { range, attrs in
    let token = text[range]
    return true  // continue
}
```

### NLTagger -- Part of Speech and Named Entity Recognition
```swift
import NaturalLanguage

// Part of speech tagging
let tagger = NLTagger(tagSchemes: [.lexicalClass])
tagger.string = "The sleek jet soars."
tagger.enumerateTags(in: range, unit: .word, scheme: .lexicalClass,
                     options: [.omitWhitespace, .omitPunctuation]) { tag, range in
    // tag?.rawValue -> "Noun", "Verb", "Adjective", etc.
    return true
}

// Named entity recognition
let nerTagger = NLTagger(tagSchemes: [.nameType])
nerTagger.string = text
// options: [.omitWhitespace, .omitPunctuation, .joinNames]
// Tags: .personalName, .placeName, .organizationName
```

### NLLanguageRecognizer
```swift
let recognizer = NLLanguageRecognizer()
recognizer.processString(text)
recognizer.dominantLanguage  // e.g., .english, .japanese
```

### Edit Distance (Levenshtein)
Measures minimum single-character edits (insert, delete, substitute) to transform one string into another. Useful for fuzzy matching and spell checking. Can be generalized to any `Equatable` sequence.

### N-grams and Markov Chains
- **Bigrams/trigrams**: Sliding windows over tokens for statistical text analysis
- **Markov chains**: Predict next word based on current state using transition probabilities from a corpus

---

## Common Pitfalls and How to Avoid Them

### 1. Character Count vs Byte Count
```swift
let emoji = "family"  // family emoji with ZWJ
emoji.count             // 1 character
emoji.utf8.count        // 25 bytes
emoji.utf16.count       // 11 code units
emoji.unicodeScalars.count  // 7 scalars
```
**Fix:** Always be explicit about which "length" you mean. Use `.count` for user-perceived characters, `.utf8.count` for byte length, `.utf16.count` for NSString/Objective-C compatibility.

### 2. NSString vs String Length Mismatch
NSString operates on UTF-16 code units. Foundation methods like `padding(toLength:)` silently use UTF-16 lengths, which can break emoji and multi-scalar characters.

**Fix:** Use comments and unit tests when mixing Foundation text APIs. Convert between `NSRange` and `Range<String.Index>` explicitly.

### 3. Integer Subscripting
Swift deliberately prevents `string[0]` to avoid O(n^2) bugs.

**Fix:** Use `string.startIndex`, `string.index(after:)`, `string.index(_:offsetBy:)`.

### 4. Using `count == 0` Instead of `isEmpty`
`count` iterates the entire string (O(n)). `isEmpty` is O(1).

### 5. Using `enumerated()` on Strings
The offset from `enumerated()` is an `Int`, NOT a `String.Index`. Do not use it as an index.

**Fix:** Use `zip(string.indices, string)` instead.

### 6. Non-localized String Comparison
`==` uses canonical equivalence but is locale-insensitive. Sorting with `<` does not follow locale collation rules.

**Fix:** Use `localizedStandardCompare(_:)` for sorting, `localizedStandardContains(_:)` for searching, `localizedCaseInsensitiveCompare(_:)` for comparison.

### 7. Non-localized Case Mapping
Turkish dotted-I, Lithuanian accented-i, and other locales have context-dependent case rules.

**Fix:** Use `localizedUppercase` / `localizedLowercase` for user-facing text.

### 8. Naive Tokenization of Natural Language
Splitting on spaces/periods fails for Chinese, Japanese, and many other languages.

**Fix:** Use `NLTokenizer` from the Natural Language framework.

### 9. Encoding Assumptions
Assuming UTF-8 when reading files without verifying can produce mojibake.

**Fix:** Specify encoding explicitly in `String(data:encoding:)`. Use BOM detection or metadata when available.

### 10. Forgetting Canonical Equivalence
Two strings can be `==` equal but have different byte representations (precomposed vs decomposed).

**Fix:** If you need binary equality, compare `.utf8` views. If you need to normalize for storage or transmission, use NFC (`precomposedStringWithCanonicalMapping`).

---

## When Reviewing Code Checklist

- [ ] **String indexing**: No integer subscripts; uses `String.Index` properly
- [ ] **Empty checks**: Uses `.isEmpty` instead of `.count == 0`
- [ ] **Iteration with indices**: Uses `zip(string.indices, string)`, not `enumerated()`
- [ ] **Localized operations**: User-facing sorting uses `localizedStandardCompare`; searching uses `localizedStandardContains`; case mapping uses `localizedUppercase`/`localizedLowercase`
- [ ] **NSString/NSRange interop**: Explicit conversion with `NSRange(_:in:)` and `Range(_:in:)`; awareness that Foundation methods may count UTF-16 code units
- [ ] **Character vs scalar vs byte counts**: Code uses the correct "length" for its context
- [ ] **Canonical equivalence awareness**: Comparison semantics are intentional (== for canonical, .utf8 for binary)
- [ ] **Encoding specified**: `String(data:encoding:)` calls specify encoding rather than relying on inference
- [ ] **Natural language tokenization**: Uses `NLTokenizer` instead of `split(separator:)` for human text
- [ ] **String extensions**: Defined on `StringProtocol` (not just `String`) to also cover `Substring`
- [ ] **Emoji/multi-scalar safety**: Code handles characters composed of multiple scalars (flags, family emoji, skin tones) correctly
- [ ] **Format strings**: User-facing numbers use `NumberFormatter`, not `String(format:)`; `NSLocalizedString` used for localized text
- [ ] **Performance**: No accidental O(n^2) from repeated index-offset calculations in loops; `reserveCapacity` used when building large strings
- [ ] **Normalization**: Strings normalized before comparison at API boundaries if needed (NFC for interchange)
