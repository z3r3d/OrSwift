# Copilot Instructions for OrSwift

## Repository Overview

**OrSwift** (`Or`) is a lightweight Swift package that provides ergonomic helpers for handling `Optional` values with clear, predictable defaults. Instead of sprinkling `?? ""` or `?? 0` everywhere, consumers use named properties (`.orEmpty`, `.orZero`, `.orTrue`, `.orFalse`) and a generic `.or(_:)` method.

- **Current version:** `1.0.2` (see `.version`)
- **Swift Tools version:** 5.10 (see `Package.swift`)
- **License:** MIT
- **Package name:** `Or` — the single importable module is `import Or`

---

## Repository Layout

```
OrSwift/
├── .github/
│   ├── copilot-instructions.md   ← this file
│   └── workflows/
│       └── swift.yml             ← CI: build + test on macos-latest
├── Sources/
│   └── Or/
│       └── Or.swift              ← entire library implementation (single file)
├── Tests/
│   └── OrTests/
│       └── OrTests.swift         ← XCTest suite (single file)
├── Package.swift
├── README.md
├── .version
└── LICENSE
```

All production code lives in **one file**: `Sources/Or/Or.swift`.  
All tests live in **one file**: `Tests/OrTests/OrTests.swift`.

---

## Build & Test

This is a pure Swift Package Manager project. There is **no Xcode project file**.

```bash
# Build
swift build

# Run all tests (parallel)
swift test --parallel

# Verbose build (matches CI)
swift build -v
swift test -v --parallel
```

CI runs on `macos-latest` via `.github/workflows/swift.yml`, triggered on push/PR to `main`.

> **Note:** `swift build` and `swift test` require a macOS host or a Linux host with Swift 5.10+ installed. This package has no Linux-specific code, but the CI is macOS-only.

---

## Architecture & Key Abstractions

### Protocols

| Protocol | Purpose |
|---|---|
| `Orable` | Defines `func or(_ value: OrValue) -> OrValue` — adopted by the `Optional` extension |
| `Thisable` | Defines `static func this<T>(optional:default:) -> T` — default implementation via a `Thisable` protocol extension; `Or` conforms |

### Public API Surface

| API | Where declared | What it does |
|---|---|---|
| `Optional.or(_:)` | `extension Optional: Orable` | Generic fallback for any optional |
| `Or.this(optional:default:)` | `public final class Or: Thisable` | Static explicit unwrap helper |
| `Optional<StringProtocol>.orEmpty` | Extension on `Optional where Wrapped: StringProtocol` | Returns `""` when nil |
| `Optional<Bool>.orTrue` | Extension on `Optional where Wrapped == Bool` | Returns `true` when nil |
| `Optional<Bool>.orFalse` | Extension on `Optional where Wrapped == Bool` | Returns `false` when nil |
| `Optional<Numeric>.orZero` | Extension on `Optional where Wrapped: Numeric` | Returns `.zero` when nil |
| `Optional<Collection>.orEmpty` | Extension on `Optional where Wrapped: Collection` | Returns empty Array/Set/Dictionary when nil; `fatalError` for unsupported types |
| `OrNameArt` | Top of `Or.swift` | ASCII art banner string — decorative, not functional |

### Implementation Notes

- Computed properties on Optional extensions use `@inlinable` + `@inline(__always)` for zero-overhead access.
- `Optional.orEmpty` on `Collection` uses runtime type-casting (`[] as? Wrapped`, `[:] as? Wrapped`, `Set<AnyHashable>([]) as? Wrapped`) and calls `fatalError` for unsupported collection types. This is intentional — adding new supported collection types requires extending this guard block.
- `Or` is a `final class` with a `private init()` — it is a pure namespace, not meant to be instantiated.
- There are **no external dependencies** — the package is dependency-free.

---

## Coding Conventions

- **File header:** Every Swift file starts with the standard comment block (filename, project name, author, copyright year, license).
- **MARK sections:** Use `// MARK: - Core` and `// MARK: Adoption` to organize sections in `Or.swift`.
- **Documentation comments:** Most public types, protocols, properties, and methods have `///` doc comments with `## Example Usage` and `- Returns:` / `- Complexity:` tags (see `Sources/Or/Or.swift`).
- **Naming:** Follow Swift API Design Guidelines. New optional-unwrapping helpers follow the `or<DefaultName>` naming pattern (e.g., `orEmpty`, `orZero`, `orTrue`, `orFalse`).
- **No force-unwraps** in the library itself; `fatalError` is used only in the unsupported collection branch of `.orEmpty`.

---

## Testing Conventions

- Framework: **XCTest** (`import XCTest`, `@testable import Or`).
- Test class: `final class OrTests: XCTestCase`.
- Each test method follows the pattern:
  1. Define a `failureMessage` string.
  2. Set up test variables (nil and non-nil).
  3. Assert expected behavior with `XCTAssert*`.
- All new public API must have corresponding test coverage in `Tests/OrTests/OrTests.swift`.
- Run tests with `swift test --parallel`.

---

## How to Extend the Library

To add support for a **new type-specific optional default** (e.g., `orNaN` for floating-point):

1. Add a new `extension Optional where Wrapped == <Type>` block in `Sources/Or/Or.swift`, following the existing property style with `@inlinable` / `@inline(__always)`.
2. Add `///` documentation following existing conventions.
3. Add test cases in `Tests/OrTests/OrTests.swift`.
4. Update `README.md` (Supported Types, Usage, and Reference sections).

To add a **new collection type** to `.orEmpty`:

1. Extend the `guard let self = self else { ... }` block in `extension Optional where Wrapped: Collection` with an additional `if let empty... as? Wrapped` branch.
2. Add a test case in `OrTests.swift`.

---

## Known Issues & Workarounds

- **`Collection.orEmpty` and custom collection types:** The current implementation only supports `Array`, `Set`, and `Dictionary` via runtime casting. Any other `Collection`-conforming type will hit `fatalError` at runtime. This is a known design limitation; document it clearly if you add new collection support.
- **`StringProtocol.orEmpty` returns `""` literal:** The `guard let self = self else { return "" }` returns a `String` literal cast to `Wrapped`. This works for `String` and `Substring` but may behave unexpectedly for custom `StringProtocol` conformances. Prefer using `.or("fallback")` for non-standard `StringProtocol` types.
- **macOS-only CI:** The workflow runs on `macos-latest`. There is no Linux CI. If Linux compatibility is important, a separate job should be added.

---

## PR & Contribution Guidelines (from README)

- Small fixes/changes: submit a PR with fix, unit tests, and updated tech docs.
- Major changes: open an issue first to discuss.
- PRs target the `main` branch.
