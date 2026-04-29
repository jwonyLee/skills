# The Code Quadrant: Where to Focus Testing Effort

Not all code deserves the same testing attention. Khorikov's framework places every type into one of four quadrants based on two axes:

- **Domain Significance** — does this code encode business rules that matter? Or is it plumbing?
- **Code Complexity** — does it contain branching, conditions, calculations? Or is it a straight pipe?

The combination determines what kind of test (if any) gives the highest value.

## The Matrix

|                              | **Low Complexity / Low Domain**       | **High Complexity / High Domain** |
|------------------------------|---------------------------------------|-----------------------------------|
| **Mostly logic**             | Don't test (trivia)                   | Unit-test heavily (domain model)  |
| **Mostly orchestration**     | One integration test (controller)     | Refactor before testing           |

Each quadrant has a different testing strategy.

---

## Q1 — Domain Model (high domain, high complexity)

This is your **primary unit-test target**. Pure logic with branching, validation, calculations, or state machines, all encoding business rules.

```swift
struct Delivery {
    let scheduledFor: Date
    let address: Address
    let priority: Priority

    func isValid(now: Date) -> ValidationResult {
        guard scheduledFor > now else { return .pastDate }
        guard !address.isEmpty else { return .missingAddress }
        if scheduledFor.timeIntervalSince(now) < 3600 {
            return priority == .express ? .ok : .tooSoon
        }
        return .ok
    }
}
```

**Testing strategy:** unit tests, dozens of them, one per behavior.

```swift
@Test func delivery_with_past_date_is_invalid() {
    let delivery = Delivery.fixture(scheduledFor: pastDate)
    #expect(delivery.isValid(now: now) == .pastDate)
}

@Test func delivery_with_express_priority_allows_short_lead_time() { … }
@Test func delivery_without_address_is_invalid() { … }
// … one test per behavior the type provides
```

This is where the 4-pillar maximum sits — high RP (catches real bugs), high RR (asserts on observable outcomes), high Fast (pure logic, no IO), high Maintainable (small Arrange).

---

## Q2 — Trivia (low domain, low complexity)

Getters, equality, simple delegations — code that has no decision and no business meaning.

```swift
struct User {
    let firstName: String
    let lastName: String

    var fullName: String { firstName + " " + lastName }
}

struct Article {
    enum Status { case draft, published, archived }
    let status: Status

    var isPublished: Bool { status == .published }
}
```

**Testing strategy:** don't test. The test's value is **negative** — it can't detect a bug class (RP = 0), it breaks on benign refactors (RR low), and it adds maintenance weight to the suite.

If a coverage tool flags this code as uncovered, the right response is:

1. Confirm there's no business rule hiding in the property (locale ordering? truncation? archived-also-counts?)
2. Exclude it from coverage measurement, or accept the gap
3. Move on to higher-value behaviors

A coverage requirement does **not** turn Q2 into Q1.

---

## Q3 — Controllers (low complexity, high domain coordination)

Code that *coordinates* between the domain model and external systems but contains no decision logic itself. Thin pass-through:

```swift
final class CreateDeliveryController {
    let deliveries: DeliveryStore
    let notifier: PushNotifier

    func create(_ request: CreateRequest) async throws -> Delivery {
        let delivery = Delivery(
            scheduledFor: request.date,
            address: request.address,
            priority: request.priority
        )
        try await deliveries.save(delivery)
        try await notifier.notify(.deliveryScheduled(delivery))
        return delivery
    }
}
```

The controller has high domain significance (it's how the business creates deliveries) but low complexity (no branches, no calculations). Unit-testing it gives low RP — the bugs that hide here are integration bugs (DB constraint violations, notifier failures, ordering), not logic bugs.

**Testing strategy:** integration tests over the longest happy path. One test that uses real `DeliveryStore` (or in-memory variant) and a real notifier (or mock for unmanaged ones). Edge cases of `Delivery.isValid` go to unit tests on `Delivery` itself.

```swift
@Test func create_persists_delivery_and_sends_notification() async throws {
    let db = TestDatabase.fresh()
    let pushClient = MockPushClient()  // unmanaged → mock is correct here
    let controller = CreateDeliveryController(
        deliveries: PostgresDeliveryStore(db: db),
        notifier: PushNotifier(client: pushClient)
    )

    let delivery = try await controller.create(.validRequest)

    let stored = try await db.queryFirst("SELECT id FROM deliveries WHERE id = $1", [delivery.id])
    #expect(stored != nil)
    #expect(pushClient.lastSent == .deliveryScheduled(delivery))
}
```

One integration test covers the wiring; the domain-model unit tests (Q1) cover all the rules. Don't unit-test the controller separately — there's nothing there to verify that isn't already covered better elsewhere.

---

## Q4 — Overcomplicated Code (high complexity, low domain)

The fourth quadrant is a smell, not a target. High complexity + low domain significance means you've built a complicated mechanism around something that doesn't matter — usually a side effect of mixing IO with logic, or a misplaced abstraction.

```swift
final class FilesystemBackedRetryingHTTPClient {
    // 200 lines: retry logic, file-based caching, exponential backoff,
    // request signing, header transformation, JSON envelope unwrapping…
    // None of it encodes the business; all of it is plumbing.
}
```

**Testing strategy:** refactor first. Pull the genuine logic (retry policy, backoff curve) into a pure type that lives in Q1; collapse the rest into a thin shell that lives in Q3. Then test each in their natural quadrant.

If you write unit tests for the current shape, you'll get fragile, slow, hard-to-read tests that score poorly on every pillar. The cost-benefit only works after the refactor.

```swift
// After the refactor:

// Q1 — pure logic, unit-tested heavily
struct RetryPolicy {
    let maxAttempts: Int
    let backoff: (Int) -> TimeInterval
    func nextDelay(after attempt: Int) -> TimeInterval? { … }
}

// Q3 — thin shell, integration-tested once
final class HTTPClient {
    let session: URLSession
    let policy: RetryPolicy
    func send(_ request: URLRequest) async throws -> Data { … }
}
```

---

## Decision Flow

When triaging a type, walk this top to bottom:

1. **Does it encode business rules with branching or calculation?** → Q1 (domain model). Unit-test heavily.
2. **Is it a getter, simple delegation, or trivial mapping?** → Q2 (trivia). Don't test.
3. **Does it orchestrate other types but contain no decision logic?** → Q3 (controller). Integration-test the happy path; let the domain types handle their own unit coverage.
4. **Is it complex but doesn't carry business meaning?** → Q4 (overcomplicated). Refactor before testing.

The goal: most of your suite (by count) lives in Q1. A handful of Q3 integration tests cover the wiring. Q2 and Q4 should be empty of tests — Q2 because there's nothing to gain, Q4 because the testing payoff happens after the refactor.

---

## Swift Examples per Quadrant

| Quadrant | Example types in a typical iOS app |
|---|---|
| **Q1 (Domain)** | `Order`, `Delivery`, `Pricing`, `Validator`, `StateMachine`, `ScoringRules`, `RetryPolicy` |
| **Q2 (Trivia)** | `User.fullName`, `Article.isPublished`, simple Equatable conformances, simple Codable types, plain DTOs |
| **Q3 (Controller)** | `CreateOrderController`, `LoadProfileViewModel` (the orchestration only), `BootstrapCoordinator` |
| **Q4 (Overcomplicated)** | `RetryingCachingHTTPClient`, mega-`Manager` types, anything with `// TODO: refactor` at the top |

---

## A Note on Coverage

A coverage tool reports a number for **each line of code**. The quadrant model says only **Q1 lines** should drive that number. A team that aims for "95 % coverage on domain models, controllers excluded" ships sound tests; a team that aims for "80 % coverage everywhere" ends up with theatre on Q2 and fragility on Q3 / Q4.

If you can split coverage measurement by target — domain models high, controllers via integration tests, plumbing excluded — that's the configuration to chase. The single global percentage is misleading and incentivises Q2 trivia.
