# Swift API Naming — The Rules Sundell Actually Applies

Swift has the [API Design Guidelines](https://www.swift.org/documentation/api-design-guidelines/). This file compresses them to the rules that consistently show up in Sundell's writing and open source.

## The One-Line Test

**Read the call site out loud. Does it read like English?**

```swift
articles.append(new)                              // Good: "append new"
articles.insert(new, at: 0)                       // Good: "insert new at 0"
articles.insertion(of: new, at: 0)                // Bad: awkward
```

If it doesn't read naturally, the name is wrong — not the grammar rules.

## Methods and Functions

### Use verbs for side effects, nouns for values

```swift
// Side-effectful — verb phrase
func load()
func save(_ article: Article)
func fetchArticles() async throws -> [Article]

// Returns a value, no side effect — noun phrase
var wordCount: Int
func sorted() -> [Article]
```

### No `get` prefix

```swift
// Bad:
func getUser() -> User?
func getArticles() async throws -> [Article]

// Good:
var user: User?                     // if it's a stored/computed property
func fetchUser() async throws -> User
func loadArticles() async throws -> [Article]
```

### Common verbs and what they imply

| Verb prefix | Meaning |
|---|---|
| `fetch…` / `load…` | Async retrieval from network/disk |
| `make…` | Factory method returning a new value (`makeRequest()`, `makeDateFormatter()`) |
| `resolve…` | Look up or decide, possibly with failure |
| `configure…` | Mutate passed-in target to a ready state |
| `render…` | Produce output (HTML, image, string) from input |
| `decode…` / `encode…` | Format conversion |

### Argument labels make call sites read naturally

```swift
// Bad: Label repeats the type name
func move(toPosition position: CGPoint)

// Good: Label describes the role
func move(to position: CGPoint)

// Bad: First argument label dropped where a preposition would help
func transition(ArticleDetailView, duration: TimeInterval)

// Good:
func transition(to view: ArticleDetailView, duration: TimeInterval)
```

## Booleans

Boolean names read as statements (assertions).

```swift
// Bad:
var loading: Bool
var error: Bool
var valid: Bool

// Good:
var isLoading: Bool
var hasError: Bool
var isValid: Bool
var shouldRefresh: Bool
var canEdit: Bool
```

| Prefix | Use for |
|---|---|
| `is…` | State: `isLoading`, `isExpanded`, `isEmpty` |
| `has…` | Possession: `hasChanges`, `hasMoreResults` |
| `should…` | Policy/intent: `shouldAnimate`, `shouldRefresh` |
| `can…` | Capability: `canEdit`, `canUndo` |
| `did…` / `will…` | Lifecycle events (delegates): `didSelectRow`, `willAppear` |

## Types

### Type names reveal role, not just kind

```swift
// Bad: Generic nouns that could mean anything
class Manager
class Helper
class Utility

// Good: Role + subject
struct ArticleLoader
struct UserSession
final class NavigationCoordinator
struct MarkdownParser
```

The pattern is usually `[Subject][Role]` — `ArticleLoader`, `ImageCache`, `URLSessionNetworking`.

### Avoid stuttering

```swift
// Bad: "Article" appears twice
struct Article {
    func articleSummary() -> String
}

// Good:
struct Article {
    func summary() -> String
}
```

### Protocols

Two common naming shapes:

- **Capability protocols** — end in `-able`, `-ible`, or describe the capability: `Equatable`, `Hashable`, `Caching`, `Loading`.
- **Role protocols** — use the role noun: `Networking`, `ArticleStore`, `ImageFetcher`.

Don't suffix every protocol with `Protocol`. `UserLoaderProtocol` is a code smell that usually means "I made a protocol just for mocking" — see `anti-patterns.md §3`.

## Extensions

### One extension, one concern

```swift
// Bad: Everything in the main declaration
struct Article {
    // stored properties
    // Equatable, Hashable, Codable conformances
    // computed helpers
    // formatting helpers
    // 400 lines
}

// Good: Split by concern
struct Article { /* core stored properties */ }

extension Article: Equatable { /* … */ }
extension Article: Hashable { /* … */ }
extension Article: Codable { /* … */ }
extension Article { /* computed helpers */ }
extension Article { /* formatting helpers */ }
```

### File naming for extensions

The commonly-seen conventions:

- `Article.swift` — the type declaration
- `Article+Equatable.swift` — one conformance per file when it's non-trivial
- `Article+Formatting.swift` — grouped helpers by concern
- `Array+Chunking.swift` — extending a standard-library type with a named capability

Don't over-split. A 5-line `Equatable` conformance can live in `Article.swift`. Split when a file gets hard to scan.

## Generics

Parameter names hint at the role:

```swift
// Bad: Single-letter generic with no constraint hint
func cache<T>(_ value: T, key: String)

// Good: Role-revealing name
func cache<Value>(_ value: Value, key: String)

// Good: When the constraint already names the role
func cache<Value: Codable>(_ value: Value, key: String)
```

Use `T` / `U` / `V` only when the role is truly generic and obvious.

## Enum Cases

### Nouns or noun phrases, lowercase

```swift
enum LoadState {
    case idle
    case loading
    case loaded(articles: [Article])
    case failed(Error)
}
```

### Associated value labels

Omit the label when redundant, keep it when clarifying:

```swift
// Bad: Redundant
case failed(error: Error)

// Good: Type is self-explanatory
case failed(Error)

// Good: Multiple values of the same type — labels disambiguate
case range(from: Int, to: Int)
```

## Closures and Callbacks

### Prefer `() -> Return` over delegate protocols for one-shot callbacks

```swift
// Bad: Delegate protocol for a single callback
protocol ArticleLoaderDelegate: AnyObject {
    func articleLoaderDidFinish(_ loader: ArticleLoader, articles: [Article])
}

// Good: A closure
struct ArticleLoader {
    var onComplete: ([Article]) -> Void
}
```

Delegate protocols make sense for **multiple** correlated callbacks (`UITableViewDataSource`). For one callback, a closure is lighter and more ergonomic.

## The Shortest Rule

> If a reviewer has to ask what a name *does*, the name is wrong.

That is the whole point of the guidelines. Everything above is examples of applying that one rule.
