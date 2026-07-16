# SwiftUI Notes — Beginner to Intermediate
### Property Wrappers, Data Flow, and Combine

---

## PART 1: PROPERTY WRAPPERS

### 1. `@State`

**What it is:** A source of truth for simple value types that live *inside* a single view.

**When to use:**
- Value types only: `Int`, `String`, `Bool`, `Double`, `struct`, `enum`, arrays/dicts of these.
- The data is owned by this view and doesn't need to be shared elsewhere.
- Toggle states, text field bindings, loading flags, sheet/alert presentation booleans.

```swift
struct CounterView: View {
    @State private var count = 0
    @State private var isShowingSheet = false

    var body: some View {
        VStack {
            Text("Count: \(count)")
            Button("Increment") { count += 1 }
        }
        .sheet(isPresented: $isShowingSheet) { DetailView() }
    }
}
```

**Rules of thumb:**
- Always mark `@State` as `private` — it should not be set from outside the view.
- Never use `@State` for reference types (classes) that need to notify other views of changes — use `@StateObject`/`@ObservedObject` instead.
- SwiftUI stores `@State` outside the struct's memory so it survives view re-creation (structs are re-initialized constantly, but `@State` value persists).

---

### 2. `@StateObject` vs `@ObservedObject`

Both are used with reference types (classes) conforming to `ObservableObject`. The difference is about **who owns the object's lifecycle**.

| | `@StateObject` | `@ObservedObject` |
|---|---|---|
| Owns the object? | **Yes** — creates & owns it | **No** — receives it from elsewhere |
| Survives view re-creation? | Yes, created once | No, re-created if the parent re-creates it |
| Use when | This view is the *source* of the object | This view *receives* the object as a parameter |

```swift
// ViewModel
class NewsViewModel: ObservableObject {
    @Published var articles: [Article] = []
    @Published var isLoading = false
}

// Parent — OWNS the view model → StateObject
struct NewsListView: View {
    @StateObject private var viewModel = NewsViewModel()

    var body: some View {
        List(viewModel.articles) { article in
            // pass the SAME instance down → child uses @ObservedObject
            NewsRowView(viewModel: viewModel)
        }
    }
}

// Child — RECEIVES the view model → ObservedObject
struct NewsRowView: View {
    @ObservedObject var viewModel: NewsViewModel
    var body: some View { /* ... */ }
}
```

**Golden rule:** If you write `= SomeClass()` inline, it should be `@StateObject`. If it's passed in via init/parameter, it should be `@ObservedObject`.

> Common bug: Using `@StateObject` in a child view that gets re-created often (e.g. inside a `ForEach` row) is fine — StateObject protects it. But using `@ObservedObject` at the *top* of a view hierarchy where the parent re-renders often will cause the object to reset unexpectedly.

---

### 3. `@Published`

**What it is:** A property wrapper used **inside an `ObservableObject` class** — announces changes to any view observing that object, triggering a re-render.

```swift
class AuthViewModel: ObservableObject {
    @Published var isLoggedIn = false
    @Published var user: User?
    @Published var errorMessage: String?
}
```

**When to use:**
- Any property inside a ViewModel/Model class whose change should refresh the UI.
- Do NOT mark every property `@Published` — only ones the UI actually reads (excessive `@Published` = unnecessary re-renders).

---

### 4. `@Environment` / `@EnvironmentObject`

Two different tools, easy to confuse:

**`@Environment`** — reads built-in system values or custom environment keys (not full objects) passed implicitly down the view tree.

```swift
struct MyView: View {
    @Environment(\.colorScheme) var colorScheme
    @Environment(\.dismiss) var dismiss   // dismiss a sheet/nav push

    var body: some View {
        Text(colorScheme == .dark ? "Dark" : "Light")
        Button("Close") { dismiss() }
    }
}
```

**`@EnvironmentObject`** — injects a shared `ObservableObject` implicitly to an entire view subtree, without manually passing it through every initializer.

```swift
// App entry point
@main
struct MyApp: App {
    @StateObject var session = SessionManager()
    var body: some Scene {
        WindowGroup {
            ContentView()
                .environmentObject(session)   // injected once, available to ALL children
        }
    }
}

// Any child, no matter how deep
struct ProfileView: View {
    @EnvironmentObject var session: SessionManager
    var body: some View {
        Text(session.currentUser?.name ?? "Guest")
    }
}
```

**When to use `@EnvironmentObject` vs passing `@ObservedObject` manually:**
- Use `@EnvironmentObject` for **app-wide/global** state (auth session, theme, user settings) needed by many unrelated views.
- Use `@ObservedObject` passed explicitly when only a *specific* parent-child chain needs the object — keeps data flow explicit and easier to trace/test.

⚠️ If a view expects an `@EnvironmentObject` that was never injected up the chain, the app **crashes at runtime**. Always inject it at a level above every consumer (commonly in the `App` struct or a top-level container).

---

### 5. `@Binding`

Not asked explicitly but essential alongside `@State` — creates a **two-way connection** so a child view can read *and write* a value owned by a parent.

```swift
struct ParentView: View {
    @State private var name = ""
    var body: some View {
        ChildTextField(name: $name)   // $ creates the Binding
    }
}

struct ChildTextField: View {
    @Binding var name: String
    var body: some View {
        TextField("Name", text: $name)
    }
}
```

**When to use:** Child needs to *modify* a value that belongs to a parent (custom toggles, custom text fields, custom steppers).

---

### 6. `@ViewBuilder`

A function attribute that lets you write **multiple SwiftUI views in a block** (like inside `body`) that get combined into one view automatically — it's what makes `body { ... }` syntax possible at all.

**When to use:**
- Writing a custom view/container that takes view content as a parameter (like `VStack`, `HStack` do).
- Conditional view logic inside a helper function/computed property that returns different view types.

```swift
struct CardContainer<Content: View>: View {
    @ViewBuilder let content: Content   // accepts multiple views as a block

    var body: some View {
        VStack {
            content
        }
        .padding()
        .background(Color.white)
        .cornerRadius(12)
    }
}

// Usage
CardContainer {
    Text("Title")
    Text("Subtitle")
    Image(systemName: "star")
}
```

Without `@ViewBuilder`, you could only return ONE view type from a closure/function. With it, Swift stitches multiple `View`s into a `TupleView` behind the scenes.

```swift
@ViewBuilder
func statusView(isLoading: Bool) -> some View {
    if isLoading {
        ProgressView()
    } else {
        Text("Done")
    }
}
```

---

### Quick Decision Table

| Situation | Wrapper |
|---|---|
| Simple value owned by this view | `@State` |
| Class instance created HERE | `@StateObject` |
| Class instance received from parent | `@ObservedObject` |
| Property inside an ObservableObject class that should update UI | `@Published` |
| Reading system value (colorScheme, dismiss) | `@Environment` |
| Shared object needed by many unrelated descendant views | `@EnvironmentObject` |
| Child needs to write back to parent's value | `@Binding` |
| Custom container/function returning multiple/conditional views | `@ViewBuilder` |

---

## PART 2: PASSING DATA BETWEEN SCREENS / FILES

### A. SwiftUI View → SwiftUI View (child needs data, doesn't modify it)

Just pass as a regular `let`/parameter through `init`:

```swift
struct DetailView: View {
    let article: Article   // plain, read-only
    var body: some View { Text(article.title) }
}

// Parent
NavigationLink(destination: DetailView(article: selectedArticle)) {
    Text("Open")
}
```

### B. SwiftUI View → SwiftUI View (child needs to modify parent's value)

Use `@Binding` (see above) — the `$` prefix passes a two-way reference.

### C. Sharing a class instance across MANY views (ViewModel pattern)

Own it once with `@StateObject` at the top, pass down as `@ObservedObject`, or inject with `.environmentObject()` for global reach. (See Part 1, sections 2 & 4.)

### D. Passing data BACK from a child screen (e.g. picker → parent)

Most common patterns:

**1. Binding (simple values, same hierarchy)**
```swift
struct ColorPickerView: View {
    @Binding var selectedColor: Color
}
```

**2. Closure callback**
```swift
struct PickerView: View {
    var onSelect: (String) -> Void
    var body: some View {
        Button("Pick Apple") { onSelect("Apple") }
    }
}
// Parent
PickerView(onSelect: { picked in self.fruit = picked })
```

**3. Shared ObservableObject** (best for complex/async flows — the child updates the shared object, parent observes it automatically — no explicit "sending back" needed).

### E. Plain Swift class/struct → Swift class/struct (no UI)

Ordinary Swift — no property wrappers needed unless the data needs to trigger UI updates:

```swift
struct NetworkService {
    func fetchUser() -> User { /* ... */ }
}

class UserManager {
    var currentUser: User?
    func updateUser(_ user: User) { currentUser = user }
}
```

### F. Plain Swift file (e.g. Model/Service) → SwiftUI View

The **Service/Model itself stays plain Swift**. The **ViewModel** (an `ObservableObject`) is the bridge — it calls the service, stores results in `@Published` properties, and the View observes the ViewModel.

```
Model (struct)  →  Service (plain Swift, fetches/transforms data)
                        ↓
                 ViewModel (@Published properties, ObservableObject)
                        ↓
                    SwiftUI View (@StateObject / @ObservedObject)
```

This is the standard **MVVM** data flow you're already using (matches your PocketLedger/Memoize/Zenith architecture).

### G. Passing data across unrelated view hierarchies (deep linking, tab switches)

Use one of:
- `@EnvironmentObject` injected at the app root.
- `NotificationCenter` (rare in SwiftUI, mostly legacy/UIKit bridging).
- A shared singleton service (`.shared`) — acceptable for things like `NetworkMonitor.shared`, but avoid overusing singletons for app state; prefer `@EnvironmentObject`.

---

## PART 3: COMBINE — `AnyPublisher`, `.map`, and friends

### Why Combine shows up in networking

`URLSession.shared.dataTaskPublisher(for:)` returns a **Combine publisher** instead of using a completion-handler closure. It lets you chain transformations (`.map`, `.decode`) declaratively instead of nesting closures.

### Breaking down your example line by line

```swift
func topHeadlines() -> AnyPublisher<NewsResponse, Error> {
    return URLSession.shared.dataTaskPublisher(for: request)
        .map(\.data)
        .decode(type: NewsResponse.self, decoder: JSONDecoder())
        .receive(on: DispatchQueue.main)
        .eraseToAnyPublisher()
}
```

| Step | What it does |
|---|---|
| `dataTaskPublisher(for: request)` | Starts the network call. Emits a `(data: Data, response: URLResponse)` tuple, or an error. |
| `.map(\.data)` | Transforms the emitted tuple → keeps only the `Data` part (key path syntax for `.map { $0.data }`). |
| `.decode(type:decoder:)` | Takes the raw `Data`, runs it through `JSONDecoder()`, turns it into your `NewsResponse` model. Throws/fails if JSON doesn't match the model. |
| `.receive(on: DispatchQueue.main)` | Switches the pipeline to the main thread from here onward — **required** before updating `@Published` properties (UI must update on main thread). |
| `.eraseToAnyPublisher()` | Hides the complicated concrete publisher type (a long chain of generics) behind the simple `AnyPublisher<NewsResponse, Error>` type — makes the function signature clean and stable even if you change the pipeline internally later. |

### When to use `AnyPublisher` as a return type

- When you want to **expose a function that returns "a stream of one value/error"** without leaking Combine's internal generic types (`Publishers.MapError<Publishers.Decode<...>>` etc.) to the caller.
- Common in a **networking/service layer** so the ViewModel just sees a clean `AnyPublisher<Model, Error>`.

### How to consume/handle an `AnyPublisher` in a ViewModel

Use `.sink` to subscribe, and store the subscription in a `Set<AnyCancellable>` (or the pipeline stops immediately when it goes out of scope):

```swift
class NewsViewModel: ObservableObject {
    @Published var articles: [Article] = []
    @Published var isLoading = false
    @Published var errorMessage: String?

    private var cancellables = Set<AnyCancellable>()   // keeps subscriptions alive
    private let service = NewsService()

    func loadHeadlines() {
        isLoading = true
        service.topHeadlines()
            .sink(receiveCompletion: { [weak self] completion in
                self?.isLoading = false
                if case .failure(let error) = completion {
                    self?.errorMessage = error.localizedDescription
                }
            }, receiveValue: { [weak self] response in
                self?.articles = response.articles
            })
            .store(in: &cancellables)   // MUST store, or it cancels instantly
    }
}
```

**Key rules:**
- `.sink` has two closures: `receiveCompletion` (fires once, either `.finished` or `.failure(error)`) and `receiveValue` (fires each time new data arrives).
- Always `.store(in: &cancellables)` — a Combine subscription is cancelled automatically when its `AnyCancellable` is deallocated.
- Use `[weak self]` in closures to avoid retain cycles (same rule as completion handlers).

### `.map` — general use, beyond this example

`.map` transforms each emitted value into something else, similar to `Array.map`:

```swift
$searchText
    .map { $0.lowercased() }          // transform each new text value
    .sink { print($0) }
```

Use `.map` whenever you need to reshape/transform data flowing through a pipeline — extracting a property (`\.data`), converting types, formatting, etc.

### Other common Combine operators worth knowing at this stage

| Operator | Use |
|---|---|
| `.map` | Transform emitted value |
| `.decode(type:decoder:)` | JSON → Model |
| `.receive(on:)` | Switch which thread/queue delivers subsequent values (almost always `.main` before touching `@Published`) |
| `.eraseToAnyPublisher()` | Simplify the exposed type |
| `.sink` | Actually subscribe/consume the stream |
| `.assign(to:on:)` | Pipe values directly into a property (alternative to `.sink` when you don't need custom logic) |
| `.debounce(for:scheduler:)` | Wait for a pause before emitting — great for live search-as-you-type |
| `.removeDuplicates()` | Skip emitting if the new value equals the last one |
| `.combineLatest` | Merge two publishers, react whenever either changes |

**`.assign` example** (search-as-you-type debounce, very common intermediate pattern):
```swift
$searchQuery
    .debounce(for: .milliseconds(300), scheduler: RunLoop.main)
    .removeDuplicates()
    .sink { [weak self] query in self?.performSearch(query) }
    .store(in: &cancellables)
```

### When to reach for Combine vs `async/await`

- Modern SwiftUI (iOS 15+) often prefers **`async/await`** for one-shot network calls — simpler, no `AnyCancellable` bookkeeping:
```swift
func topHeadlines() async throws -> NewsResponse {
    let (data, _) = try await URLSession.shared.data(for: request)
    return try JSONDecoder().decode(NewsResponse.self, from: data)
}
```
- **Combine still wins** for continuous streams: live search debouncing, combining multiple observable values, WebSocket/SSE-style continuous data (relevant to your LivePulse and Memoize projects), or reacting to `@Published` changes from multiple sources at once.
- Many real codebases mix both: Combine for reactive UI state, `async/await` for simple one-off fetches.

---

## Quick Mental Model Summary

```
Model (struct)        →  plain data, Codable
Service (plain class) →  fetch/transform, returns AnyPublisher or async
ViewModel             →  ObservableObject, @Published properties,
                          subscribes to Service via .sink / await
View (SwiftUI)         →  @StateObject (owns VM) or @ObservedObject (receives VM)
                          reads @Published properties, view auto-updates
```

This is the flow underlying your PocketLedger, LivePulse, Memoize, and Zenith projects.
