# Testing Patterns

Patterns Sundell-style code uses to stay testable without a mocking framework.

> For the principled framework behind these patterns — the four pillars of test value, mocks vs stubs, managed vs unmanaged dependencies, and the code quadrant — see `rieul-plugins:khorikov-unit-test`.

## The Core Rule

**If it's hard to test, the design is the problem — not the test framework.**

Non-deterministic dependencies (time, randomness, network, filesystem) should be *injected*, not constructed inline. This file shows the specific shapes that keep tests fast, deterministic, and readable.

---

## 1. Inject the Clock

**Problem.** `Date()` inside business logic makes tests depend on wall-clock time.

```swift
// Bad: Non-deterministic
struct ArticleScheduler {
    func nextSlot() -> Date {
        Calendar.current.date(byAdding: .hour, value: 1, to: Date())!
    }
}

// Good: Clock injected as a function
struct ArticleScheduler {
    let now: () -> Date

    func nextSlot() -> Date {
        Calendar.current.date(byAdding: .hour, value: 1, to: now())!
    }
}

// Production
let scheduler = ArticleScheduler(now: Date.init)

// Tests
let fixed = Date(timeIntervalSince1970: 1_700_000_000)
let scheduler = ArticleScheduler(now: { fixed })
#expect(scheduler.nextSlot() == fixed.addingTimeInterval(3600))
```

`() -> Date` is lighter than a `Clock` protocol and sufficient for most cases. Reach for Swift's `Clock` when you need sleeping, cancellation, or scheduling.

## 2. Networking as a Protocol — When You Have Two Real Impls

**Problem.** Tests need canned responses. `URLSession` is awkward to stub.

**Rule of thumb.** Introduce a `Networking` protocol only if you have (or plan to have) 2+ real implementations — e.g., a real network + an offline/cached path. If the only "other impl" is a test double, see §3.

```swift
protocol Networking {
    func fetch<Response: Decodable>(_ endpoint: Endpoint<Response>) async throws -> Response
}

struct URLSessionNetworking: Networking {
    let session: URLSession
    let decoder: JSONDecoder

    func fetch<Response: Decodable>(_ endpoint: Endpoint<Response>) async throws -> Response {
        let (data, _) = try await session.data(for: endpoint.request)
        return try decoder.decode(Response.self, from: data)
    }
}

struct OfflineNetworking: Networking {
    let cache: ResponseCache

    func fetch<Response: Decodable>(_ endpoint: Endpoint<Response>) async throws -> Response {
        guard let data = cache.data(for: endpoint) else { throw NetworkError.offlineMiss }
        return try JSONDecoder().decode(Response.self, from: data)
    }
}

// Tests — a fake that doesn't need to be a class
struct FakeNetworking: Networking {
    var stubs: [AnyHashable: Any]

    func fetch<Response: Decodable>(_ endpoint: Endpoint<Response>) async throws -> Response {
        guard let value = stubs[endpoint.key] as? Response else {
            throw NetworkError.notStubbed(endpoint.key)
        }
        return value
    }
}
```

## 3. When You Don't Need a Protocol — Configurable Type

**Problem.** One real `UserLoader`, and a protocol only existed to make a `MockUserLoader`.

**Fix.** Replace the protocol with a configurable struct holding closures. One type, two constructors.

```swift
struct UserLoader {
    let load: (UserID) async throws -> User
    let refresh: (UserID) async throws -> User
}

extension UserLoader {
    static func live(networking: Networking) -> Self {
        .init(
            load:    { id in try await networking.fetch(.user(id)) },
            refresh: { id in try await networking.fetch(.userForceRefresh(id)) }
        )
    }
}

// Tests
let loader = UserLoader(
    load:    { _ in User.fixture },
    refresh: { _ in User.fixture.withUpdatedName("Fresh") }
)
```

This is the *Point-Free dependencies style*. One type, two constructors. No `UserLoaderProtocol`, no `MockUserLoader`.

## 4. Test Fixtures as Static Extensions

**Problem.** Tests construct model objects ad-hoc. Duplication, drift, noise in the test body.

```swift
extension User {
    static let fixture = User(
        id: UserID(rawValue: "u_1"),
        displayName: "Jane",
        email: "jane@example.com"
    )

    static func fixture(id: String = "u_1", name: String = "Jane") -> User {
        User(id: UserID(rawValue: id), displayName: name, email: "\(id)@example.com")
    }
}

// Tests
let user = User.fixture
let bob = User.fixture(name: "Bob")
```

**Why inside an `#if DEBUG`-gated extension or in a test target:** you don't want fixtures in the production binary.

## 5. Test Behavior, Not Implementation

**Problem.** Tests assert on how a function did its work, not on the observable outcome. They break the moment the implementation shifts — and that's failure, not coverage.

```swift
// Bad: Couples to the networking call sequence
func test_load_callsNetworkingOnce() async throws {
    let networking = SpyNetworking()
    let loader = ArticleLoader.live(networking: networking)
    _ = try await loader.load()
    #expect(networking.fetchCallCount == 1)
}

// Good: Asserts the outcome
func test_load_returnsDecodedArticles() async throws {
    let networking = FakeNetworking(stubs: [.articles: [Article.fixture]])
    let loader = ArticleLoader.live(networking: networking)
    let articles = try await loader.load()
    #expect(articles == [Article.fixture])
}
```

Spy on call counts only when the side effect *is* the observable behavior (e.g., "this must hit the analytics endpoint exactly once").

## 6. Don't Mock What You Don't Own

`URLSession`, `FileManager`, `UserDefaults`, `Calendar` — these are large surfaces owned by Apple. Mocking them directly ties your tests to Apple's implementation details.

**Pattern.** Put a thin wrapper (with the surface your code actually uses) between you and the platform, and test against the wrapper.

```swift
// Your own thin abstraction — the only API your code uses
struct ArticleStorage {
    let read: (UserID) throws -> [Article]
    let write: (UserID, [Article]) throws -> Void
}

extension ArticleStorage {
    static func live(directory: URL) -> Self {
        .init(
            read: { id in
                let url = directory.appending(path: "\(id.rawValue).json")
                let data = try Data(contentsOf: url)
                return try JSONDecoder().decode([Article].self, from: data)
            },
            write: { id, articles in
                let url = directory.appending(path: "\(id.rawValue).json")
                let data = try JSONEncoder().encode(articles)
                try data.write(to: url)
            }
        )
    }

    static func inMemory() -> Self {
        var store: [UserID: [Article]] = [:]
        return .init(
            read: { id in store[id] ?? [] },
            write: { id, articles in store[id] = articles }
        )
    }
}
```

Now tests use `ArticleStorage.inMemory()` and never touch the filesystem.

## 7. One Assertion Per Behavior

Long tests that assert twelve things fail opaquely — you fix one, the next one fails, and you learn about the regressions one at a time.

Small, focused tests tell you exactly what broke.

```swift
// Bad:
func test_editor_workflow() async throws {
    let editor = makeEditor()
    editor.set(title: "A"); #expect(editor.title == "A")
    editor.set(body: "B");  #expect(editor.body == "B")
    editor.save();          #expect(editor.isSaved)
    editor.set(title: "C"); #expect(!editor.isSaved)
}

// Good:
func test_setTitle_updatesTitle() { … }
func test_setBody_updatesBody()   { … }
func test_save_marksSaved()        { … }
func test_editAfterSave_marksDirty() { … }
```

Sundell's own test files follow this pattern — each test's name reads as a specification.

## Quick Reference

| You're testing… | Use |
|---|---|
| Time-dependent behavior | Injected `() -> Date` |
| Network calls with 2+ real strategies | `Networking` protocol + `URLSessionNetworking` / `OfflineNetworking` / `FakeNetworking` |
| A type with one real impl that was "protocolized for testing" | Configurable struct with injected closures |
| Model construction noise | `extension User { static let fixture = … }` |
| Filesystem, `UserDefaults`, Apple SDKs | Your own thin wrapper + in-memory version |
| A workflow (`save` → `isSaved`) | One test per observable behavior, not one mega-test |
