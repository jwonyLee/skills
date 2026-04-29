# The Four Pillars of a Good Unit Test

A test's value is determined by **four properties**, but they don't add — they multiply. Maximizing all four at once is impossible; minimizing any single one zeros the test.

## The Formula

```
Test Value = (Regression Protection × Refactor Resistance) × (Fast × Maintainable)
```

The first two are the *signal* the test produces. The second two are its *cost*. Treat them as orthogonal: a test can max one and tank another, and the multiplicative form means the worst pillar dominates.

---

## 1. Regression Protection

**How likely is this test to catch a real bug introduced by a change?**

A test scores high when:
- It exercises a non-trivial amount of production code
- The code path it covers is reachable from real user behavior
- A genuine bug in that path would cause an assertion to fail

A test scores low when:
- The code under test is trivial (the assertion restates the implementation)
- The path is unreachable in production
- The bug class is one the test couldn't detect (e.g., `count == 1` won't catch a bad calculation inside the call)

```swift
// Low RP: restates the implementation — no bug class to detect
@Test func fullName_concatenates_first_and_last() {
    let user = User(firstName: "Jane", lastName: "Doe")
    #expect(user.fullName == "Jane Doe")
}

// High RP: a real bug class (locale ordering) is detectable
@Test func fullName_uses_family_name_first_in_zh_locale() {
    let user = User(firstName: "Jane", lastName: "Doe", locale: .zh_CN)
    #expect(user.fullName == "Doe Jane")
}
```

---

## 2. Refactor Resistance

**How often does this test fail when behavior is unchanged?**

A test scores high when:
- It asserts on observable outcomes (return values, public state, externally visible side effects)
- It passes through the public API
- It uses real collaborators where they're managed

A test scores low when:
- It asserts on implementation details (call counts, internal sequences, private state)
- It uses `@testable import` to reach into privates
- It mocks pure logic that has no out-of-process effect

```swift
// Low RR: breaks if the controller adds a cache layer, even though behavior is fine
@Test func controller_calls_loader_exactly_once() async throws {
    let stub = StubUserLoader(stubbed: .fixture)
    let controller = ProfileController(loader: stub)
    _ = try await controller.load(id: .fixture)
    #expect(stub.loadCallCount == 1)
}

// High RR: survives any refactor that preserves the displayed name
@Test func controller_displays_loaded_user() async throws {
    let stub = StubUserLoader(stubbed: .fixture)
    let controller = ProfileController(loader: stub)
    try await controller.load(id: .fixture)
    #expect(controller.displayedName == "Jane Doe")
}
```

**Why these multiply:** if RR is zero, every refactor breaks tests, devs stop trusting the suite, and they delete or skip the failing tests — eliminating any RP that was there. A zero on either pillar destroys the value of both.

---

## 3. Fast Feedback

**How long does the test take to run?**

A test scores high when:
- It runs in milliseconds
- It avoids IO (network, disk, real database)
- It avoids `Thread.sleep`, real timers, or polling

A test scores low when:
- It hits a real network or file system
- It instantiates large object graphs
- It uses real `Date()` and `UUID()` instead of injected seams

```swift
// Slow: real network, real time
@Test func loader_fetches_from_real_endpoint() async throws {
    let loader = ArticleLoader(networking: URLSessionNetworking(session: .shared))
    let articles = try await loader.load()  // real HTTP — seconds
    #expect(!articles.isEmpty)
}

// Fast: in-process stub, deterministic
@Test func loader_returns_decoded_articles() async throws {
    let stub = StubNetworking(stubbed: [.articles: [Article.fixture]])
    let loader = ArticleLoader(networking: stub)
    let articles = try await loader.load()
    #expect(articles == [Article.fixture])
}
```

---

## 4. Maintainability

**How much code do I have to read to understand this test, and how often do I have to update it for unrelated reasons?**

A test scores high when:
- The Arrange is small or factored into a fixture
- The Act is one line
- The Assert is one observable outcome
- Test setup doesn't reach into multiple unrelated systems

A test scores low when:
- The Arrange spans 30 lines of object construction
- The test is one of many that share a brittle giant `setUp`
- Adding a constructor parameter breaks 200 tests

```swift
// Low Maintainability: 25-line Arrange, every test pays the cost
@Test func order_total_with_seasonal_discount() {
    let user = User(id: .init(rawValue: "u_1"), …)
    let address = Address(line1: …, city: …, postal: …, country: …)
    let payment = Payment(method: .card, …, …)
    // … 22 more lines of construction …
    let order = Order(user: user, items: …, address: address, payment: payment, …)
    #expect(order.total == 90)
}

// High Maintainability: fixtures absorb the Arrange
@Test func order_total_with_seasonal_discount() {
    let order = Order.fixture(promotion: .seasonal, items: [item(at: 100)])
    #expect(order.total == 90)
}
```

---

## Why the First Two Multiply

`Regression Protection × Refactor Resistance` form the *signal* of a test.

A test with **zero RR** — that breaks on every refactor — gets disabled or rewritten until it stops breaking, which usually means it stops checking anything real. Zero RR cascades into zero RP.

A test with **zero RP** — that can't catch any bug — has nothing for refactor resistance to protect.

This is why the formula uses multiplication, not addition: a test isn't `RP + RR`, it's `RP × RR`. Either zero zeros the whole thing.

---

## Why Fast and Maintainable Matter (But Differently)

These are the *cost* axis. A high-cost test isn't worthless — but it tilts the cost-benefit. End-to-end tests have very high RP, low Fast, low Maintainable. They're valuable in small numbers as integration coverage; they're disastrous as the bulk of a suite.

Run economics matter:

- 10 000 unit tests × 10 ms = **100 seconds**
- 10 000 E2E tests × 5 s = **14 hours**

The `Fast` pillar is what lets you have *enough* tests for `Regression Protection` to cover the system densely.

---

## Worked Examples

### Example A — End-to-end UI test

```swift
@Test func user_can_complete_checkout() async throws {
    let app = XCUIApplication()
    app.launch()
    app.tabBars.buttons["Cart"].tap()
    app.buttons["Checkout"].tap()
    app.textFields["Card"].typeText("4242…")
    app.buttons["Pay"].tap()
    #expect(app.staticTexts["Order Confirmed"].waitForExistence(timeout: 30))
}
```

| Pillar | Score | Why |
|--------|-------|-----|
| Regression Protection | High | Exercises the full system end to end |
| Refactor Resistance | High | Asserts on user-visible outcome |
| Fast | Low | Tens of seconds per run |
| Maintainable | Low | UI selector churn, flake from real network |

**Verdict:** valuable as one or two tests for the smoke path. Disastrous as the suite's bulk.

### Example B — Trivial getter test

```swift
@Test func fullName_concatenates() {
    #expect(User(firstName: "Jane", lastName: "Doe").fullName == "Jane Doe")
}
```

| Pillar | Score | Why |
|--------|-------|-----|
| Regression Protection | **Zero** | Restates the implementation — no bug class to detect |
| Refactor Resistance | Low | Breaks if `fullName` becomes `"\(first) \(last)"` (no behavior change) |
| Fast | High | Microseconds |
| Maintainable | High | Two lines |

**Verdict:** Test Value = `(0 × low) × (high × high)` = **0**. Don't write it.

### Example C — Behavioral unit test

```swift
@Test func delivery_with_past_date_is_invalid() {
    let delivery = Delivery(date: pastDate, clock: { now })
    #expect(delivery.isValid == false)
}
```

| Pillar | Score | Why |
|--------|-------|-----|
| Regression Protection | High | Catches a real validation bug |
| Refactor Resistance | High | Asserts the public outcome, not internals |
| Fast | High | Pure logic, milliseconds |
| Maintainable | High | Three lines, name reads as a spec |

**Verdict:** the kind of test you want most of the suite to look like.

---

## How to Apply

When triaging a test (yours or someone else's), score it on the four pillars in 30 seconds:

1. Could a real bug make this fail? (RP)
2. Would a benign refactor make this fail? (RR — high RR = no)
3. Does it run fast? (Fast)
4. Is it readable in under 10 seconds? (Maintainable)

If RP or RR is zero, the test should not exist in its current form. Either rewrite it to assert on a different thing, or delete it.

Coverage targets are not an exception — a test with zero value adds zero protection regardless of which percentage it bumps.
