---
name: swiftui-kickstart
description: Apply Daniel Steinberg's SwiftUI Kickstart patterns with modern APIs (iOS 17+/18+). Value-type views, declarative composition, state-driven UI, @Observable data flow, NavigationStack, and proportional layout. Use when writing or reviewing SwiftUI code.
metadata:
  author: Henry Hudson
  version: "2.0"
  source: "A SwiftUI Kickstart by Daniel Steinberg (patterns modernized for iOS 17+/18+)"
---

Apply SwiftUI patterns and principles from "A SwiftUI Kickstart" when writing, reviewing, or refactoring SwiftUI code. These represent a practical, Steinberg-style approach to SwiftUI development, updated to modern APIs.

## Core Principles

### 1. Views Are Value Types

- All SwiftUI views are structs, not classes. They are lightweight, cheap, and disposable.
- You don't mutate views. When state changes, SwiftUI creates new instances with the new values.
- This is fundamentally different from UIKit where you held references to views and mutated their properties.

```swift
// SwiftUI way - declarative, value-type
struct ContentView: View {
    var body: some View {
        Text("Hello")
            .font(.largeTitle)
            .foregroundStyle(.purple)
    }
}
```

### 2. Opaque Return Types (`some View`)

- `some View` means "returns exactly one concrete type conforming to View, but the caller doesn't need to know which."
- The compiler infers the actual return type. This works because `View` has an associated type (`Body`) that must be resolved.
- ViewBuilders handle cases where `body` contains conditional or multiple views by wrapping them in types like `_ConditionalContent` or `TupleView`.

### 3. Compose with Stacks, Not Implicit Layout

Always use explicit stack containers rather than relying on compiler-inserted wrappers:

```swift
// Explicit and intentional
VStack {
    Text("Title")
    HStack {
        Button("Back", action: back)
        Button("Forward", action: forward)
    }
}
```

- `VStack` - vertical arrangement
- `HStack` - horizontal arrangement
- `ZStack` - layered arrangement (back to front)
- Stacks nest freely. Each is a struct conforming to `View`.

### 4. Modifiers Wrap, They Don't Mutate

Modifiers create new wrapper views. They don't change existing views.

```swift
Text("Hello")
    .padding()
    .background(.blue)
    .clipShape(.rect(cornerRadius: 8))
```

**Key behaviors:**
- Modifiers on a container set values in the **environment** for all children
- Children can **override** inherited modifier values
- Order matters: `.padding().background(.blue)` differs from `.background(.blue).padding()`
- Use `ShapeStyle` directly (`.blue`) rather than spelling out `Color.blue`

### 5. State-Driven UI (Single Direction Data Flow)

Actions change state. State changes update UI. Never try to imperatively update views.

```swift
// The button does NOT present the sheet.
// The button changes a Bool, and as a RESULT the sheet appears.
Button("Show Sheet") {
    isSheetDisplayed = true  // change state
}
.sheet(isPresented: $isSheetDisplayed) {  // UI reacts to state
    SheetContent()
}
```

### 6. Property Wrapper Decision Tree

Choose the right property wrapper based on ownership and type:

| Situation | Wrapper | Notes |
|-----------|---------|-------|
| Value type, owned by this view | `@State` | Always `private`. Initial value required. |
| Value type, owned by parent | `@Binding` | Two-way connection. Use `$property` to pass. |
| `@Observable` class, created here | `@State` | View owns the lifecycle. |
| `@Observable` class, passed in (read-only) | plain `var` or `let` | No wrapper needed — observation is automatic. |
| `@Observable` class, passed in (need bindings) | `@Bindable` | Enables `$property` access on observed object. |
| `@Observable` class, shared via environment | `@Environment` | Injected via `.environment()`. |
| Small persistent value | `@AppStorage` | Backed by UserDefaults. |

**How they connect:**

```swift
// Owner creates with @State
struct Parent: View {
    @State private var count = 0
    @State private var viewModel = ViewModel()

    var body: some View {
        // Pass value binding down with $
        CounterView(count: $count)
        // Pass observable object — no wrapper needed on receiving end
        DetailView(viewModel: viewModel)
    }
}

// Child receives value binding with @Binding
struct CounterView: View {
    @Binding var count: Int  // can read AND write
}

// Child receives @Observable object — just a plain property
struct DetailView: View {
    var viewModel: ViewModel  // observation is automatic

    // If you need to create bindings into the object:
    // @Bindable var viewModel: ViewModel
}
```

### 7. Observation with @Observable

The pattern for separating model from view:

```swift
// Model - immutable value type
struct Model {
    let value: Int

    var decrease: Model { Model(value: value - 1) }
    var increase: Model { Model(value: value + 1) }
}

// View Support / ViewModel - reference type, observable
@Observable
class ContentViewSupport {
    private var model = Model(value: 0)

    var currentValue: Int { model.value }
    func back() { model = model.decrease }
    func forward() { model = model.increase }
}

// View - value type, observes automatically
struct ContentView: View {
    @State private var support = ContentViewSupport()

    var body: some View {
        VStack {
            Text("\(support.currentValue)")
            HStack {
                Button("Back", action: support.back)
                Button("Forward", action: support.forward)
            }
        }
    }
}
```

What makes `@Observable` simpler:
1. Mark the class with `@Observable` — no protocol conformance needed
2. All stored properties are automatically tracked — no `@Published`
3. View holds reference with `@State` — no `@StateObject` vs `@ObservedObject` distinction
4. SwiftUI tracks **which properties each view actually reads**, so views only re-evaluate when their specific dependencies change (fine-grained observation)

### 8. Environment Objects

Share state across the entire view hierarchy without passing through every level:

```swift
// Define
@Observable
class Accent {
    var color: Color = .red
}

// Inject at top level
ContentView()
    .environment(Accent())

// Pull out at any depth
struct DeepChild: View {
    @Environment(Accent.self) private var accent
    // use accent.color directly
}
```

**Important:** Sheets and other detached presentations may not inherit the environment. Pass `.environment()` explicitly to sheet content when needed.

### 9. Extract and Compose Views

**Programming by intention**: write the call site you want, then build the types to make it compile.

```swift
// Write this first
var body: some View {
    VStack {
        IntDisplay(value: currentValue)
        SymbolButton("arrow.right", action: forward)
    }
}

// Then build IntDisplay and SymbolButton
struct IntDisplay: View {
    let value: Int
    var body: some View {
        Text(value.description)
            .font(.largeTitle)
    }
}
```

**Separate struct from View conformance** for clarity:

```swift
struct ContentView {
    @State private var currentValue = 0
}

extension ContentView: View {
    var body: some View { /* ... */ }
}
```

### 10. Custom ViewModifiers

When multiple views share the same modifier chain, extract to a `ViewModifier`:

```swift
struct DisplayModifier: ViewModifier {
    func body(content: Content) -> some View {
        content
            .font(.largeTitle)
            .padding()
            .background(.blue.opacity(0.1))
            .clipShape(.rect(cornerRadius: 8))
    }
}

// Usage
Text("Hello").modifier(DisplayModifier())
```

### 11. Lists, ForEach, and Grids

**ForEach** generates views from collections (you cannot use `for` loops in ViewBuilders):

```swift
// Preferred: model conforms to Identifiable
ForEach(items) { item in
    ItemRow(item: item)
}

// Alternative: provide key path
ForEach(names, id: \.self) { name in
    Text(name)
}
```

**List** provides scrolling, laziness, and standard styling for free:

```swift
List(items) { item in
    ItemRow(item: item)
}
```

Separate `List` and `ForEach` when you need `.onDelete` or `.onMove`.

**Grids** use `LazyVGrid` with `GridItem` columns:

```swift
let columns = [
    GridItem(.adaptive(minimum: 60))  // responsive
]
ScrollView {
    LazyVGrid(columns: columns) {
        ForEach(items) { item in
            ItemView(item: item)
        }
    }
}
```

GridItem types: `.flexible()`, `.fixed(200)`, `.adaptive(minimum: 60)`

### 12. Navigation

Use `NavigationStack` for push-based navigation with type-safe destinations:

```swift
NavigationStack {
    List(items) { item in
        NavigationLink(value: item) {
            ItemRow(item: item)
        }
    }
    .navigationTitle("Items")
    .toolbar { EditButton() }
    .navigationDestination(for: Item.self) { item in
        DetailView(item: item)
    }
}
```

For multi-column layouts (iPad/Mac sidebar patterns), use `NavigationSplitView`:

```swift
NavigationSplitView {
    List(items, selection: $selectedItem) { item in
        Text(item.name)
    }
} detail: {
    if let selectedItem {
        DetailView(item: selectedItem)
    } else {
        ContentUnavailableView("Select an Item", systemImage: "sidebar.left")
    }
}
```

- Attach `.navigationTitle` to the content inside, not the `NavigationStack`
- Use `TabView` for tab-based navigation
- Use `.sheet(isPresented:)` or `.sheet(item:)` for modal presentation
- Use `.navigationDestination(for:)` to define destinations by type — keeps navigation logic declarative

### 13. Proportional Layout

Avoid hard-coded sizes. Prefer proportional and responsive layout:

**`containerRelativeFrame` (iOS 17+)** — the simplest way to size relative to the container:

```swift
Image("photo")
    .resizable()
    .scaledToFit()
    .containerRelativeFrame(.horizontal) { width, _ in
        width * 0.8
    }
```

**`GeometryReader`** — when you need full access to the proposed size:

```swift
GeometryReader { proxy in
    let size = min(proxy.size.width, proxy.size.height)
    Circle()
        .frame(width: size * 0.8, height: size * 0.8)
        .offset(y: -size * 0.1)
}
```

- Parent offers space, child says how much it needs
- Shapes are greedy (claim all offered space)
- Derive a key dimension and express everything as fractions of it
- Prefer `containerRelativeFrame` when you only need one axis; use `GeometryReader` when you need both dimensions or coordinate offsets

### 14. Custom Drawing with Path

```swift
var path = Path()
path.move(to: startPoint)
path.addLine(to: endPoint)
path.addCurve(to: end, control1: cp1, control2: cp2)

path.stroke(lineWidth: size / 50)
    .frame(width: size, height: size / 4)
```

Pre-compute named constants for coordinates. Make line widths relative to dimensions.

### 15. Persistence with @AppStorage

```swift
@AppStorage("moodRating") private var moodRating = 50.0
@State private var rating = 50.0  // separate UI state

var body: some View {
    Slider(value: $rating, in: 0...100)
        .onAppear { rating = moodRating }
        .onChange(of: scenePhase) { oldValue, newValue in
            if newValue == .background { moodRating = rating }
        }
}
```

Don't bind `@AppStorage` directly to frequently-updating controls. Keep a separate `@State` and sync at appropriate moments.

## When Reviewing SwiftUI Code

Look for these improvement opportunities:

1. **Class where struct would work** - does this view genuinely need reference semantics?
2. **Imperative UI updates** - are you trying to modify views instead of changing state?
3. **Wrong property wrapper** - is ownership and type correctly matched?
4. **Missing `private` on `@State`** - `@State` should always be private
5. **Hard-coded layout values** - should these use `containerRelativeFrame` or `GeometryReader`?
6. **Duplicated modifier chains** - extract to a custom ViewModifier
7. **Monolithic views** - extract sub-views with properties
8. **`@AppStorage` bound to Slider** - separate UI state from persistence
9. **Using `ObservableObject`/`@Published`** - prefer `@Observable` for iOS 17+
10. **Using `NavigationView`** - use `NavigationStack` or `NavigationSplitView`
11. **Using `.cornerRadius()`** - prefer `.clipShape(.rect(cornerRadius:))`
12. **Using `@StateObject`/`@ObservedObject`** - with `@Observable`, just use `@State` or plain property

## Philosophy

- Views are cheap and disposable. Don't be afraid to create many small ones.
- "Programming by intention" - write the call site first, then build the support.
- Data flows in one direction: actions -> state changes -> UI updates.
- Separate what a view *is* (stored properties) from how it *looks* (body).
- Observation should be automatic and fine-grained — `@Observable` tracks exactly what each view reads.
- "Hire a designer and listen to them."
