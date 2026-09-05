---
type: "agent_requested"
description: "Swift 6 tvOS (SwiftUI + AVKit + UniFFI) coding guidelines"
---
# Swift 6.3 on tvOS: SwiftUI, AVKit, and a Rust Core over UniFFI

This stack builds a native Apple TV app: a SwiftUI interface driven by the tvOS focus engine, media playback through AVKit, and shared business logic compiled from Rust and surfaced to Swift by UniFFI, all assembled with an XcodeGen-generated project over Swift Package Manager. It is exceptional at delivering a fluid 10-foot media experience with a small, testable Swift layer sitting on a portable Rust core. Optimize for: the focus-driven navigation model (nothing is "tapped" — everything is *focused* then selected), main-actor-by-default concurrency with narrow, deliberate escapes to the background, a clean `Sendable` boundary between the Rust core and SwiftUI, and disciplined resource cleanup around `AVPlayer`.

The biggest ways agents write wrong-but-plausible code here come from importing iOS/phone habits: reaching for gestures, sheets, and `TextField`-heavy forms that do not fit tvOS; assuming a writable `Documents` directory (tvOS has none — only a purgeable `Caches`); wrapping every `nonisolated` async function expecting it to hop to the background (under approachable concurrency it now runs on the caller's actor unless marked `@concurrent`); marking UniFFI-generated types `@unchecked Sendable` to silence the compiler; and treating `AVPlayer` KVO/time observers as fire-and-forget instead of owning their teardown. Write code that assumes the main actor, escapes it on purpose, and cleans up after itself.

## Toolchain, language mode, and deployment target

Three version axes are independent and must not be conflated:

- **Toolchain / SDK:** Swift 6.3, shipped in Xcode 26.4 with the tvOS 26 SDK. Build against the current SDK.
- **Language mode:** Swift 6 (`-swift-version 6` / `.swiftLanguageMode(.v6)`). This turns data-race diagnostics into errors.
- **Minimum deployment target:** tvOS 18. This is a floor to build on, not a value to bump. Newer SDK APIs (anything gated to tvOS 26) are usable only behind an `if #available(tvOS 26, *)` check; code that supports tvOS 18 behind such a check is current code, not legacy.

Adopt **approachable concurrency**: default every app/UI module to `@MainActor` isolation and escape deliberately. This is the modern default for app targets and eliminates the annotation noise that made earlier Swift 6 adoption painful.

A note on the new performance types: `InlineArray` and `Span` (introduced with the 6.2 standard library) are attractive for FFI buffer handling, but on Apple platforms their availability floor is tvOS 26. With a tvOS 18 minimum they require an availability check, so reserve them for genuinely hot paths and gate them; do not scatter them through ordinary view or model code.

## Project generation with XcodeGen

`project.yml` is the single source of truth; the `.xcodeproj` is generated and git-ignored. Run `xcodegen generate` after any change to the spec or after adding files. Set the Swift 6 language mode and approachable-concurrency build settings here so they apply to the app target uniformly.

```yaml
# project.yml
name: LivingRoom
options:
  bundleIdPrefix: com.example.livingroom
  deploymentTarget:
    tvOS: "18.0"
  createIntermediateGroups: true
  xcodeVersion: "26.4"

settings:
  base:
    SWIFT_VERSION: "6.0"                       # language mode, not toolchain
    SWIFT_STRICT_CONCURRENCY: complete
    SWIFT_APPROACHABLE_CONCURRENCY: "YES"
    SWIFT_DEFAULT_ACTOR_ISOLATION: MainActor
    TVOS_DEPLOYMENT_TARGET: "18.0"
    TARGETED_DEVICE_FAMILY: "3"                # 3 = Apple TV
    ENABLE_USER_SCRIPT_SANDBOXING: "YES"

packages:
  LivingRoomCore:
    path: LivingRoomCore                       # local SwiftPM package (Rust core + wrappers)
  SwiftLintPlugins:
    url: https://github.com/SimplyDanny/SwiftLintPlugins
    from: "0.65.1"

targets:
  LivingRoom:
    type: application
    platform: tvOS
    deploymentTarget: "18.0"
    sources:
      - path: Sources
    settings:
      base:
        INFOPLIST_FILE: Sources/Info.plist
        PRODUCT_BUNDLE_IDENTIFIER: com.example.livingroom.app
        ASSETCATALOG_COMPILER_APPICON_NAME: "App Icon & Top Shelf Image"
    dependencies:
      - package: LivingRoomCore
        product: LivingRoomCore
    buildToolPlugins:
      - plugin: SwiftLintBuildToolPlugin
        package: SwiftLintPlugins
    scheme:
      testTargets:
        - LivingRoomTests
      gatherCoverageData: true

  LivingRoomTests:
    type: bundle.unit-test
    platform: tvOS
    deploymentTarget: "18.0"
    sources:
      - path: Tests
    dependencies:
      - target: LivingRoom
```

`TARGETED_DEVICE_FAMILY: "3"` is the tvOS device family; omitting it is a common cause of "unsupported destination" build failures. Keep exact tool versions in the `packages:` block and the version table, not scattered through prose.

## Swift Package manifest and module layout

The app's logic lives in a local SwiftPM package that also hosts the Rust core as a binary target. A newly created SwiftPM package does **not** inherit the app's main-actor default or approachable-concurrency flags — you must set them per target. Set `defaultIsolation` for UI-facing targets; leave a pure networking or core-logic target `nonisolated` if it is genuinely off-main.

```swift
// swift-tools-version: 6.3
import PackageDescription

let package = Package(
    name: "LivingRoomCore",
    platforms: [.tvOS(.v18)],
    products: [
        .library(name: "LivingRoomCore", targets: ["LivingRoomCore"]),
    ],
    targets: [
        // Prebuilt Rust core, packaged as an XCFramework (see UniFFI section).
        .binaryTarget(
            name: "RustCoreFFI",
            path: "Artifacts/RustCoreFFI.xcframework"
        ),
        // Generated UniFFI Swift bindings; kept nonisolated because the Rust
        // core is not tied to the main actor.
        .target(
            name: "RustCore",
            dependencies: ["RustCoreFFI"],
            path: "Sources/RustCore",
            swiftSettings: [
                .swiftLanguageMode(.v6),
            ]
        ),
        // UI-facing view models and helpers default to the main actor.
        .target(
            name: "LivingRoomCore",
            dependencies: ["RustCore"],
            swiftSettings: [
                .swiftLanguageMode(.v6),
                .defaultIsolation(MainActor.self),
            ]
        ),
        .testTarget(
            name: "LivingRoomCoreTests",
            dependencies: ["LivingRoomCore"],
            swiftSettings: [
                .swiftLanguageMode(.v6),
                .defaultIsolation(MainActor.self),
            ]
        ),
    ]
)
```

`swift build` and `swift test` work for the pure-Swift and Rust-binding targets from the command line, but anything that links UIKit/AVKit/SwiftUI for the tvOS SDK must be built and tested through `xcodebuild` with a tvOS destination — `swift test` on the host macOS cannot run tvOS-only code.

## Concurrency: main-actor by default, escape on purpose

Under approachable concurrency the mental model inverts: **everything is `@MainActor` unless you say otherwise**, and you prove only the few places you leave the main actor. Three rules cover almost every case:

- Leave UI types, view models, and view code implicit (they are main-actor by default).
- Mark CPU-bound or blocking work `@concurrent` to push it to the global executor. A bare `nonisolated async` function no longer hops to the background — it runs on the *caller's* actor (`nonisolated(nonsending)`), so `@concurrent` is now the explicit escape hatch.
- Use an `actor` for mutable state that is shared across tasks and is genuinely off-main, such as a playback-session coordinator.

```swift
import Foundation

// Main-actor by default (module default isolation). No annotation needed.
@Observable
final class CatalogViewModel {
    private(set) var rows: [Shelf] = []
    private(set) var loadState: LoadState = .idle

    enum LoadState: Equatable { case idle, loading, loaded, failed(String) }

    private let core: CatalogService   // from the Rust core, async API

    init(core: CatalogService) { self.core = core }

    func load() async {
        loadState = .loading
        do {
            // `core.fetchShelves()` is async; awaiting it does not block the
            // main actor. Decoding/sorting heavy work lives behind @concurrent.
            let fetched = try await core.fetchShelves()
            rows = try await Self.sort(fetched)
            loadState = .loaded
        } catch is CancellationError {
            loadState = .idle                 // task was cancelled; not an error
        } catch {
            loadState = .failed(String(describing: error))
        }
    }

    // Deliberately off-main: sorting a large catalog should not touch the UI thread.
    @concurrent
    private static func sort(_ shelves: [Shelf]) async throws -> [Shelf] {
        try Task.checkCancellation()
        return shelves.sorted { $0.rank < $1.rank }
    }
}
```

Never spin up a `Task.detached` just to "get off the main thread" — it discards actor context and priority. Prefer `@concurrent` functions or an actor. Reserve `Task.detached` for the rare case where you explicitly want no inherited context.

## SwiftUI for the living room

tvOS UI is built around the **focus engine**, not touch. There are no tap targets; the user moves focus with the Siri Remote and selects. Design every screen so focus can travel in straight horizontal/vertical lines, and group related controls with `focusSection` when their geometry would otherwise strand focus.

### Focus, navigation, and the remote

```swift
import SwiftUI

struct HomeView: View {
    @State private var model: CatalogViewModel
    @FocusState private var focusedShelf: Shelf.ID?

    init(model: CatalogViewModel) { _model = State(initialValue: model) }

    var body: some View {
        NavigationStack {
            ScrollView {
                LazyVStack(alignment: .leading, spacing: 48) {
                    ForEach(model.rows) { shelf in
                        ShelfRow(shelf: shelf)
                            .focusSection()               // keep focus inside the row
                            .focused($focusedShelf, equals: shelf.id)
                    }
                }
                .scrollTargetLayout()
            }
            .scrollTargetBehavior(.viewAligned)
            .task { await model.load() }                   // auto-cancels on disappear
            .onExitCommand {                                // Menu button on the remote
                focusedShelf = model.rows.first?.id
            }
        }
    }
}

struct ShelfRow: View {
    let shelf: Shelf

    var body: some View {
        VStack(alignment: .leading) {
            Text(shelf.title).font(.headline)
            ScrollView(.horizontal) {
                LazyHStack(spacing: 40) {
                    ForEach(shelf.items) { item in
                        NavigationLink(value: item) {
                            PosterCard(item: item)
                        }
                        .buttonStyle(.card)                 // tvOS focus lift + parallax
                    }
                }
                .scrollTargetLayout()
            }
        }
    }
}
```

Key tvOS specifics:

- `.buttonStyle(.card)` gives the native focus lift and parallax for poster art; `.borderless` is the default text-button treatment. Do not hand-build focus scaling with `scaleEffect` — the system does it correctly and consistently.
- `.task` ties an async load to the view's lifetime and cancels automatically when the view disappears; handle `CancellationError` as a non-error.
- `onExitCommand` handles the Menu button, `onPlayPauseCommand` the Play/Pause button, and `onMoveCommand` directional swipes when you need raw input (rare — prefer focus-driven navigation).
- Use `defaultFocus(_:_:)` to place initial focus on the most likely control; without it, focus can land unpredictably.

### TabView and sidebars

On tvOS 18 the collapsible sidebar is available with `.tabViewStyle(.sidebarAdaptable)` on a `TabView` built from the `Tab`/`TabSection` API. There is a verified behavioral gotcha: with more than seven tabs, or when `TabSection` groups are present, the remote's back-swipe may fail to reveal the collapsed sidebar. Keep primary tabs at seven or fewer and test sidebar reveal when you introduce sections.

```swift
struct RootView: View {
    var body: some View {
        TabView {
            Tab("Home", systemImage: "house") { HomeView(model: .live) }
            Tab("Movies", systemImage: "film") { MoviesView() }
            Tab("Search", systemImage: "magnifyingglass", role: .search) { SearchView() }
        }
        .tabViewStyle(.sidebarAdaptable)
    }
}
```

What to avoid from the phone playbook: swipe/drag gestures, `.sheet`/`.popover` as primary navigation (present full-screen `NavigationStack` destinations instead), and dense `TextField` forms — text entry on tvOS routes through the system keyboard and should be minimized. Account for **overscan**: respect the safe area and never place essential content at the extreme screen edges.

## Video playback with AVKit

Two integration paths, pick by need:

- **`VideoPlayer` (SwiftUI):** the fastest path for simple playback. It renders an `AVPlayer` inline but exposes almost no transport-bar customization.
- **`AVPlayerViewController` via `UIViewControllerRepresentable`:** required for tvOS transport-bar customization (`transportBarCustomMenuItems`, `contextualActions`, `infoViewActions`, `customInfoViewControllers`), interstitials, and fine control. This is the default for a real media app.

```swift
import SwiftUI
import AVKit

struct PlayerScreen: UIViewControllerRepresentable {
    let player: AVPlayer
    let customMenu: [UIMenuElement]

    func makeUIViewController(context: Context) -> AVPlayerViewController {
        let controller = AVPlayerViewController()
        controller.player = player
        controller.transportBarCustomMenuItems = customMenu
        controller.allowsPictureInPicturePlayback = true
        controller.delegate = context.coordinator
        return controller
    }

    func updateUIViewController(_ controller: AVPlayerViewController, context: Context) {
        controller.transportBarCustomMenuItems = customMenu
    }

    func makeCoordinator() -> Coordinator { Coordinator() }

    final class Coordinator: NSObject, AVPlayerViewControllerDelegate {}
}
```

### Owning player state with an actor and observing time safely

`AVPlayer`'s periodic time observer and KVO must be *removed by the same owner that added them*, on the same object, before the player is torn down or its item is replaced — failure to do so is a well-documented crash on tvOS. Wrap observation in an `AsyncStream` whose `onTermination` handler removes the observer, so cancellation and cleanup are one path.

```swift
import AVFoundation

@MainActor
final class PlaybackController {
    let player = AVPlayer()
    private var timeObserver: Any?

    /// Emits the current playback time roughly twice a second. The observer is
    /// installed when iteration begins and removed when the stream terminates
    /// (consumer cancels, or the task is cancelled).
    func times() -> AsyncStream<CMTime> {
        AsyncStream { continuation in
            let interval = CMTime(seconds: 0.5, preferredTimescale: 600)
            let token = player.addPeriodicTimeObserver(
                forInterval: interval, queue: .main
            ) { time in
                continuation.yield(time)
            }
            self.timeObserver = token
            continuation.onTermination = { [weak self] _ in
                // Hop back to the main actor to remove the observer safely.
                Task { @MainActor in self?.removeTimeObserver() }
            }
        }
    }

    private func removeTimeObserver() {
        if let token = timeObserver {
            player.removeTimeObserver(token)
            timeObserver = nil
        }
    }

    func replaceItem(with item: AVPlayerItem) {
        // Remove observers BEFORE swapping the item to avoid KVO teardown crashes.
        removeTimeObserver()
        player.replaceCurrentItem(with: item)
    }

    deinit { /* AVPlayer released; token invalidated with it. */ }
}
```

For observing item status prefer `AVPlayerItem.publisher(for:)` bridged to `.values`, but attach it to the item you actually keep a reference to; a `publisher(for:)` on `AVPlayer`'s `currentItem` keypath can silently stop delivering after `replaceCurrentItem`.

### HLS, DRM, audio, and background policy

- HLS is the delivery format; for protected content use FairPlay via `AVContentKeySession`. The key exchange below is an excerpt — the certificate and CKC come from your license server.

```swift
// Excerpt — license-server calls omitted.
import AVFoundation

final class ContentKeyManager: NSObject, AVContentKeySessionDelegate {
    let session = AVContentKeySession(keySystem: .fairPlayStreaming)

    func attach(to asset: AVURLAsset) {
        session.setDelegate(self, queue: .main)
        session.addContentKeyRecipient(asset)
    }

    func contentKeySession(_ session: AVContentKeySession,
                           didProvide request: AVContentKeyRequest) {
        // 1. fetch app certificate, 2. request.makeStreamingContentKeyRequestData,
        // 3. POST SPC to license server, 4. respond with AVContentKeyResponse(fairPlayStreamingKeyResponseData:).
    }
}
```

- Set an `AVAudioSession` category of `.playback` on tvOS so audio behaves correctly and continues appropriately; do this once before playback starts.
- Picture in Picture is available on tvOS via `allowsPictureInPicturePlayback`; use `AVPlayer.audiovisualBackgroundPlaybackPolicy` to express whether playback should continue when the app is backgrounded.
- Surface item and player errors from `AVPlayerItem.status == .failed` (read `item.error`) and from `AVPlayer.error`; do not assume a URL that loads in the simulator streams on device.

## The Rust core via UniFFI

The Rust core is compiled to static libraries, wrapped by UniFFI-generated Swift, and shipped as an XCFramework binary target. As of the current release, `aarch64-apple-tvos` (device) and `aarch64-apple-tvos-sim` (Apple-silicon simulator) are Tier 2 Rust targets (without host tools) distributed through `rustup` — you can `rustup target add` them and build with stable Cargo, no nightly `-Zbuild-std` required. Only the legacy `x86_64-apple-tvos` Intel simulator remains Tier 3 and would need a `-Zbuild-std` build. App Store submissions require bitcode-free static frameworks, which is what this pipeline produces.

Prefer **proc-macro** bindings (`#[uniffi::export]`, `uniffi::setup_scaffolding!()`) over the older UDL file; it needs no separate interface definition and is what `uniffi-bindgen-swift` operates on in library mode.

```rust
// src/lib.rs
uniffi::setup_scaffolding!();

#[derive(uniffi::Record)]
pub struct Shelf { pub id: String, pub title: String, pub rank: u32 }

#[derive(uniffi::Error, Debug, thiserror::Error)]
pub enum CatalogError {
    #[error("network unavailable")]
    Network,
    #[error("decode failed: {msg}")]
    Decode { msg: String },
}

#[derive(uniffi::Object)]
pub struct CatalogService { /* ... */ }

#[uniffi::export]
impl CatalogService {
    #[uniffi::constructor]
    pub fn new() -> Self { Self { /* ... */ } }

    // async fn surfaces as Swift `async throws`, driven by Swift's executor.
    pub async fn fetch_shelves(&self) -> Result<Vec<Shelf>, CatalogError> {
        Ok(vec![])
    }
}
```

Build and package (device + arm64 simulator):

```bash
# One-time
rustup target add aarch64-apple-tvos aarch64-apple-tvos-sim

# Build static libs
cargo build --release --target aarch64-apple-tvos
cargo build --release --target aarch64-apple-tvos-sim

# Generate Swift bindings, headers, and an XCFramework-compatible modulemap
cargo run -p uniffi-bindgen-swift -- \
  target/aarch64-apple-tvos/release/librust_core.a \
  Generated \
  --swift-sources --headers --modulemap \
  --module-name RustCoreFFI --modulemap-filename module.modulemap

# Assemble the XCFramework consumed by the SwiftPM binary target
xcodebuild -create-xcframework \
  -library target/aarch64-apple-tvos/release/librust_core.a -headers Generated \
  -library target/aarch64-apple-tvos-sim/release/librust_core.a -headers Generated \
  -output Artifacts/RustCoreFFI.xcframework
```

Move the generated `*.swift` into the `RustCore` SwiftPM target's sources; keep the `module.modulemap` and header inside the XCFramework's header directory. The generated modulemap must be named `module.modulemap` or Swift will not find the module.

**Sendable across the boundary is the sharpest gotcha.** UniFFI has partial Swift 6 support: most generated types conform to `Sendable`, but async-returning generated code is known *not* to conform yet (tracked upstream). Do not paper over this with `@unchecked Sendable` on generated types — regenerate against a current UniFFI, and where an async Rust object triggers a "sending self-isolated value" error, confine that object to a single actor (e.g. hold it inside a `@MainActor` view model, or an `actor`) rather than passing it across isolation domains. When you implement a UniFFI foreign trait (callback interface) in Swift, the generated protocol requires `Sendable`, so your implementation must genuinely be safe to call from any thread. Async Rust functions surface as Swift `async` and are cancellable through normal Swift `Task` cancellation.

## Data and persistence

Networking is `URLSession`'s async API with `Codable`; do not add a third-party HTTP client without a concrete need.

```swift
struct FeedClient {
    let session: URLSession = .shared

    func shelves(from url: URL) async throws -> [ShelfDTO] {
        let (data, response) = try await session.data(from: url)
        guard let http = response as? HTTPURLResponse, (200..<300).contains(http.statusCode) else {
            throw URLError(.badServerResponse)
        }
        return try JSONDecoder().decode([ShelfDTO].self, from: data)
    }
}
```

**tvOS storage is severely constrained and this drives architecture.** There is no persistent `Documents` directory. Only the `Caches` and `tmp` directories are writable, and the system may purge them at any time while the app is not running — never treat cached files as durable. `UserDefaults` is the only local persistent store and per Apple's App Programming Guide for tvOS your app "can only access 500 KB of persistent storage that is local to the device (using the `NSUserDefaults` class)"; the system posts `sizeLimitExceededNotification` as a warning at 512 KB and **terminates the app** once user-defaults storage reaches 1 MB. Use it for small preferences only; the app bundle is read-only. For anything larger or that must survive, use iCloud key-value store (`NSUbiquitousKeyValueStore`) or CloudKit / your own backend, and keep secrets in the Keychain. Design the app to re-fetch or re-derive anything it stores locally.

## Linting and formatting

Two complementary tools, both current defaults: **SwiftLint** for style/convention rules and **swift-format** (bundled with the toolchain, invoked as `swift format` with a space) for deterministic formatting. Prefer Apple's swift-format over the third-party SwiftFormat here — it ships with the toolchain and needs no extra dependency. Run SwiftLint as the SwiftPM `SwiftLintBuildToolPlugin` (from the SimplyDanny `SwiftLintPlugins` package, which keeps a pinned binary in sync with each SwiftLint release) so it lints on every build; fall back to a Run Script phase only if a setup requires `--config` across targets.

```yaml
# .swiftlint.yml
opt_in_rules:
  - empty_count
  - first_where
  - explicit_init
  - toggle_bool
  - unneeded_parentheses_in_closure_argument
analyzer_rules:
  - unused_declaration
  - unused_import
disabled_rules:
  - trailing_whitespace   # handled by swift-format
line_length:
  warning: 120
  error: 200
  ignores_comments: true
  ignores_urls: true
identifier_name:
  min_length: { warning: 2 }
  excluded: [id, x, y]
included:
  - Sources
excluded:
  - .build
  - Generated               # do not lint UniFFI-generated Swift
```

```json
// .swift-format
{
  "version": 1,
  "lineLength": 120,
  "indentation": { "spaces": 4 },
  "maximumBlankLines": 1,
  "respectsExistingLineBreaks": true,
  "lineBreakBeforeEachArgument": true
}
```

Commands:

```bash
swift format lint --strict --recursive Sources          # format check
swift format --in-place --recursive Sources             # apply formatting
swiftlint lint --strict                                  # style lint, warnings fail
swiftlint analyze --strict                               # analyzer rules (needs a compile log)
swiftlint --fix                                          # autocorrect fixable violations
```

Exclude the `Generated` UniFFI directory from both tools — machine-generated bindings should not be reformatted or linted. SwiftLint analyzes only compilable code, so run `--fix` on code that already builds.

## Testing

**Swift Testing** (`import Testing`) is the default for unit and logic tests; keep XCTest only where you still need `XCUIApplication` UI automation. Design for testability by injecting the Rust-backed services behind protocols so tests use fakes rather than the FFI.

```swift
import Testing
@testable import LivingRoomCore

@Suite("Catalog loading")
struct CatalogTests {
    @Test("Empty feed yields no shelves")
    func emptyFeed() async throws {
        let vm = CatalogViewModel(core: FakeCatalog(shelves: []))
        await vm.load()
        #expect(vm.rows.isEmpty)
        #expect(vm.loadState == .loaded)
    }

    @Test("Shelves sort by rank", arguments: [
        ([3, 1, 2], [1, 2, 3]),
        ([1], [1]),
    ])
    func sorting(input: [UInt32], expected: [UInt32]) async throws {
        let shelves = input.map { Shelf(id: "\($0)", title: "t", rank: $0) }
        let vm = CatalogViewModel(core: FakeCatalog(shelves: shelves))
        await vm.load()
        #expect(vm.rows.map(\.rank) == expected)
    }

    @Test("Missing feed surfaces a failure")
    func networkError() async throws {
        let vm = CatalogViewModel(core: FailingCatalog())
        await vm.load()
        let failed = try #require(vm.loadState.failureMessage)
        #expect(!failed.isEmpty)
    }
}
```

Use `#expect` for soft assertions (test continues) and `#require` for preconditions that must hold to proceed (throws and stops the test). Use `confirmation` for callback/event expectations, including `expectedCount: 0` to assert an event does *not* fire. Swift Testing runs tests in parallel by default, so keep them independent and free of shared mutable global state. Swift Testing 6.3 adds warning-severity issues (`Issue.record(_:severity:)`) and mid-test cancellation (`try Test.cancel()`) for skipping parameterized arguments.

Run on a tvOS simulator:

```bash
xcodegen generate
xcodebuild test \
  -project LivingRoom.xcodeproj -scheme LivingRoom \
  -destination 'platform=tvOS Simulator,name=Apple TV,OS=latest'
```

## CI commands

One coherent pipeline, in order:

```bash
# 1. Build the Rust core and package the XCFramework
rustup target add aarch64-apple-tvos aarch64-apple-tvos-sim
cargo build --release --target aarch64-apple-tvos
cargo build --release --target aarch64-apple-tvos-sim
./scripts/make-xcframework.sh          # wraps uniffi-bindgen-swift + xcodebuild -create-xcframework

# 2. Generate the Xcode project
xcodegen generate

# 3. Format + lint gates
swift format lint --strict --recursive Sources
swiftlint lint --strict

# 4. Build and test on tvOS
xcodebuild -project LivingRoom.xcodeproj -scheme LivingRoom \
  -destination 'generic/platform=tvOS' build
xcodebuild test -project LivingRoom.xcodeproj -scheme LivingRoom \
  -destination 'platform=tvOS Simulator,name=Apple TV,OS=latest'
```

Pin the Xcode version explicitly in CI; if the runner auto-updates, the local and CI toolchains diverge silently and concurrency diagnostics can differ.

## Anti-patterns to avoid

| Wrong | Why | Right |
|---|---|---|
| Building UI around tap/drag gestures | tvOS has no touch; input is focus + select via the remote | Design focus-driven layouts with `.focusable`, `.focusSection()`, `defaultFocus`, and `.buttonStyle(.card)` |
| Writing app data to `Documents` / the app bundle | tvOS has no persistent `Documents`; the bundle is read-only | Use `Caches`/`tmp` (treat as purgeable), `UserDefaults` (500 KB local limit), iCloud KVS, CloudKit, or Keychain |
| Storing more than ~512 KB in `UserDefaults` | tvOS warns at 512 KB (`sizeLimitExceededNotification`) and terminates the app at 1 MB | Keep only small preferences in `UserDefaults`; move bulk data to iCloud/CloudKit/backend |
| Expecting a bare `nonisolated async` func to run in the background | Under approachable concurrency it runs on the caller's actor (`nonisolated(nonsending)`) | Mark CPU-bound work `@concurrent`, or move shared mutable state into an `actor` |
| `Task.detached { … }` to "get off main" | Discards actor context and priority | Use a `@concurrent` function or an actor with inherited context |
| Marking UniFFI-generated types `@unchecked Sendable` | Hides a real data-race hazard; async generated code isn't `Sendable`-safe yet | Regenerate against current UniFFI; confine async Rust objects to one actor |
| Adding/removing `AVPlayer` time observers or KVO without a single owner | Removing on the wrong object or after item replacement crashes on tvOS | Own the token; remove before `replaceCurrentItem`; tie teardown to `AsyncStream.onTermination` |
| Using `VideoPlayer` then trying to customize the transport bar | SwiftUI `VideoPlayer` exposes almost no transport-bar API | Wrap `AVPlayerViewController` with `UIViewControllerRepresentable` for `transportBarCustomMenuItems` etc. |
| Hand-rolling focus scaling with `scaleEffect`/animations | Fights the system focus engine and looks non-native | Use built-in tvOS button styles that provide the focus lift and parallax |
| Bumping the deployment target to use a tvOS 26 API | The tvOS 18 minimum is a constraint, not a stale value | Gate new APIs behind `if #available(tvOS 26, *)` |
| Setting `SWIFT_VERSION` expecting it to pick the toolchain | `SWIFT_VERSION` selects the *language mode*, not the compiler | Toolchain comes from Xcode; set language mode to `6.0` and mode flags separately |
| Eight-plus tabs (or `TabSection`) with `.sidebarAdaptable` | Verified tvOS 18 bug: back-swipe fails to reveal the collapsed sidebar | Keep primary tabs ≤ 7; verify sidebar reveal when using sections |
| `swift-format` invoked as one hyphenated word inside a Swift build | The bundled toolchain command is `swift format` (with a space) | Call `swift format …`, or `xcrun --find swift-format` for the binary path |

## Version & compatibility

| Component | Targeted line | Notes / availability floor |
|---|---|---|
| Swift toolchain | 6.3 | Ships in Xcode 26.4 (later patches: 6.3.3 in Xcode 26.6) |
| Language mode | Swift 6 | `.swiftLanguageMode(.v6)` / `SWIFT_VERSION = 6.0` |
| Xcode / SDK | 26.4+ (tvOS 26 SDK) | Build against current SDK |
| Minimum deployment target | tvOS 18.0 | Hard constraint; gate tvOS 26 APIs behind `#available` |
| Approachable concurrency | Swift 6.2+ | `defaultIsolation(MainActor.self)`, `nonisolated(nonsending)`, `@concurrent` |
| `InlineArray` / `Span` | stdlib 6.2 | Availability floor tvOS 26 — must be gated at this minimum |
| XcodeGen | 2.46.0 | `project.yml`; `xcodegen generate` |
| SwiftPM tools-version | 6.3 | `.defaultIsolation` available since tools-version 6.2 |
| SwiftLint | 0.65.1 | via `SwiftLintPlugins` 0.65.1 build-tool plugin (SwiftLint dev requires Swift 6.1+) |
| swift-format | 603.0.0 | Bundled with toolchain; run as `swift format` |
| UniFFI (uniffi-rs) | 0.32.0 | Proc-macro bindings; `uniffi-bindgen-swift` |
| Rust tvOS targets | Tier 2 (no host tools) | `aarch64-apple-tvos`, `aarch64-apple-tvos-sim` via `rustup`; stable Cargo, no `-Zbuild-std` |
| Swift Testing | 6.3 | Default for unit tests; XCTest for `XCUIApplication` UI tests |

- **Research date:** 2026-09-05
