# Swift Claude Skills

A collection of Swift and SwiftUI skills for [Claude Code](https://claude.com/claude-code).

## Skills

| Skill | Description |
|-------|-------------|
| **swift-kickstart** | Idiomatic Swift patterns from Daniel Steinberg's "A Swift Kickstart" |
| **swift-fp** | Functional programming principles in Swift from "A Functional Programming Kickstart" |
| **swift-codable** | Comprehensive Codable reference from "Flight School Guide to Swift Codable" |
| **swift-strings** | Swift String mastery from "Flight School Guide to Swift Strings" |
| **swift-numbers** | Numeric types, precision, and formatting from "Flight School Guide to Swift Numbers" |
| **swiftui-kickstart** | SwiftUI patterns from Daniel Steinberg's "A SwiftUI Kickstart" |
| **swiftui-pro** | SwiftUI code review for modern APIs, performance, and best practices |

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
