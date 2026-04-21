---
name: john-sundell
description: Use when writing, reviewing, refactoring, or explaining Swift/SwiftUI/UIKit code for iOS/macOS — especially for design choices involving struct vs class, protocol vs concrete type, composition vs inheritance, singleton vs dependency injection, extension-based organization, and Swift API naming.
---

# John Sundell-style Swift/iOS Development

## Overview

Write Swift the Swift way, not the Java way or the architecture-astronaut way.

**Core principle:** Value types by default, compose small, inject dependencies, let types tell the story.

**Pragmatism over dogma.** Every rule below has an exception. Name the tradeoff out loud — don't hide behind "best practices."

## When to Use

- Designing Swift types (model, service, coordinator, view, view model)
- Reviewing Swift code — spotting over-abstraction, hidden dependencies, raw stringly-typed APIs
- Refactoring legacy iOS code with singletons, deep inheritance, or giant view controllers
- Naming types, methods, booleans following Swift API Design Guidelines
- Questions framed as "how would you structure…", "is this too much abstraction?", "struct or class?"

## When NOT to Use

- Language/platform outside Swift / iOS / macOS
- A project-specific style guide (CLAUDE.md) conflicts with this skill — **the user's rules win**
- Keeping legacy architecture unchanged is the lower-risk choice
- Swift Concurrency deep dives → use `swift-concurrency` skill
- SwiftUI-specific deep topics (state details, Liquid Glass, performance) → use `swiftui-expert-skill`

## Quick Reference

| # | Principle | Apply when |
|---|---|---|
| 1 | Value types by default | Modeling data, transferring state, anything without identity |
| 2 | Protocols abstract behavior, not test mocks | 2+ real impls or genuine polymorphism |
| 3 | Composition over inheritance | Sharing behavior between unrelated types |
| 4 | Catch bugs at compile time via the type system | Any raw `String`/`Int` carrying domain meaning |
| 5 | Inject dependencies, no hidden singletons | Anything a type reaches for that isn't a pure value |
| 6 | Organize with extensions | Protocol conformances, grouped helpers |
| 7 | Testability is a design signal | Hard-to-test = design problem, not test-framework problem |
| 8 | Naming is documentation | Every type, method, boolean |
| 9 | Respect both SwiftUI and UIKit | New work SwiftUI-first; UIKit is valid |
| 10 | Refactor incrementally; rule of three | Before generalizing, wait for 3 instances |

## The Ten Principles

### 1. Value types by default

`struct` / `enum` first. `class` only when identity, inheritance, or ObjC interop demands it. Start with `let`; reach for `var` only when mutation is actually needed. Models are almost always structs — adopt `Equatable`/`Hashable`/`Codable` eagerly.

```swift
// Bad: Reference type for pure data — identity you don't need, synthesized conformances you lose
final class Article {
    var id: String
    var title: String
    init(id: String, title: String) { self.id = id; self.title = title }
}

// Good: Struct by default
struct Article: Equatable, Hashable, Codable {
    let id: ArticleID
    let title: String
}
```

### 2. Protocols abstract behavior, not test mocks

Ask **"are there actually multiple implementations, or is this just for a mock?"** If one impl, use the concrete type. When a protocol genuinely needs a default, extend it with a `where` clause.

```swift
// Bad: Protocol exists only so tests can inject a fake
protocol UserServiceProtocol { func load(id: UserID) async throws -> User }
final class UserService: UserServiceProtocol { /* only impl */ }

// Good: Concrete type; inject a closure or configurable dependency if you need a test seam
struct UserService {
    let load: (UserID) async throws -> User
}
```

### 3. Composition over inheritance

Replace forced "is-a" relationships with "has-a" composition. Instead of `BaseViewController` that every screen inherits, extract `LoadingPresenter`, `ErrorHandler`, etc. and inject them.

```swift
// Bad: Shared behavior via inheritance
class BaseViewController: UIViewController { func showLoading() { … } func showError(_: Error) { … } }
class ArticleViewController: BaseViewController { … }

// Good: Composition
final class ArticleViewController: UIViewController {
    private let loading: LoadingPresenter
    private let errors: ErrorPresenter
    init(loading: LoadingPresenter, errors: ErrorPresenter) { … }
}
```

### 4. Catch bugs at compile time

Wrap raw `String`/`Int` that carry domain meaning. Use `enum` to make illegal states unrepresentable. Reach for phantom types, generic constraints, `@dynamicMemberLookup` **when they solve a real problem.**

```swift
// Bad: Compiles even when arguments are swapped
func fetchUser(id: String, token: String)

// Good: The compiler stops you
struct UserID: Hashable { let rawValue: String }
struct AuthToken: Hashable { let rawValue: String }
func fetchUser(id: UserID, token: AuthToken)
```

### 5. Inject dependencies

No singletons, no `static` globals, no hidden reach-outs. Accept dependencies in `init`. If the `init` grows long, the type is doing too much.

```swift
// Bad: Hidden singleton dependency
struct ArticleLoader {
    func load() async throws -> [Article] { try await Networking.shared.fetch(.articles) }
}

// Good: Dependencies visible in the signature
struct ArticleLoader {
    let networking: Networking
    func load() async throws -> [Article] { try await networking.fetch(.articles) }
}
```

See `references/anti-patterns.md` for the full singleton → DI refactor walkthrough.

### 6. Organize with extensions

The main declaration holds core properties and public API. Split protocol conformances into `extension`s — one extension, one responsibility. Answer "what is this file about?" at a glance.

```swift
// Bad: Every concern piled into the main declaration
struct Article: Equatable, Hashable, Codable {
    // core properties
    // Equatable impl
    // Hashable impl
    // Codable impl
    // formatting helpers
    // 200 lines, unclear what this file is "about"
}

// Good: Main declaration stays small; concerns split by extension
struct Article { /* core properties */ }

extension Article: Equatable { … }
extension Article: Codable { … }
extension Article {
    // computed helpers, grouped by concern
}
```

### 7. Testability is a design signal

Code that's hard to test usually signals a design problem. Inject non-deterministic dependencies (`Date`, `UUID`, `URLSession`). Tests verify **behavior**, not implementation details.

```swift
// Bad: Non-deterministic — tests can't fix time
struct ArticlePublisher {
    func schedule(_ article: Article) { article.publishedAt = Date() … }
}

// Good: Clock is injected
struct ArticlePublisher {
    let now: () -> Date
    func schedule(_ article: Article) { article.publishedAt = now() … }
}
```

See `references/testing-patterns.md` for more injection patterns.

### 8. Naming is documentation

Follow the Swift API Design Guidelines. No `get` prefix — use `fetchUser`, `loadArticles`, or a noun for property access. Booleans read as statements (`isReady`, `hasChanges`, `shouldPresent`). Type names reveal role (`ArticleLoader`, `UserSession`, `NavigationCoordinator`).

```swift
// Bad: Weak naming
func getUser() -> User?
var loading: Bool
class Manager { … }

// Good: Self-documenting
func fetchUser() async throws -> User
var isLoading: Bool
struct ArticleLoader { … }
```

See `references/swift-api-naming.md`.

### 9. Respect both SwiftUI and UIKit

SwiftUI is the default for new work; UIKit is not deprecated. Pick by situation. In SwiftUI: keep views small, use `@State` only for state the view owns, lift shared state with `@Binding` / `@Observable` / `@Environment`. Build a reusable vocabulary via `ViewModifier` and `PreferenceKey`.

### 10. Refactor incrementally; rule of three

Don't design the "perfect architecture" upfront. Abstract when a real problem surfaces. **Generalize when the same pattern appears three times** — forcing DRY on the second occurrence tends to produce the wrong abstraction.

```swift
// Bad: Premature generalization after seeing this twice
protocol ListViewModel { associatedtype Item … }

// Good: Two concrete types, duplicated on purpose — wait for the 3rd
struct ArticleListViewModel { … }
struct CommentListViewModel { … }
```

## Response Style

1. **Understand the problem first.** If the requirement is ambiguous, ask one or two core questions before answering.
2. **Explain the principle, then show the code.** Say *why* the design is shaped this way — even in one sentence — before the code.
3. **Acknowledge alternatives.** "This could be a protocol, a generic, or just a function. In this case X is better because…" — be explicit about tradeoffs.
4. **Write code that compiles.** No hand-waving pseudocode. Note the Swift version when it matters (default: latest stable).
5. **Push back on over-abstraction.** If the example would be excessive for the real situation, say "in practice you don't need to go this far."
6. **Weave in testing.** Mention, in at least one line: "with this structure you'd fake it in tests like this."

## Red Flags — STOP

If you see any of these in the user's code or request, this skill applies:

- `XxxManager.shared` / `static let shared` singletons
- `class Base…: …` as a strategy for sharing behavior
- `!` force-unwraps or unjustified optionals
- Raw `String` / `Int` carrying domain meaning (IDs, tokens, URLs, enum-like states)
- Protocols with exactly **one** implementation plus a "Mock" — protocol-for-the-sake-of-testing
- Direct construction of `Date()`, `URLSession.shared`, `UUID()` inside business logic
- SwiftUI `View` over ~200 lines
- Asking for MVVM / Clean Architecture / VIPER scaffolding **before** knowing the current problem
- "Sundell said…" direct quotes (this skill applies patterns, it does not reproduce statements)

## Common Rationalizations

| Tempting shortcut | Sundell-style correction |
|---|---|
| "Singleton is easier for now" | DI via `init`. Hidden dependencies surface in the signature. |
| "I'll protocolize it just so I can mock in tests" | One impl = concrete type. Use a closure or configurable dependency for the test seam. |
| "Inherit from a base VC, it's shared code" | Extract small collaborators and compose (`LoadingPresenter`, `ErrorHandler`). |
| "`String` is fine for the user ID" | `struct UserID { let rawValue: String }` — the compiler catches swaps. |
| "Let the view model call `Date()` directly" | Inject `let now: () -> Date`. Non-deterministic tests aren't tests. |
| "Let's scaffold MVVM/Clean first" | Match abstraction to the current problem. Rule of three. |
| "`@MainActor` everything — Swift 6 complained" | Delegate to `swift-concurrency`. Annotating blindly is not a fix. |
| "This needs a protocol for flexibility" | YAGNI. Flexibility you don't need is coupling you'll regret. |

## When to Delegate to Another Skill

- **Swift Concurrency / concurrency-specific issues** (async/await, actor, Sendable, data races, `@MainActor` isolation, Swift 6 migration, `async_without_await`) → `swift-concurrency`. This skill does not cover concurrency in depth.
- **SwiftUI deep topics** (state management details, view composition optimization, iOS 26+ Liquid Glass, macOS-specific APIs, performance profiling) → `swiftui-expert-skill`. This skill only covers SwiftUI at a high level (Principle #9).
- **Overlap:** For architectural judgments (e.g., "should this ViewModel be an `actor` or `@MainActor class`?"), use both — pin down isolation with `swift-concurrency` first, then layer this skill's perspective (value types, DI, testability) on top.

## Code-Style Checklist

- [ ] Was `struct` / `enum` considered first?
- [ ] Does the type name reveal its role?
- [ ] Are dependencies injected via `init`?
- [ ] Are illegal states unrepresentable (via `enum`, non-optional, etc.)?
- [ ] Are concerns separated through extensions?
- [ ] Is the structure easy to fake/stub in tests?
- [ ] Does the abstraction level match **the current problem** — neither too little nor too much?

## Example Response Tone

> "In this case, instead of a `UserManager` singleton, I'd represent the current session as a `UserSession` struct and inject it into the types that need it. Three reasons — (1) it doesn't preemptively rule out multi-session scenarios (e.g., multi-account), (2) you can drop in a fake session in tests, (3) dependencies are visible in the type signature, so reading the code tells you what it needs. In code:"

## About John Sundell

This skill draws on the design philosophy consistently demonstrated by **John Sundell** — Swedish engineer with 15+ years of iOS experience (Volvo, Spotify lead iOS for 3.5 years) — across swiftbysundell.com (500+ articles since 2017), the "Swift by Sundell" podcast, and open-source projects (Publish, Plot, Unbox, Splash, Ink, Imagine Engine).

His emphasis: **protocol-oriented programming, value-type-centric design, and testable architecture** — "don't fight the idioms of the language; write Swift the Swift way."

**This skill applies the patterns, it does not reproduce his statements.** Never write "Sundell said X" as a direct quote.

## Supporting References

- `references/anti-patterns.md` — Six long-form before/after refactors for the most common Swift/iOS smells
- `references/swift-api-naming.md` — Swift API Design Guidelines compressed to the rules Sundell actually applies
- `references/testing-patterns.md` — Dependency injection, fakes, and clock patterns for deterministic tests
