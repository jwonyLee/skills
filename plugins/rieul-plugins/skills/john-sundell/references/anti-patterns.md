# Anti-Patterns: Before / After

Long-form refactors for the six most common Swift/iOS smells. Each entry shows the symptom, a realistic "before", the Sundell-style "after", and why.

---

## 1. Singleton session → Injected `UserSession`

**Symptom.** `UserManager.shared` reached for anywhere that needs "the current user." Dependencies are invisible; tests need to mutate global state; multi-account is impossible by construction.

### Before

```swift
final class UserManager {
    static let shared = UserManager()
    private init() {}

    private(set) var currentUser: User?
    var isAuthenticated: Bool { currentUser != nil }

    func signIn(_ user: User) { currentUser = user }
    func signOut() { currentUser = nil }
}

struct ProfileView: View {
    var body: some View {
        if let user = UserManager.shared.currentUser {
            Text(user.displayName)
        } else {
            Text("Signed out")
        }
    }
}
```

### After

```swift
struct UserSession {
    var currentUser: User?
    var isAuthenticated: Bool { currentUser != nil }

    mutating func signIn(_ user: User) { currentUser = user }
    mutating func signOut() { currentUser = nil }
}

struct ProfileView: View {
    let session: UserSession

    var body: some View {
        if let user = session.currentUser {
            Text(user.displayName)
        } else {
            Text("Signed out")
        }
    }
}
```

For apps that need an observable current session across the view tree, expose it via `@Observable` or `@Environment` — not a singleton.

### Why this is Sundell-style

- Dependencies are visible: `ProfileView` declares it needs a session.
- Multi-account isn't ruled out by the type: you can own more than one `UserSession`.
- Tests pass in a fixed session literal; no global mutation.

---

## 2. Base view controller → Small collaborators

**Symptom.** Every screen inherits from `BaseViewController` for loading/error/retry. The base class grows, subclasses override in inconsistent ways, and "shared" means "shared in confusing ways."

### Before

```swift
class BaseViewController: UIViewController {
    func showLoading() { /* 30 lines */ }
    func hideLoading() { /* 10 lines */ }
    func showError(_ error: Error) { /* 25 lines */ }
    func retry() { fatalError("subclass must override") }
}

class ArticleViewController: BaseViewController {
    override func retry() { loader.load() }
    func viewDidLoad() { super.viewDidLoad(); showLoading(); loader.load() }
}
```

### After

```swift
final class LoadingPresenter {
    func show(on: UIViewController) { /* … */ }
    func hide() { /* … */ }
}

final class ErrorPresenter {
    func present(_ error: Error, on: UIViewController, retry: @escaping () -> Void) { /* … */ }
}

final class ArticleViewController: UIViewController {
    private let loader: ArticleLoader
    private let loading: LoadingPresenter
    private let errors: ErrorPresenter

    init(loader: ArticleLoader,
         loading: LoadingPresenter = .init(),
         errors: ErrorPresenter = .init()) {
        self.loader = loader
        self.loading = loading
        self.errors = errors
        super.init(nibName: nil, bundle: nil)
    }

    override func viewDidLoad() {
        super.viewDidLoad()
        load()
    }

    private func load() {
        loading.show(on: self)
        Task {
            do {
                let articles = try await loader.load()
                loading.hide()
                render(articles)
            } catch {
                loading.hide()
                errors.present(error, on: self, retry: { [weak self] in self?.load() })
            }
        }
    }
}
```

### Why this is Sundell-style

- Shared behavior is *named* (`LoadingPresenter`, `ErrorPresenter`) rather than hidden in a base class.
- No `fatalError` "override me" contract; retry is an explicit closure.
- You can mix and match presenters per screen — no forced inheritance chain.

---

## 3. "Test-only" protocol → Concrete type + closure seam

**Symptom.** A protocol exists only so tests can inject a `MockX`. There's exactly one real implementation and nothing else. Every call site pays the protocol tax; tests carry a parallel hierarchy.

### Before

```swift
protocol UserLoading {
    func load(id: UserID) async throws -> User
}

final class UserLoader: UserLoading {
    private let networking: Networking
    init(networking: Networking) { self.networking = networking }
    func load(id: UserID) async throws -> User {
        try await networking.fetch(.user(id))
    }
}

// Tests
final class MockUserLoader: UserLoading {
    var stubbed: Result<User, Error> = .failure(TestError.notStubbed)
    func load(id: UserID) async throws -> User { try stubbed.get() }
}
```

### After

```swift
struct UserLoader {
    let load: (UserID) async throws -> User

    static func live(networking: Networking) -> UserLoader {
        UserLoader { id in try await networking.fetch(.user(id)) }
    }
}

// Tests
let loader = UserLoader(load: { _ in User.fixture })
```

### Why this is Sundell-style

- One implementation → no protocol needed. The type is its own seam.
- Fakes are one-liners, not a separate class.
- Call sites stay on the concrete type; no artificial polymorphism.

**When protocols are still right:** genuinely 2+ implementations (`URLSession` vs offline cache), cross-module abstraction, or platform-specific polymorphism. The test is "does a second implementation exist today or is planned?", not "can I mock it?"

---

## 4. Raw `String` ID → Domain type

**Symptom.** Every API takes `String` for IDs, tokens, codes, and URLs. Arguments get swapped at call sites. Equality checks silently compare a user ID to an article ID.

### Before

```swift
func fetchArticles(userID: String, token: String, category: String) async throws -> [Article]

// Caller — silently broken, compiler says nothing
let articles = try await fetchArticles(
    userID: token,       // swap!
    token: userID,
    category: categoryID // different kind of ID
)
```

### After

```swift
struct UserID: Hashable, Codable { let rawValue: String }
struct AuthToken: Hashable { let rawValue: String }
struct CategoryID: Hashable, Codable { let rawValue: String }

func fetchArticles(userID: UserID, token: AuthToken, category: CategoryID) async throws -> [Article]

// Caller — compiler stops the swap
let articles = try await fetchArticles(
    userID: token,        // compile error: cannot convert value of type 'AuthToken' to expected argument type 'UserID'
    token: userID,
    category: categoryID
)
```

### Why this is Sundell-style

- The compiler enforces intent. Wrong-kind-of-string bugs become unrepresentable.
- `Hashable` / `Codable` conformance comes along for free via synthesized implementations.
- The cost is minimal (`struct` with one property). The payoff compounds across the codebase.

**Optional enhancement:** conform to `ExpressibleByStringLiteral` if literals are needed in tests/fixtures.

---

## 5. Inline `Date()` / `URLSession.shared` → Injected

**Symptom.** Business logic constructs `Date()`, `UUID()`, or reaches for `URLSession.shared` directly. Tests can't fix time, can't stub responses, can't verify ordering.

### Before

```swift
struct ArticlePublisher {
    func schedule(_ article: Article) -> ScheduledArticle {
        ScheduledArticle(
            id: UUID(),
            article: article,
            publishedAt: Date()
        )
    }

    func fetchLatest() async throws -> [Article] {
        let (data, _) = try await URLSession.shared.data(from: .articlesEndpoint)
        return try JSONDecoder().decode([Article].self, from: data)
    }
}
```

### After

```swift
struct ArticlePublisher {
    let now: () -> Date
    let uuid: () -> UUID
    let networking: Networking

    func schedule(_ article: Article) -> ScheduledArticle {
        ScheduledArticle(id: uuid(), article: article, publishedAt: now())
    }

    func fetchLatest() async throws -> [Article] {
        try await networking.fetch(.articles)
    }
}

extension ArticlePublisher {
    static func live(networking: Networking) -> Self {
        .init(now: Date.init, uuid: UUID.init, networking: networking)
    }
}

// Tests
let fixedNow = Date(timeIntervalSince1970: 1_700_000_000)
let publisher = ArticlePublisher(
    now: { fixedNow },
    uuid: { UUID(uuidString: "0000…")! },
    networking: .fixture(articles: [.sample])
)
```

### Why this is Sundell-style

- Non-deterministic things (clock, randomness, network) are dependencies, not hidden facts.
- Tests run in milliseconds, deterministically, without swizzling.
- The `live` factory keeps production call sites ergonomic.

---

## 6. Giant SwiftUI `View` → Sub-views + `ViewModifier`

**Symptom.** A single `View` is 300+ lines. State ownership is unclear. Small changes force re-reading everything. The preview is slow or broken.

### Before

```swift
struct ArticleDetailView: View {
    @Observable var viewModel: ArticleDetailViewModel
    @State private var isBookmarked = false
    @State private var isSharing = false
    @State private var selectedImageIndex = 0
    @State private var commentsExpanded = false
    // … 15 more @State

    var body: some View {
        ScrollView {
            VStack {
                // 80 lines of header
                // 100 lines of image carousel
                // 60 lines of article body
                // 70 lines of comments
                // 40 lines of related articles
            }
        }
        .toolbar { /* 30 lines */ }
        .sheet(isPresented: $isSharing) { /* 25 lines */ }
    }
}
```

### After

```swift
struct ArticleDetailView: View {
    @Observable var viewModel: ArticleDetailViewModel

    var body: some View {
        ScrollView {
            VStack(spacing: 24) {
                ArticleHeader(article: viewModel.article)
                ArticleImageCarousel(images: viewModel.article.images)
                ArticleBody(markdown: viewModel.article.body)
                CommentsSection(comments: viewModel.comments, onLoadMore: viewModel.loadMore)
                RelatedArticlesRow(related: viewModel.related)
            }
            .padding(.horizontal)
        }
        .articleToolbar(viewModel: viewModel)
    }
}

private struct ArticleHeader: View { /* 20-40 lines, owns only header state */ }
private struct ArticleImageCarousel: View { /* … */ }
private struct ArticleBody: View { /* … */ }
private struct CommentsSection: View { /* owns `commentsExpanded` */ }
private struct RelatedArticlesRow: View { /* … */ }

private struct ArticleToolbarModifier: ViewModifier {
    let viewModel: ArticleDetailViewModel
    @State private var isSharing = false
    @State private var isBookmarked = false

    func body(content: Content) -> some View {
        content
            .toolbar { /* … */ }
            .sheet(isPresented: $isSharing) { ShareSheet(article: viewModel.article) }
    }
}

extension View {
    func articleToolbar(viewModel: ArticleDetailViewModel) -> some View {
        modifier(ArticleToolbarModifier(viewModel: viewModel))
    }
}
```

### Why this is Sundell-style

- Each sub-view has one job and owns exactly the state it needs.
- `ViewModifier` gives a reusable, named unit for cross-cutting UI behavior.
- Previews become fast and meaningful — you can preview `ArticleHeader` in isolation.
- File shrinks to what it's actually about: composition of parts.

---

## The Pattern Behind the Patterns

All six refactors apply the same moves:

1. **Make dependencies visible** (singleton → DI, `Date()` → injected clock).
2. **Prefer composition to inheritance** (base class → collaborators, giant view → sub-views).
3. **Let types carry meaning** (raw string → domain type).
4. **Don't abstract prematurely** (protocol-for-tests → concrete type with a seam).

If you find yourself reaching for any of the "before" shapes, walk through this list before writing more code.
