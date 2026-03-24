# Swift Claude Skills

A collection of Swift and SwiftUI skills for [Claude Code](https://claude.com/claude-code).

## Skills

### Book Skills

| Skill | Description |
|-------|-------------|
| **swift-kickstart** | Idiomatic Swift patterns from Daniel Steinberg's "A Swift Kickstart" |
| **swift-fp** | Functional programming principles in Swift from "A Functional Programming Kickstart" |
| **swift-codable** | Comprehensive Codable reference from "Flight School Guide to Swift Codable" |
| **swift-strings** | Swift String mastery from "Flight School Guide to Swift Strings" |
| **swift-numbers** | Numeric types, precision, and formatting from "Flight School Guide to Swift Numbers" |
| **swiftui-kickstart** | SwiftUI patterns from Daniel Steinberg's "A SwiftUI Kickstart" |

### Community Skills

| Skill | Author | Description |
|-------|--------|-------------|
| **swiftui-pro** | Paul Hudson (Hacking with Swift) | SwiftUI code review for modern APIs, performance, and best practices |
| **swiftui-expert-skill** | Antoine van der Lee (SwiftLee) | Expert-level SwiftUI guidance |
| **swift-concurrency** | Antoine van der Lee (SwiftLee) | Swift concurrency patterns — async/await, actors, structured concurrency |
| **swiftui-performance-audit** | Thomas Ricouard (IceCubesApp) | SwiftUI performance auditing and optimization |
| **swiftui-liquid-glass** | Thomas Ricouard (IceCubesApp) | iOS 26 liquid glass design patterns |
| **swiftui-ui-patterns** | Thomas Ricouard (IceCubesApp) | Common SwiftUI UI patterns and components |

## Installation

Add the marketplace to your Claude Code `settings.json`:

```json
{
  "extraKnownMarketplaces": {
    "henryhudson": {
      "source": {
        "source": "github",
        "repo": "henryhudson/claude-plugins"
      }
    }
  },
  "enabledPlugins": {
    "swift-skills@henryhudson": true
  }
}
```
