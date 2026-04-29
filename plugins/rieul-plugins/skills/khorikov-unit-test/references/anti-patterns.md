# Khorikov-style Test Anti-Patterns

Seven Bad/Good refactors, each fixing a violation of one or more pillars.

## 1. Testing Private Methods → Test Through the Public API

**The smell.** A test reaches into private internals via `@testable import` because the public API doesn't expose what the test wants to verify.

```swift
// Bad
@testable import App

@Test func validates_internal_state() {
    let publisher = ArticlePublisher()
    #expect(publisher._validate(article: .fixture) == .ok)
}
```

The desire to test privates is a design alarm. Either:

- The behavior IS public — test it through the actual surface, OR
- The private logic is a separate concept that wants extracting into a real type with its own public API

**The refactor.** Extract the validator into its own type:

```swift
// Good
struct ArticleValidator {
    func validate(_ article: Article) -> ValidationResult {
        guard !article.title.isEmpty else { return .empty_title }
        // …
        return .ok
    }
}

@Test func validator_rejects_empty_title() {
    let result = ArticleValidator().validate(.empty)
    #expect(result == .empty_title)
}
```

Now `ArticlePublisher` uses `ArticleValidator` as a real collaborator (Classical school — no mock needed), and the validation logic is testable through its own public API.

---

## 2. `Date()` / `UUID()` Inline → Injected Seam

**The smell.** Production code constructs non-deterministic values inline, making tests depend on real time / randomness.

```swift
// Bad
struct ArticleScheduler {
    func nextSlot() -> Date {
        Calendar.current.date(byAdding: .hour, value: 1, to: Date())!
    }
}
```

Tests have to either accept flake (`#expect(scheduler.nextSlot() > Date())`) or smuggle in their own time via static overrides.

**The refactor.** Inject the seam at `init`:

```swift
// Good
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

Same pattern for `UUID`, `Random`, anything non-deterministic. See `john-sundell/references/testing-patterns.md` §1 for the canonical Swift shape.

---

## 3. Asserting on Stubs → Assert on Outcome

**The smell.** A test stubs an input dependency, then asserts on the stub's call count.

```swift
// Bad
@Test func controller_loads_user() async throws {
    let stub = StubUserLoader(stubbed: .fixture)
    let controller = ProfileController(loader: stub)
    _ = try await controller.load(id: .fixture)
    #expect(stub.loadCallCount == 1)  // ← asserting on a stub
}
```

This breaks the moment the controller adds caching, batches loads, or moves the load to a background warming step — even though the user-visible behavior is unchanged.

**The refactor.** Assert on what the user actually sees:

```swift
// Good
@Test func controller_displays_loaded_user() async throws {
    let stub = StubUserLoader(stubbed: .fixture)
    let controller = ProfileController(loader: stub)
    try await controller.load(id: .fixture)
    #expect(controller.displayedName == User.fixture.displayName)
}
```

Spy on call counts only when the side effect *is* the observable behavior (e.g., "this MUST hit the analytics endpoint exactly once" — analytics is unmanaged-outgoing, the perfect mock target).

---

## 4. Production `#if TEST` Switches → Inject Behavior

**The smell.** Production code carries test-only switches, static overrides, or conditionally-compiled code paths.

```swift
// Bad
struct ArticleScheduler {
    static var fixedNow: Date?

    func nextSlot() -> Date {
        let now = Self.fixedNow ?? Date()
        return Calendar.current.date(byAdding: .hour, value: 1, to: now)!
    }
}

// In tests:
ArticleScheduler.fixedNow = Date(timeIntervalSince1970: 1_700_000_000)
defer { ArticleScheduler.fixedNow = nil }  // every test must remember to clear it
```

Three problems: production code carries weight that exists only for tests; tests share global state; and forgetting the cleanup poisons sibling tests.

**The refactor.** Same as anti-pattern #2 — inject at `init`. Production code stays pure; tests pass the seam they need. No global state, no shared overrides, no cleanup ritual.

---

## 5. Mocking the Database → Use a Real DB in Integration Tests

**The smell.** Tests mock a managed dependency (your own database, your own filesystem) and assert on call counts.

```swift
// Bad
@Test func service_creates_user() async throws {
    let mockDB = MockDatabase()
    let service = UserService(db: mockDB)
    try await service.create(.fixture)
    #expect(mockDB.insertCallCount == 1)
}
```

This test verifies *that the service called insert once*. It does NOT verify: that the SQL is correct, the schema matches, the constraints fire, foreign keys resolve. Bugs in any of those layers slip through silently.

**The refactor.** Use a real database (or in-memory equivalent) at integration scope:

```swift
// Good
@Test func service_creates_user_in_db() async throws {
    let db = TestDatabase.fresh()
    let service = UserService(db: db)
    try await service.create(.fixture)
    let count = try await db.queryFirst("SELECT COUNT(*) FROM users") as? Int
    #expect(count == 1)
}
```

The test is slower than a unit test, faster than E2E, and it actually catches SQL bugs. Reserve mocks for **unmanaged** dependencies (external HTTP APIs, third-party SDKs, payment processors).

---

## 6. Mega-Test with Multiple Acts → One Test per Behavior

**The smell.** A single test exercises a multi-step workflow with several Act-and-Assert pairs.

```swift
// Bad
@Test func editor_workflow() {
    let editor = makeEditor()

    editor.set(title: "A");  #expect(editor.title == "A")
    editor.save();           #expect(editor.isSaved)
    editor.set(body: "B");   #expect(!editor.isSaved)
    editor.save();           #expect(editor.isSaved)
}
```

Three problems: the first failure hides the others; a benign change to step 2 breaks "the workflow test" without telling you which behavior actually regressed; the test name doesn't describe a single thing.

**The refactor.** One test per observable behavior, sharing a fixture for the Arrange:

```swift
// Good
private func makeEditor() -> Editor { … }

@Test func setting_title_updates_title() {
    let editor = makeEditor()
    editor.set(title: "A")
    #expect(editor.title == "A")
}

@Test func saving_marks_clean() {
    let editor = makeEditor().with(title: "A")
    editor.save()
    #expect(editor.isSaved)
}

@Test func editing_after_save_marks_dirty() {
    let editor = makeEditor().with(title: "A").saved()
    editor.set(body: "B")
    #expect(!editor.isSaved)
}
```

Each test names the behavior, fails for one specific reason, and reads as a spec.

---

## 7. `Method_Scenario_Result` → Natural-Language Test Names

**The smell.** Test names follow a mechanical template that adds zero information when applied uniformly across a suite.

```swift
// Bad
@Test func IsValid_PastDate_ReturnsFalse() { … }
@Test func IsValid_FutureDate_ReturnsTrue() { … }
@Test func Save_NetworkError_ThrowsError() { … }
```

When every test in 12 000 tests starts with `IsValid_` or `Save_` or `Load_`, the prefix becomes visual noise. The only signal is the middle and end — which natural-language names express more clearly anyway.

**The refactor.** Names that read as specifications:

```swift
// Good
@Test func delivery_with_past_date_is_invalid() { … }
@Test func delivery_with_future_date_is_valid() { … }
@Test func save_throws_when_network_fails() { … }
```

The test name reads like a sentence in the requirements doc, which is what a test name *is*.

**If the team mandates `Method_Scenario_Result`** — surface the tradeoff in one sentence ("natural-language reads as a spec; the template adds no information when applied uniformly across the suite — but I'll match if you insist"), then comply only if the user explicitly reaffirms. Authority + sunk-cost ("12 000 tests already use it") is the loudest signal that pushback matters here, not a reason to suppress it.

---

## Quick Reference

| Anti-pattern | Fix |
|---|---|
| Testing private methods | Extract the logic into a real type with its own public API |
| `Date()` / `UUID()` inline | Inject `() -> Date` / `() -> UUID` at `init` |
| Asserting on stubs (`callCount == N`) | Assert on the outcome the user sees |
| `#if TEST` / `static var override` in production | Inject the dependency at `init` |
| Mocking the database / filesystem | Real DB or in-memory variant in integration test |
| Mega-test with multiple acts | One test per behavior, fixture for shared Arrange |
| `Method_Scenario_Result` mechanical names | Natural-language sentences that read as specs |
