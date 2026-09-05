---
type: "agent_requested"
description: "Swift 6 + iOS 18 + SwiftUI/SwiftData coding guidelines"
---
# Swift 6 on iOS 18: SwiftUI, Observation & SwiftData Field Guide

This stack builds native iOS/iPadOS apps with the Swift 6.3 toolchain in Swift 6 language mode (strict concurrency), a minimum deployment target of iOS 18, SwiftUI + the Observation framework for UI and state, SwiftData for persistence, SwiftPM plus XcodeGen for project structure, Swift Testing for tests, and SwiftLint plus swift-format for linting and formatting. It is exceptional at expressing correct, data-race-free UI code with very little ceremony: `@Observable` models drive fine-grained view updates, `@Model` types collapse a persistence layer into plain Swift declarations, and the compiler proves thread safety at build time. Optimize for main-actor-by-default code that leaves the main actor only on purpose, value-based navigation, and models whose isolation and Sendability the compiler can verify.

The biggest way agents write wrong-but-plausible code here is by importing habits from adjacent ecosystems: reaching for `ObservableObject`/`@Published`/`@StateObject` and Combine instead of `@Observable`/`@State`/`@Bindable`; sprinkling `@MainActor` everywhere (or, worse, `@unchecked Sendable` and `nonisolated(unsafe)`) to silence concurrency errors instead of adopting default main-actor isolation; passing SwiftData model objects across actor boundaries as if they were `Sendable`; and using `NavigationView`, XCTest (`XCTAssert…`), or manual `Info.plist` wiring that the modern APIs have replaced. A new SDK is not a new deployment target: build against the iOS 26 SDK (an App Store requirement — apps uploaded to App Store Connect must be built with Xcode 26 or later using an SDK for iOS 26, iPadOS 26, tvOS 26, visionOS 26, or watchOS 26) while keeping iOS 18 as the runtime floor, and guard any newer API behind an availability check.

## Toolchain, language mode, and the SDK/target split

Three version axes are independent and must not be conflated:

- **Toolchain/SDK:** Xcode 26 (Swift 6.3 compiler, iOS 26 SDK). The App Store requires builds made with the iOS 26 SDK; this is what you compile against.
- **Language mode:** Swift 6 (`SWIFT_VERSION = 6.0`), which turns on complete strict-concurrency checking. This is a *language* setting, not the toolchain number.
- **Minimum runtime:** iOS 18. This is a hard floor to build on, never to be raised silently. Code that supports iOS 18 behind an availability check is current code.

Availability floors that matter on this stack, because several attractive APIs sit *above* iOS 18:

| API / feature | Availability floor | On iOS 18 target |
|---|---|---|
| `#Index`, `#Unique`, `#Expression`, custom `DataStore` | iOS 18 | Usable directly |
| SwiftData class inheritance (`@available` subclasses) | iOS 26 | Requires `if #available(iOS 26, *)` |
| `Observations` async sequence | iOS 26 | Requires `if #available(iOS 26, *)` |
| `MainActorMessage` / `AsyncMessage` notification types | iOS 26 | Requires availability guard |

New Xcode 26 app targets enable *approachable concurrency* by default (`SWIFT_APPROACHABLE_CONCURRENCY = YES` and `SWIFT_DEFAULT_ACTOR_ISOLATION = MainActor`). Keep these on: they make the whole module main-actor-isolated unless a declaration opts out, which is the correct default for a UI app and eliminates most spurious data-race diagnostics.

## Concurrency: default to the main actor, leave it on purpose

Under default main-actor isolation, every type, function, and property in the target is implicitly `@MainActor` unless marked otherwise. You no longer annotate views, view models, and models individually. The model inverts the burden of proof: everything is on the main actor, and you prove the few places you leave it.

Two Swift 6.2 features change how async functions behave and are the ones agents most often misuse:

- **`nonisolated(nonsending)` (SE-0461, "Run nonisolated async functions on the caller's actor by default," the new default for `nonisolated async` functions under the `NonisolatedNonsendingByDefault` upcoming-feature flag):** a `nonisolated async` function runs on the *caller's* actor instead of hopping to the global executor. This removes accidental thread hops and the `Sendable`-across-the-boundary errors they caused.
- **`@concurrent` (the escape hatch):** marks a `nonisolated async` function to run on the global executor (a background thread). Use it deliberately for heavy CPU work — decoding, image processing, parsing — that must not block the caller.

```swift
import Foundation

// With default MainActor isolation, this type is @MainActor implicitly.
@Observable
final class FeedViewModel {
    private(set) var articles: [Article] = []
    private(set) var isLoading = false
    var loadError: String?

    private let loader: ArticleLoader

    init(loader: ArticleLoader) {
        self.loader = loader
    }

    // Runs on the main actor. The await below does NOT hop threads,
    // so `articles` assignment is a plain main-actor mutation.
    func load() async {
        isLoading = true
        defer { isLoading = false }
        do {
            articles = try await loader.fetchLatest()
        } catch is CancellationError {
            // Task was cancelled (view disappeared) — leave state as-is.
        } catch {
            loadError = error.localizedDescription
        }
    }
}

struct ArticleLoader: Sendable {
    let session: URLSession

    // nonisolated(nonsending) by default: runs on the caller's actor.
    func fetchLatest() async throws -> [Article] {
        let (data, _) = try await session.data(from: .latestFeed)
        // Push the CPU-bound decode off the main actor deliberately.
        return try await Self.decode(data)
    }

    // Explicitly runs on the global executor; keep it pure and Sendable in/out.
    @concurrent
    private static func decode(_ data: Data) async throws -> [Article] {
        try JSONDecoder().decode([Article].self, from: data)
    }
}
```

Rules that hold regardless of isolation defaults:

- **Never** use `@unchecked Sendable` or `nonisolated(unsafe)` to silence a diagnostic. They disable the exact check that makes this stack safe. Restructure ownership instead.
- Prefer value types (`struct`, `enum`) crossing actor boundaries; they are `Sendable` when their stored properties are.
- Honor cancellation: catch `CancellationError` (or check `Task.isCancelled`) in long-running work, and let `.task` cancellation tear down in-flight requests when a view disappears.
- Use structured concurrency (`async let`, `withTaskGroup`) for parallel work; reserve unstructured `Task { }` for fire-and-forget work tied to a view lifecycle.

## State and models: Observation, not ObservableObject

The Observation framework replaces `ObservableObject` entirely for new code. `@Observable` instruments each stored property so a SwiftUI view re-renders only when a property it actually read changes — finer-grained and faster than `@Published`, with no Combine dependency.

Pick the property wrapper by ownership, not by habit:

| Wrapper | Use when | Notes |
|---|---|---|
| `@State` | The view *owns* the observable object or value | Create the object here; SwiftUI keeps it alive across body re-runs |
| `@Bindable` | You need `$`-bindings to an `@Observable` object the view does *not* own | Replaces `@ObservedObject` for two-way binding to reference types |
| `@Environment` | Reading a shared `@Observable` injected up the tree | Replaces `@EnvironmentObject`; inject with `.environment(_:)` |
| `@Binding` | Two-way binding to a value type owned elsewhere | For `struct`/`enum`/primitives, not observable classes |
| `@ObservationIgnored` | A stored property must not trigger view updates | Caches, engines, injected services |

```swift
import SwiftUI
import Observation

@Observable
final class EditorModel {
    var title: String
    var body: String
    @ObservationIgnored private var autosaveTimer: Timer?

    init(title: String = "", body: String = "") {
        self.title = title
        self.body = body
    }
}

// Owner creates the model with @State.
struct EditorScreen: View {
    @State private var model = EditorModel()

    var body: some View {
        EditorForm(model: model)
    }
}

// Child needs bindings but does not own the model: use @Bindable.
struct EditorForm: View {
    @Bindable var model: EditorModel

    var body: some View {
        Form {
            TextField("Title", text: $model.title)
            TextEditor(text: $model.body)
        }
    }
}
```

On iOS 26 and later you can observe an `@Observable` model as an `AsyncSequence` with `Observations`, which delivers transactional (batched, did-set) snapshots — multiple synchronous property changes coalesce into one emission at the next suspension point, avoiding redundant UI work. Because it is an iOS 26 API, guard it and keep an iOS 18 path:

```swift
@available(iOS 26, *)
func persistState(of model: EditorModel) async {
    for await snapshot in Observations({ (model.title, model.body) }) {
        await Storage.shared.save(title: snapshot.0, body: snapshot.1)
    }
}
```

Do **not** mix `@Observable` with `ObservableObject`/`@Published` in the same type, and do not wrap an `@Observable` object in `@StateObject` — that pairing is for the legacy protocol only.

## App structure and navigation

Compose the app from an `App` value, a `WindowGroup`, and value-based navigation. Use `NavigationStack` with `navigationDestination(for:)`; `NavigationView` is superseded. Model routes as a `Hashable` enum and drive them through a `NavigationPath` (heterogeneous) or a typed array (homogeneous) held in `@State`.

```swift
import SwiftUI

enum Route: Hashable {
    case article(Article.ID)
    case settings
}

struct RootView: View {
    @State private var path = NavigationPath()

    var body: some View {
        NavigationStack(path: $path) {
            FeedList { id in path.append(Route.article(id)) }
                .navigationTitle("Feed")
                .navigationDestination(for: Route.self) { route in
                    switch route {
                    case .article(let id): ArticleDetail(id: id)
                    case .settings:        SettingsView()
                    }
                }
                .toolbar {
                    ToolbarItem(placement: .topBarTrailing) {
                        Button("Settings") { path.append(Route.settings) }
                    }
                }
        }
    }
}
```

Prefer `NavigationLink(value:)` over `NavigationLink(destination:)` so navigation is state-driven and testable. Centralize the `navigationDestination` mapping at the stack root rather than scattering destination views. Reset to root with `path = NavigationPath()`; pop one level with `path.removeLast()`. A known iOS 18 quirk: `navigationDestination` inside a `TabView` tab can fire twice after switching tabs and back — keep destination construction idempotent (no side effects in the destination builder).

## Data layer: SwiftData

Define models as `final class` with `@Model`. Under default main-actor isolation, opt models *out* to `nonisolated` when a background `@ModelActor` will touch them, and keep them plain otherwise. Configure the container once at the app entry point and read with `@Query` in views; do writes and bulk work off the main actor.

```swift
import SwiftData
import Foundation

@Model
final class Trip {
    #Unique<Trip>([\.id], [\.name, \.startDate])
    #Index<Trip>([\.startDate], [\.name])

    @Attribute(.unique) var id: UUID
    var name: String
    var destination: String
    var startDate: Date

    // Always specify delete rule and inverse explicitly.
    @Relationship(deleteRule: .cascade, inverse: \BucketListItem.trip)
    var items: [BucketListItem] = []

    init(id: UUID = UUID(), name: String, destination: String, startDate: Date) {
        self.id = id
        self.name = name
        self.destination = destination
        self.startDate = startDate
    }
}

@Model
final class BucketListItem {
    var title: String
    var isInPlan: Bool
    var trip: Trip?

    init(title: String, isInPlan: Bool = false) {
        self.title = title
        self.isInPlan = isInPlan
    }
}
```

`#Index` and `#Unique` are iOS 18 schema macros — usable directly at this floor. Each may appear once per model and takes one or more key-path arrays; `#Unique` performs an upsert on conflict rather than throwing. Add an index only when it matches a hot, repeated fetch (equality/range filters, sort descriptors); indexes are stored metadata and are not free. Treat adding or removing either macro as a schema change and test existing stores before shipping.

Container and query setup:

```swift
import SwiftUI
import SwiftData

@main
struct TripsApp: App {
    let container: ModelContainer

    init() {
        do {
            let schema = Schema([Trip.self, BucketListItem.self])
            let config = ModelConfiguration(schema: schema, isStoredInMemoryOnly: false)
            container = try ModelContainer(for: schema, configurations: config)
        } catch {
            fatalError("Failed to create ModelContainer: \(error)")
        }
    }

    var body: some Scene {
        WindowGroup { TripList() }
            .modelContainer(container)
    }
}

struct TripList: View {
    @Environment(\.modelContext) private var context
    @Query(sort: \Trip.startDate, order: .forward) private var trips: [Trip]

    var body: some View {
        List {
            ForEach(trips) { trip in
                NavigationLink(trip.name, value: Route.article(trip.id))
            }
            .onDelete { offsets in
                for index in offsets { context.delete(trips[index]) }
            }
        }
    }
}
```

Concurrency is the sharpest edge in SwiftData. `ModelContainer` and `PersistentIdentifier` are `Sendable`; **model objects and `ModelContext` are not**. Never pass a `Trip` or a context between actors. For background import/sync/batch work, send the container into a `@ModelActor`, create a local context inside it, and pass `PersistentIdentifier` values across the boundary:

```swift
import SwiftData
import Foundation

@ModelActor
actor TripStore {
    @discardableResult
    func importTrip(name: String, destination: String, startDate: Date) throws -> PersistentIdentifier {
        let trip = Trip(name: name, destination: destination, startDate: startDate)
        modelContext.insert(trip)
        try modelContext.save()
        return trip.persistentModelID
    }

    func rename(id: PersistentIdentifier, to newName: String) throws {
        guard let trip = self[id, as: Trip.self] else { return }
        trip.name = newName
        try modelContext.save()
    }

    func deleteAll(matching predicate: Predicate<Trip>) throws {
        try modelContext.delete(model: Trip.self, where: predicate)
        try modelContext.save()
    }
}
```

Build predicates with the `#Predicate` macro and use `FetchDescriptor` with `fetchLimit`/`propertiesToFetch` for large sets:

```swift
let searchText = "beach"
let descriptor = FetchDescriptor<Trip>(
    predicate: #Predicate { trip in
        searchText.isEmpty ? true : trip.destination.localizedStandardContains(searchText)
    },
    sortBy: [SortDescriptor(\.startDate)]
)
```

Test against an in-memory container (`isStoredInMemoryOnly: true`) for fast, isolated tests. Plan migrations with `VersionedSchema` and `SchemaMigrationPlan` from the start. SwiftData class inheritance exists but is iOS 26-only and still has reported edge cases in superclass fetches — reach for it only behind availability and test thoroughly.

## Testing with Swift Testing

Swift Testing is the default; XCTest remains only for UI tests and performance (`XCUIApplication`, `measure`). Use `@Test`/`@Suite`, `#expect` (soft — records and continues) and `#require` (hard — throws and stops). Suites get a fresh instance per test, so put setup in `init` and cleanup in `deinit`. Tests run in parallel by default and support `async`/`await` natively.

```swift
import Testing
import Foundation
@testable import Trips

@Suite("Trip formatting")
struct TripFormattingTests {
    let calendar = Calendar(identifier: .gregorian)

    @Test("Empty search returns all trips")
    func emptySearchReturnsAll() throws {
        let store = TripFilter(trips: .sample)
        let result = try #require(store.filter(search: ""))
        #expect(result.count == store.trips.count)
    }

    // Parameterized: one independent case per argument, run in parallel.
    @Test("Destinations normalize", arguments: [
        ("Beach", "beach"),
        ("LAKE", "lake"),
        ("Woods ", "woods"),
    ])
    func normalize(input: String, expected: String) {
        #expect(input.normalizedDestination == expected)
    }
}
```

Traits worth knowing: `.tags(_:)` for cross-cutting filtering, `.enabled(if:)`/`.disabled("reason")` for conditional runs, `.bug("url")` to link an issue, `.timeLimit(.minutes(1))` (minute granularity only — sub-minute limits are not supported), and `.serialized` on a suite that shares mutable state such as a database. Use `confirmation` for callback/event code (including `expectedCount: 0` to assert something never happens) and `withKnownIssue { }` for a known failure instead of commenting a test out.

```swift
@Test("Login posts a notification")
func loginPostsNotification() async {
    await confirmation("userDidLogin fired") { confirm in
        let token = NotificationCenter.default.addObserver(
            forName: .userDidLogin, object: nil, queue: nil
        ) { _ in confirm() }
        defer { NotificationCenter.default.removeObserver(token) }
        AuthManager().logIn(user: .test)
    }
}

@Suite(.serialized)
struct StoreTests {
    let container: ModelContainer

    init() throws {
        container = try ModelContainer(
            for: Trip.self,
            configurations: ModelConfiguration(isStoredInMemoryOnly: true)
        )
    }

    @Test func insertAndFetch() async throws {
        let store = TripStore(modelContainer: container)
        let id = try await store.importTrip(name: "A", destination: "B", startDate: .now)
        #expect(id != nil)
    }
}
```

## Project generation and packages

Structure the project so the `.xcodeproj` is generated, never committed. XcodeGen reads `project.yml`; keep exact build settings there and split reusable feature code into local SwiftPM packages. Two syntax facts to get right: `options.deploymentTarget` is a *per-platform map*, while a target-level `deploymentTarget` is a *bare string*; and in the advanced `settings` form you must not mix a flat map with `base:`/`configs:` (the flat map is silently ignored).

`project.yml`:

```yaml
name: Trips

options:
  bundleIdPrefix: com.example
  createIntermediateGroups: true
  deploymentTarget:
    iOS: "18.0"

settings:
  base:
    SWIFT_VERSION: "6.0"                 # Swift 6 language mode
    SWIFT_STRICT_CONCURRENCY: complete
    SWIFT_APPROACHABLE_CONCURRENCY: YES
    SWIFT_DEFAULT_ACTOR_ISOLATION: MainActor
    IPHONEOS_DEPLOYMENT_TARGET: "18.0"

packages:
  Dependencies:
    url: https://github.com/pointfreeco/swift-dependencies
    from: 1.0.0
  FeatureKit:
    path: ../Packages/FeatureKit

targets:
  Trips:
    type: application
    platform: iOS
    deploymentTarget: "18.0"
    sources:
      - path: Sources/Trips
    settings:
      base:
        PRODUCT_BUNDLE_IDENTIFIER: com.example.Trips
        GENERATE_INFOPLIST_FILE: YES
        INFOPLIST_KEY_UIApplicationSceneManifest_Generation: YES
        INFOPLIST_KEY_UILaunchScreen_Generation: YES
    dependencies:
      - package: Dependencies
      - package: FeatureKit

  TripsTests:
    type: bundle.unit-test
    platform: iOS
    deploymentTarget: "18.0"
    sources:
      - path: Tests/TripsTests
    settings:
      base:
        GENERATE_INFOPLIST_FILE: YES
    dependencies:
      - target: Trips        # auto-wires TEST_HOST

schemes:
  Trips:
    build:
      targets:
        Trips: all
        TripsTests: [test]
    test:
      gatherCoverageData: true
      coverageTargets:
        - Trips
      targets:
        - name: TripsTests
          parallelizable: true
          randomExecutionOrder: true
    run:
      config: Debug
    archive:
      config: Release
```

Regenerate with `xcodegen generate`. Swift Testing needs no XcodeGen key — it is discovered automatically in a `bundle.unit-test` target. There is no `swiftSettings` key in XcodeGen; set language mode and feature flags through the real Xcode build settings above (`SWIFT_VERSION`, and `OTHER_SWIFT_FLAGS` with `-enable-upcoming-feature <Name>` if needed).

A local package's `Package.swift` is where per-package isolation is declared:

```swift
// swift-tools-version: 6.2
import PackageDescription

let package = Package(
    name: "FeatureKit",
    platforms: [.iOS(.v18)],
    products: [
        .library(name: "FeatureKit", targets: ["FeatureKit"]),
    ],
    targets: [
        .target(
            name: "FeatureKit",
            swiftSettings: [
                .defaultIsolation(MainActor.self),
                .enableUpcomingFeature("NonisolatedNonsendingByDefault"),
                .enableUpcomingFeature("InferIsolatedConformances"),
            ]
        ),
        .testTarget(name: "FeatureKitTests", dependencies: ["FeatureKit"]),
    ]
)
```

For a networking or domain package with no UI, do *not* default it to `MainActor`; leave it nonisolated so callers choose isolation.

## Linting and formatting

Two complementary tools, distinct roles: **SwiftLint** enforces conventions and catches API misuse; **swift-format** (bundled with the Swift toolchain and integrated into Xcode) applies whitespace and layout. Use swift-format for formatting rather than a third-party formatter here, since it ships with the toolchain and matches Xcode's built-in *Format File* command. Run swift-format first, then SwiftLint, so lint checks see formatted code.

Install SwiftLint as an SPM plugin so the version is pinned per repo. (The SwiftLint executable builds with a Swift 6 compiler and its SPM plugins work down to Swift 5.9, so the plugin is safe on this toolchain.)

```swift
// In the app package or a tooling package
.package(url: "https://github.com/SimplyDanny/SwiftLintPlugins", from: "0.65.0")
```

`.swiftlint.yml`:

```yaml
opt_in_rules:
  - empty_count
  - explicit_init
  - first_where
  - force_unwrapping
  - redundant_nil_coalescing
  - toggle_bool
  - unused_import

disabled_rules:
  - todo

included:
  - Sources
  - Tests

excluded:
  - .build
  - "**/Generated"

line_length:
  warning: 120
  error: 160

identifier_name:
  min_length: 2
  excluded: [id, x, y]
```

`.swift-format` (JSON, kept in repo root):

```json
{
  "version": 1,
  "lineLength": 120,
  "indentation": { "spaces": 4 },
  "maximumBlankLines": 1,
  "respectsExistingLineBreaks": true,
  "lineBreakBeforeControlFlowKeywords": false,
  "lineBreakBeforeEachArgument": false
}
```

Commands, forming one coherent setup:

```bash
# Format in place, recursively
swift format --in-place --recursive --configuration .swift-format Sources Tests

# Verify formatting in CI (non-zero exit on drift)
swift format lint --strict --recursive --configuration .swift-format Sources Tests

# Lint (via the SPM plugin)
swift package plugin --allow-writing-to-package-directory swiftlint
swiftlint --strict          # if also installed via Homebrew for local runs

# Generate project, build, test
xcodegen generate
xcodebuild -scheme Trips -destination 'platform=iOS Simulator,name=iPhone 17' test
```

## Anti-patterns to avoid

| Wrong | Why | Right |
|---|---|---|
| `class VM: ObservableObject { @Published var x }` with `@StateObject` | Legacy Combine path; coarse invalidation, extra dependency | `@Observable final class VM { var x }` with `@State`/`@Bindable` |
| `@unchecked Sendable` / `nonisolated(unsafe)` to silence errors | Disables the data-race check that makes Swift 6 safe | Restructure ownership; use actors or `Sendable` value types |
| Marking everything `@MainActor` by hand | Redundant under default main-actor isolation; hides real off-main needs | Keep default isolation on; mark only escapes with `@concurrent` |
| Passing a `@Model` object or `ModelContext` to another actor | Neither is `Sendable`; corruption/crashes | Pass `PersistentIdentifier`; refetch inside a `@ModelActor` |
| `NavigationView { NavigationLink(destination:) }` | Superseded; imperative, poor deep-linking | `NavigationStack` + `NavigationLink(value:)` + `navigationDestination` |
| `XCTAssertEqual` in new tests | XCTest is superseded for unit tests here | `#expect(a == b)` / `try #require(...)` in Swift Testing |
| `@Relationship var items` with no delete rule/inverse | Undefined cascade behavior, orphaned rows | `@Relationship(deleteRule: .cascade, inverse: \Child.parent)` |
| Raising the deployment target to use an iOS 26 API | Breaks the iOS 18 floor | Guard with `if #available(iOS 26, *)` and keep an iOS 18 path |
| `Task { await heavyDecode() }` on the main actor without `@concurrent` | Under default isolation it stays on main and blocks UI | Mark the worker `@concurrent` to run on the global executor |
| Committing `.xcodeproj` | Merge conflicts; drifts from `project.yml` | Generate with `xcodegen generate`; gitignore the project |

## Version & compatibility

| Component | Targeted release line | Notes |
|---|---|---|
| Xcode / SDK | Xcode 26 (iOS 26 SDK) | App Store requires iOS 26 SDK builds; apps uploaded since April 28, 2026 must use Xcode 26+ and an SDK for iOS/iPadOS/tvOS/visionOS/watchOS 26 |
| Swift toolchain | Swift 6.3 | Bundled with Xcode 26 |
| Language mode | Swift 6 (`SWIFT_VERSION = 6.0`) | Complete strict concurrency |
| Minimum deployment target | iOS / iPadOS 18 | Hard floor; guard newer APIs with availability |
| SwiftUI / Observation / SwiftData | iOS 18 baseline | `#Index`/`#Unique`/`#Expression` at 18; `Observations`, SwiftData inheritance, `MainActorMessage`/`AsyncMessage` at 26 |
| Swift Testing | Bundled with toolchain | Default for unit tests; XCTest only for UI/performance |
| SwiftPM | swift-tools-version 6.2 | Per-target `.defaultIsolation`, upcoming-feature flags |
| XcodeGen | 2.46.x | Builds cleanly under Swift 6.x; project generated, not committed |
| SwiftLint | 0.65.x | Pin via SwiftLintPlugins SPM plugin |
| swift-format | Bundled with Swift 6.3 toolchain | Formatter of record; complements SwiftLint |

- **Research date:** 2026-09-05
