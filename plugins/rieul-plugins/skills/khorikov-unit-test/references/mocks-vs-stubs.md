# Mocks vs Stubs

The single most-misunderstood concept in unit testing — and the source of most overspecified, fragile suites.

## The Definitions

A **test double** stands in for a real collaborator. It has exactly one of two roles, never both:

- **Stub** — replaces an *incoming* dependency. Provides data the SUT consumes. The test never asserts on it.
- **Mock** — verifies an *outgoing* communication. Records side effects the SUT produces. The test asserts on its records.

| Role | Direction | Used for | Test asserts on it? |
|---|---|---|---|
| **Stub** | Incoming | Data flowing INTO the SUT (read from DB, fetched from API, supplied by user) | **Never** |
| **Mock** | Outgoing | Side effects flowing OUT of the SUT (write to DB, send email, publish event) | Yes — on whether the side effect was produced |

If you find yourself asserting on a stub (`stub.fetchCallCount == 1`), you've conflated the two roles. The test is verifying *implementation details*, not behavior.

---

## Managed vs Unmanaged Dependencies

The second axis. A dependency is:

- **Managed** — the team owns it. The Postgres database your service writes to. The Redis cache the team operates. The filesystem you control.
- **Unmanaged** — out of your control. An external HTTP API. A third-party SDK that calls home. A payment processor.

The test treatment depends on *both* axes (direction × management):

| Dep direction | Managed | Unmanaged |
|---|---|---|
| **Incoming** (data flows in) | Real (in-memory variant) OR stub | Stub |
| **Outgoing** (side effect goes out) | Real, asserted via integration test | Mock |

Two rules collapse most decisions:

1. **Never mock a managed dependency.** If the database is yours, mocking it gives you a test that passes while the SQL is wrong. Use a real DB (or `inMemory()` shape) and verify behavior end-to-end.
2. **Never assert on a stub.** If you stub a loader, you're not allowed to write `#expect(stub.loadCallCount == ...)`. Pick one role per double.

---

## Worked Examples

### Incoming + Managed: real, or stub

```swift
// Incoming: ArticleLoader reads articles. Managed: our own DB.

// Good: real DB, integration scope (preferred for managed deps)
@Test func loader_returns_articles_from_db() async throws {
    let db = TestDatabase.fresh()
    try await db.insert(.fixture)
    let loader = ArticleLoader(db: db)
    let articles = try await loader.load()
    #expect(articles == [Article.fixture])
}

// Acceptable: stub for fast unit-scope verification of a specific behavior
@Test func loader_filters_to_published_only() async throws {
    let stub = StubArticleStorage(stubbed: [.draftFixture, .publishedFixture])
    let loader = ArticleLoader(storage: stub)
    let articles = try await loader.load()
    #expect(articles == [.publishedFixture])
}
```

### Incoming + Unmanaged: always stub

```swift
// Incoming: PriceLoader reads exchange rates. Unmanaged: external FX API.

// Good: stub the external service — fast, deterministic
@Test func price_loader_converts_using_fx_rates() async throws {
    let stub = StubFXService(stubbed: ["USD-EUR": 0.92])
    let loader = PriceLoader(fx: stub)
    let priceEUR = try await loader.priceInEUR(usd: 100)
    #expect(priceEUR == 92)
}

// Bad: real call to the external service makes the test slow + flaky
@Test func price_loader_converts_using_real_api() async throws {
    let loader = PriceLoader(fx: HTTPFXService(session: .shared))
    let priceEUR = try await loader.priceInEUR(usd: 100)
    #expect(priceEUR > 0)  // not even deterministic
}
```

### Outgoing + Managed: real (integration scope)

```swift
// Outgoing: OrderRecorder writes orders. Managed: our own DB.

// Good: real DB, verify state via integration test
@Test func recorder_writes_order_to_db() async throws {
    let db = TestDatabase.fresh()
    let recorder = OrderRecorder(db: db)
    try await recorder.record(.fixture)
    let stored = try await db.queryFirst("SELECT id FROM orders") as? String
    #expect(stored == Order.fixture.id.rawValue)
}

// Bad: mock the DB (managed dependency)
@Test func recorder_writes_order_to_db_BAD() async throws {
    let mockDB = MockDatabase()
    let recorder = OrderRecorder(db: mockDB)
    try await recorder.record(.fixture)
    #expect(mockDB.insertCallCount == 1)
    // Test passes while the SQL is malformed — DB never tried it.
}
```

### Outgoing + Unmanaged: mock

```swift
// Outgoing: NotificationSender posts to Slack. Unmanaged: external Slack webhook.

// Good: mock the external client and verify the outgoing call
@Test func sender_publishes_alert_to_slack() async throws {
    let mock = MockSlackClient()
    let sender = NotificationSender(slack: mock)
    try await sender.alert(.deploymentFailed)
    #expect(mock.lastSent == .deploymentFailed)
}
```

---

## The Anti-Pattern: Asserting on Stubs

The most common Khorikov-violation in real codebases:

```swift
// Bad: stub being treated as a mock — overspecification
@Test func controller_loads_user() async throws {
    let stub = StubUserLoader(stubbed: .fixture)
    let controller = ProfileController(loader: stub)
    _ = try await controller.load(id: .fixture)
    #expect(stub.loadCallCount == 1)  // ← asserting on a stub
}
```

This test fails the moment the controller adds caching, batches loads, or moves the load to a background warming step — even though the behavior (showing the right user) is unchanged.

The fix is to assert on the **outcome**, not the call count:

```swift
// Good: assert on the visible outcome
@Test func controller_displays_loaded_user() async throws {
    let stub = StubUserLoader(stubbed: .fixture)
    let controller = ProfileController(loader: stub)
    try await controller.load(id: .fixture)
    #expect(controller.displayedName == User.fixture.displayName)
}
```

---

## The Anti-Pattern: Mocking the Database

```swift
// Bad: managed dep being mocked
@Test func service_creates_user() async throws {
    let mockDB = MockDatabase()
    let service = UserService(db: mockDB)
    try await service.create(.fixture)
    #expect(mockDB.insertCallCount == 1)
}
```

What this test verifies: *that `service.create` calls `db.insert` once*. What it does NOT verify: that the SQL is correct, the schema matches, the constraints fire, foreign keys resolve. Bugs in any of those layers slip through silently because the mock doesn't run them.

The fix is integration scope with a real database (or its in-memory equivalent):

```swift
// Good: real DB, behavior verified end-to-end
@Test func service_creates_user_in_db() async throws {
    let db = TestDatabase.fresh()
    let service = UserService(db: db)
    try await service.create(.fixture)
    let count = try await db.queryFirst("SELECT COUNT(*) FROM users") as? Int
    #expect(count == 1)
}
```

This test is *integration scope* — slower than a unit test, faster than E2E, and it actually catches SQL bugs.

---

## Decision Table

When deciding what to do with a collaborator, walk this table top to bottom:

| Question | If YES | If NO |
|---|---|---|
| 1. Is this a side effect leaving the SUT? | go to (2) | go to (3) |
| 2. Is the receiver managed? | use the real thing in an integration test | mock; verify the call |
| 3. Is this an incoming source? | go to (4) | (it's pure logic — use the real type) |
| 4. Is the source managed? | use the real thing OR a stub | stub |

Then sanity-check: am I about to assert on a stub? If yes — pick a different assertion, or restructure to verify an outgoing communication instead.

---

## Why "Don't Mock What You Don't Own" Is Not Enough

The classic Sundell-style rule says: don't mock `URLSession`, `FileManager`, etc., because they're Apple-owned. Wrap them in your own thin abstraction and test against the wrapper. (See `john-sundell/references/testing-patterns.md` §6.)

Khorikov goes further: **don't mock what you DO own, either** — when it's an outgoing communication to a managed system. The Postgres DB is yours; mocking it still produces a test that proves nothing about the SQL. The right answer is integration scope.

The combined rule: **mocks only for outgoing communications to unmanaged systems.** Everything else is either real (integration), stubbed (incoming), or unwrapped (Apple SDK → thin wrapper + in-memory variant).

---

## Quick Reference

| You're testing… | Use |
|---|---|
| Pure logic with input data (no IO) | Real collaborators (Classical school) |
| Code that reads from your own DB | Real DB in integration test, OR stub for unit-scope filtering logic |
| Code that reads from an external API | Stub the API |
| Code that writes to your own DB | Real DB in integration test |
| Code that writes to an external API | Mock the API client |
| Code that needs the current time | Inject `() -> Date`, use a fixed time in tests (a stub for an input) |
| Apple SDK (`URLSession`, `FileManager`, `UserDefaults`) | Thin wrapper + in-memory variant; test against the wrapper |
