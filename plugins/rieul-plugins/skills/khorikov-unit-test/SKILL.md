---
name: khorikov-unit-test
description: Use when designing, writing, reviewing, or refactoring unit tests in Swift — especially for decisions about what code is worth testing, mocks vs stubs (input vs output dependencies), test isolation strategy, test naming style, integration vs unit scope, evaluating test value, and avoiding test fragility from over-specification.
---

# Khorikov-style Unit Testing for Swift

## Overview

Test value is multiplicative, not additive: `Value = (Regression Protection × Refactor Resistance) × (Fast × Maintainable)`. A zero on any pillar zeros the whole test.

**Core principle:** A unit test verifies a unit of *behavior*, not a unit of *code*. Trivial code, asserting on input dependencies, and rigid naming templates each kill at least one pillar.

**Pragmatism over purity.** This is a framework for *deciding*, not a creed to recite. Cite the underlying principle, not the author.

## When to Use

- Designing new tests — what to test, what doubles to use, what to name them
- Reviewing test PRs for fragility, overspecification, or coverage-driven trivia
- Auditing legacy suites with `MockEverythingService` and call-count assertions
- Choosing between unit vs integration scope for a database / network / file IO concern
- Refactoring tests that break every time the implementation moves an inch
- A user asks "should I mock this?" or "is this test valuable?" or "how do I name this test?"

## When NOT to Use

- Non-Swift codebases — the principles transfer, but the Swift idioms in this skill don't
- Driving the TDD workflow itself (RED/GREEN/REFACTOR cadence) → `superpowers:test-driven-development`
- Swift Testing macro syntax, traits, parameterization, parallel runs → `swift-testing-expert`
- Concurrency-specific test isolation, `@MainActor` or actor reentrance → `swift-concurrency`
- Property-based, snapshot, or UI tests — Khorikov's framework barely covers these

## Quick Reference

| # | Principle | Apply when |
|---|---|---|
| 1 | Test units of *behavior*, not units of code | Asked to "test class X" — pick a behavior instead |
| 2 | Maximize the four pillars (RP × RR × Fast × Maintainable) | Triaging any test, especially legacy ones |
| 3 | Classical (Detroit) school by default | Choosing what to mock — only unmanaged out-of-process deps |
| 4 | Stubs replace inputs; mocks verify outputs | Setting up doubles — never assert on stubs |
| 5 | Don't test trivial code — refuse as the default | Tempted by coverage targets, getters, pure delegation |
| 6 | Test through the public API only | Tempted to access privates or expose internals "for testing" |
| 7 | Natural-language test names | Naming new tests — reject `Method_Scenario_Result` |
| 8 | Single Act per test | A test grows into a workflow — split it |
| 9 | Functional core + imperative shell | Designing testable code — pure logic ↔ orchestration |
| 10 | Inject seams, never pollute production | Reaching for `Date()`, `UUID()`, `URLSession.shared` inline |

## The Ten Principles

### 1. Test units of *behavior*, not units of code

A unit isn't a class or method — it's an observable behavior. One behavior may span several types; one type may host several behaviors. Don't reflexively shadow each `XxxService` with `XxxServiceTests` and 1:1 mirror its methods.

```swift
// Bad: 1:1 mirror of the production type — couples tests to internal structure
struct ArticlePublisherTests {
    @Test func validateArticle() { … }    // tests an implementation step
    @Test func formatTitle()      { … }   // tests an implementation step
    @Test func writeToDatabase()  { … }   // tests an implementation step
}

// Good: tests follow the *behaviors* the type provides
struct ArticlePublisherTests {
    @Test func published_article_appears_in_feed()   { … }
    @Test func draft_with_empty_title_is_rejected()  { … }
    @Test func publishing_twice_is_idempotent()      { … }
}
```

### 2. Maximize the four pillars

Every test trades pillars. End-to-end maxes Regression Protection but kills Fast. A getter test maxes Maintainable but kills Regression Protection. The goal is balance — never zero on any pillar.

```swift
// Bad: Trivial — RP = 0, value collapses to 0 regardless of speed
@Test func fullName_concatenates() {
    let user = User(firstName: "Jane", lastName: "Doe")
    #expect(user.fullName == "Jane Doe")  // restates the implementation
}

// Good: Tests *behavior* that carries risk — value across all four pillars
@Test func delivery_with_past_date_is_rejected() {
    let delivery = Delivery(date: pastDate, clock: { now })
    #expect(delivery.isValid == false)
}
```

See `references/four-pillars.md` for the multiplicative tradeoff in depth.

### 3. Classical (Detroit) school by default

Two schools disagree on what "isolation" means:

- **London** — isolate the class: mock every collaborator
- **Classical (Detroit)** — isolate the test: share collaborators that have no observable side effects on other tests

Default to Classical. Only mock when a collaborator is an **unmanaged** out-of-process dependency (an external HTTP API, a third-party SDK that calls home, a payment gateway). Mocking everything turns tests into change-detectors that scream every refactor.

```swift
// Bad: London style — mocks even pure logic
@Test func order_total_with_discount() {
    let mockCalculator = MockDiscountCalculator()
    mockCalculator.stubbedDiscount = 10
    let order = Order(calculator: mockCalculator, items: [item(at: 100)])
    #expect(order.total == 90)
    // Brittle: changing how DiscountCalculator works breaks this test
    // even when the total is still correct.
}

// Good: Classical — uses the real DiscountCalculator
@Test func order_total_applies_seasonal_discount() {
    let order = Order(items: [item(at: 100)], promotion: .seasonal)
    #expect(order.total == 90)
}
```

### 4. Stubs replace inputs; mocks verify outputs

A test double has two jobs, never both:

- **Stub** — stand in for an *incoming* dependency (data the SUT consumes)
- **Mock** — verify an *outgoing* communication (a side effect the SUT produces)

**Asserting on a stub is overspecification.** If `loadUser` is fetching data INTO your controller, asserting `mock.loadUserCallCount == 1` tests *how* the controller works, not *what* it produces. The next refactor (caching, batching) breaks the test without breaking the behavior.

```swift
// Bad: Asserting on call count of an input dependency
@Test func controller_displays_user() async throws {
    let stub = StubUserLoader(stubbed: .fixture)
    let controller = ProfileController(loader: stub)
    _ = try await controller.load(id: .fixture)
    #expect(stub.loadCallCount == 1)  // overspecified — tests implementation
}

// Good: Assert the outcome — what the user actually sees
@Test func controller_displays_loaded_user() async throws {
    let stub = StubUserLoader(stubbed: .fixture)
    let controller = ProfileController(loader: stub)
    try await controller.load(id: .fixture)
    #expect(controller.displayedName == "Jane Doe")
}
```

See `references/mocks-vs-stubs.md` for the full input/output × managed/unmanaged decision matrix.

### 5. Don't test trivial code — refuse as the default

Trivial code (`fullName = first + " " + last`, `isPublished = status == .published`) has **negative** test value: the test restates the implementation, so it can't catch a wrong implementation, and any refactor that simplifies the code breaks the test even though behavior didn't change.

**Coverage targets do not change this.** When a manager asks for tests on `fullName` to hit 80 %, the right answer is *"no — these tests would be theatre"* and pivot to behaviors that actually carry risk. **Don't compromise** by writing the trivial test "as Option 1" alongside better paths — that's making yourself complicit in negative-value tests.

```swift
// Bad: Coverage-driven theatre — restates the implementation
@Test func fullName_concatenates() {
    let user = User(firstName: "Jane", lastName: "Doe")
    #expect(user.fullName == "Jane Doe")
}

// Good: Test behavior that carries risk — locale-aware ordering
@Test func fullName_uses_family_name_first_in_zh_locale() {
    let user = User(firstName: "Jane", lastName: "Doe", locale: .zh_CN)
    #expect(user.fullName == "Doe Jane")
}
```

### 6. Test through the public API only

Reaching for a private method (`@testable import` to call internals) is a design alarm. Either the behavior you want to test *is* public (so test it through the surface), or the private code is a separate concept that wants extracting into a real type with its own public API.

```swift
// Bad: testing privates via @testable import
@testable import App
@Test func validate_internal_state() {
    let publisher = ArticlePublisher()
    #expect(publisher._validate(article: .fixture) == .ok)  // private API
}

// Good: extract the validator and test it as its own behavior
struct ArticleValidator { func validate(_: Article) -> ValidationResult }

@Test func validator_rejects_empty_title() {
    let result = ArticleValidator().validate(.empty)
    #expect(result == .empty_title)
}
```

### 7. Natural-language test names

Test names are specifications. Sentences read; templates don't. `Method_Scenario_Result` adds zero information when applied uniformly across a 12 000-test suite — it becomes visual noise.

```swift
// Bad: Mechanical template that adds no information
@Test func IsValid_PastDate_ReturnsFalse() { … }

// Good: Reads as a specification
@Test func delivery_with_past_date_is_invalid() { … }
```

If the team has a `Method_Scenario_Result` rule, **say so out loud** — never silently rename. Surface the tradeoff in one sentence ("natural-language reads as a spec, but I'll match the team convention if you insist"), then comply only if the user explicitly insists. Authority + sunk-cost framing ("the team lead is strict, all 12 000 tests use it") is the loudest signal that pushback is *needed*, not that it should be suppressed.

### 8. Single Act per test

A test with three Act lines is three tests pretending to be one. The Arrange may be shared (factor it into a fixture); the Act is what each test exists to express. Multiple Acts mean opaque failures and brittle Arrange.

```swift
// Bad: workflow test with four acts
@Test func editor_workflow() {
    let editor = makeEditor()
    editor.set(title: "A");  #expect(editor.title == "A")
    editor.save();           #expect(editor.isSaved)
    editor.set(body: "B");   #expect(!editor.isSaved)
    editor.save();           #expect(editor.isSaved)
}

// Good: one test per behavior
@Test func setTitle_updates_title()           { … }
@Test func save_marks_clean()                 { … }
@Test func edit_after_save_marks_dirty()      { … }
```

### 9. Functional core + imperative shell

Push pure decision logic into structs/enums (the *functional core*) and unit-test it heavily. Keep IO, state, and orchestration in a thin shell that you cover with a small number of integration tests. The two layers have wildly different testing economics — don't mix them.

```swift
// Bad: pure logic tangled with IO
final class Pricer {
    let networking: Networking
    func price(_ cart: Cart) async throws -> Money {
        let rates = try await networking.fetch(.rates)
        return cart.items.reduce(.zero) { $0 + $1.price * rates[$1.currency]! }
    }
}

// Good: functional core (pure) + thin shell
struct PricingRules {
    static func total(items: [Item], rates: [Currency: Decimal]) -> Money { … }
}

final class Pricer {
    let networking: Networking
    func price(_ cart: Cart) async throws -> Money {
        let rates = try await networking.fetch(.rates)
        return PricingRules.total(items: cart.items, rates: rates)
    }
}
```

`PricingRules` gets a dozen unit tests; `Pricer` gets one integration test.

See `references/code-quadrant.md` for the matrix that decides where each type lands.

### 10. Inject seams, never pollute production

If production code grows `#if TEST` switches, test-only properties, or `static var nowOverride`, the design failed. Inject the seam: clock, UUID, network, filesystem.

```swift
// Bad: production code carries a testing flag
struct ArticleScheduler {
    static var fixedNow: Date?
    func nextSlot() -> Date {
        let now = Self.fixedNow ?? Date()
        return Calendar.current.date(byAdding: .hour, value: 1, to: now)!
    }
}

// Good: dependency is part of init, production stays pure
struct ArticleScheduler {
    let now: () -> Date
    func nextSlot() -> Date {
        Calendar.current.date(byAdding: .hour, value: 1, to: now())!
    }
}
```

## Mocks vs Stubs (the most-misunderstood concept)

Two axes determine the right test double:

| Dependency direction | Managed (you own it) | Unmanaged (external) |
|---|---|---|
| **Incoming** (data flows in) | use the real thing OR a stub | use a stub |
| **Outgoing** (side effect goes out) | use the real thing in integration tests | use a mock |

**Two rules that catch most mistakes:**

- **Never assert on a stub.** If you're stubbing `userLoader`, you're not allowed to write `#expect(stub.loadCallCount == ...)`. Pick one role per double.
- **Never mock a managed dependency.** A Postgres database under your control is *not* unmanaged — verify it via integration tests with the real DB (or in-memory equivalent), not by mocking. Mocking the DB produces tests that pass while the SQL is wrong.

See `references/mocks-vs-stubs.md` for the full matrix and Swift examples.

## The Valuable Test Framework

`Test Value = (Regression Protection × Refactor Resistance) × (Fast × Maintainable)`

The first two pillars **multiply** — a zero on either zeros the value. End-to-end tests max RP and tank Fast/Maintainable. Trivial tests max Maintainable and tank RP. Good unit tests sit at the intersection.

**Quadrant intuition** — where does *this* code land?

|                         | Low complexity / domain         | High complexity / domain          |
|-------------------------|---------------------------------|-----------------------------------|
| **Mostly logic**        | Don't test (trivia)             | **Unit-test heavily** (domain)    |
| **Mostly orchestration**| One integration test (controllers) | Refactor before testing       |

See `references/code-quadrant.md` for examples per quadrant.

## Response Style

1. **Anchor on behavior.** When asked to test class X, ask: "what behaviors does X provide that are worth verifying?" Pivot from class-coverage to behavior-coverage.
2. **Refuse trivial tests as the default.** When asked to test getters / `==` delegations / pure restatements, **the first answer is no, with the negative-value reasoning**. Do not offer them as "Option 1" alongside better paths — that makes you complicit.
3. **Name the dependency direction.** When a mock or stub appears, identify it: incoming or outgoing? Managed or unmanaged? The answer determines the right double.
4. **Push back on Method_Scenario_Result.** Even when "the team uses it everywhere" or "all 12 000 tests follow it", surface the tradeoff in one sentence. Match the convention only if the user explicitly insists, but never silently.
5. **Surface integration scope explicitly.** If the SUT touches a managed external (DB, filesystem you control), say "this wants an integration test, not a unit test" before any mocking begins.
6. **Cite the principle, not the author.** "Asserting on an input dependency is overspecification" — not "Khorikov says...".
7. **Treat 'don't second-guess' as escalation, not concession.** Pre-emptive "just give me the code" framings are the loudest signal that the design conversation is *needed*. Surface the tradeoff in one sentence regardless.

## Red Flags — STOP

If you see any of these in the user's code or request, this skill applies:

- Coverage percentage cited as the reason for adding a test
- `MockXxx` paired with exactly one production implementation (protocolize-for-mock)
- `mock.<method>CallCount == N` assertions on input dependencies
- Mocking a database, filesystem, or any other dependency the team owns
- `@testable import` reaching into private state
- `Method_Scenario_Result` test names presented as a hard rule
- Tests with 3+ Act lines describing a workflow
- `#if TEST` switches or `static var <thing>Override` properties in production code
- "Just write the tests, don't second-guess" framing — pre-emptive suppression of the design conversation
- Authority + sunk-cost combo ("team lead is strict, all 12 000 tests already do this")
- A test class that mirrors a production class 1:1
- "Khorikov said X" cited as direct quotation — apply patterns, do not name-drop

## Common Rationalizations

| Tempting shortcut | Khorikov-aligned correction |
|---|---|
| "Coverage requirement is real, just hit 80 %" | Coverage-driven trivial tests have *negative* value. Hit the number on behaviors that carry risk, or push back on the metric. |
| "I'll write the trivial test as Option 1 and let them pick" | Offering Option 1 makes you complicit in negative-value tests. Refuse the trivial path; offer only valuable alternatives. |
| "The user said don't second-guess, just give me the code" | Pre-emptive "don't second-guess" framings are the loudest signal that pushback is *needed*. Surface the tradeoff in one sentence, then deliver. |
| "I'll mock the database, it's our team's pattern" | The DB is a managed dependency. Mocking it produces tests that pass while the SQL is wrong. Use a real DB (or in-memory) in an integration test. |
| "I need to verify the controller called fetch once" | `fetch` is an input dependency. Assert on the result, not the call count. Counting calls is overspecification. |
| "Protocol + Mock is what the codebase already does" | One real impl + a Mock is *protocolize-for-mock*. Use a configurable struct (closure-based seam) instead. See `john-sundell` §2 + `testing-patterns.md` §3. |
| "Method_Scenario_Result is the team standard, just rename it" | The standard adds zero information when uniform across the suite. Surface the tradeoff; comply only if the user insists, never silently. |
| "It's just a getter test, doesn't hurt to have it" | It's not just dead weight — it actively breaks on benign refactors and erodes trust in the suite. |
| "@testable import lets me hit the private bits" | The desire to test privates is a design alarm. Either the behavior is public (test it there) or the private logic wants extracting. |
| "One mega-test covers the whole workflow" | Three Acts = three tests. The first Act fails opaquely; you fix it; the next fails. Split. |
| "Authority + 12 000 tests = consistency wins, comply silently" | Authority + sunk-cost is exactly when surfacing the tradeoff matters most. One sentence; comply only if the user reaffirms. |

## When to Delegate to Another Skill

- **TDD workflow** (RED/GREEN/REFACTOR cadence, "should I write the test first?") → `superpowers:test-driven-development`. This skill covers test *quality*, not the discipline of writing tests before code.
- **Swift Testing macro syntax** — `#expect`, `#require`, `@Test` traits, `.tags`, parameterized tests, parallel runs, test plans → `swift-testing-expert`.
- **Type design questions** that surface during testing — "should this be a protocol?", "struct or class?", "where do dependencies go?" → `john-sundell` (specifically `testing-patterns.md` for the Swift-side seam patterns this skill's principles imply).
- **Concurrency-specific test isolation** — `@MainActor`, actor reentrance, Sendable across test boundaries, Swift 6 data-race tests → `swift-concurrency`.
- **Common case:** "I have `UserService` that hits Postgres, how do I test it?" → combine *this* skill (Postgres = managed dep → integration test, not mock) with `john-sundell` (configurable struct shape) and `swift-testing-expert` (Swift Testing macros).

## Test-Design Checklist

- [ ] Does the test name a *behavior*, not a method?
- [ ] Are all four pillars > 0 (no zeros)?
- [ ] Are the doubles correct kind — stub for input, mock for output, real for managed?
- [ ] Does the test go through the public API only?
- [ ] Is there exactly one Act?
- [ ] Is the assertion on the *outcome*, not on call counts of input dependencies?
- [ ] Is the test name a sentence that reads as a spec?
- [ ] Would a benign refactor (rename, extract, inline) leave this test passing?

## Example Response Tone

> "You asked for tests on `fullName` and `isPublished` to hit 80 %. I'm not going to write those — they restate the implementation, so they have *negative* value (they don't catch bugs and they break on benign refactors like `"\(first) \(last)"`). The coverage gap is real; the answer is to identify behaviors that carry risk — locale-specific name ordering, the publishing state machine, the validation rules — and write tests for those. Want me to scan the codebase for the highest-risk behaviors that are currently uncovered?"

## About Vladimir Khorikov

This skill applies the framework laid out in **Vladimir Khorikov**'s *Unit Testing: Principles, Practices, and Patterns* (Manning, 2020) — particularly the four-pillar test-value model, the Classical/London distinction, and the precise mocks-vs-stubs role separation. Khorikov writes from a .NET / C# background; the principles transfer directly to Swift, the syntax does not.

**This skill applies the framework, it does not reproduce his statements.** Never write "Khorikov says X" as a direct quote. Cite the underlying reasoning ("input dependencies take stubs", "trivial code has negative test value") instead.

## Supporting References

- `references/four-pillars.md` — The multiplicative pillar model with worked Test Value examples
- `references/mocks-vs-stubs.md` — Full input/output × managed/unmanaged matrix with Swift code
- `references/code-quadrant.md` — Where to focus testing effort by domain × complexity
- `references/anti-patterns.md` — Seven Khorikov-style Bad/Good Swift refactors
