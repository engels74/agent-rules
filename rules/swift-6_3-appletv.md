---
type: "agent_requested"
description: "Swift 6 + tvOS 26 Apple TV coding guidelines"
---
# Apple TV Engineering: Swift 6.3, tvOS 26, and the Focus-First Living Room Stack

The user-named "tvOS 18" is superseded: as of August 2026 the current shipping platform is **tvOS 26** (26.5, released May 2026), built with **Xcode 26.x** and **Swift 6.3** (6.3.1 was the stable release on 17 April 2026; Swift 6.4 is still in preview and must not be targeted). Target tvOS 26 with the tvOS 26 SDK, Swift 6 language mode with strict concurrency, SwiftUI as the primary UI layer, SwiftPM for modularization, XcodeGen for the `.xcodeproj`, AVFoundation/AVKit for playback, UniFFI for a Rust core, and SwiftLint 0.65.x for style. This stack is exceptional at one thing above all: a **focus-driven, 10-foot media experience** rendered by SwiftUI and played by AVPlayer, with data-race safety guaranteed at compile time.

The single biggest way agents write wrong-but-plausible code here is **importing iOS/UIKit habits**. Apple TV has no touchscreen, no tap gestures, no swipe-to-dismiss, no `NavigationView` push-by-tap — every interaction flows through the **focus engine** driven by the Siri Remote. Modifiers agents reach for reflexively (`.onTapGesture`, `DragGesture`, `.navigationBarItems`, `.hoverEffect(.lift)` as an interaction, `.searchable` styling assumptions, `.refreshable`) either don't exist, don't fire, or behave differently on tvOS. The second-biggest mistake is fighting Swift 6 concurrency instead of adopting it: Xcode 26 turns on **main-actor-by-default isolation** for new app targets, so most SwiftUI code is already on the main actor and should stay there — reach for concurrency deliberately, not defensively. Optimize for: correct focus structure, `@MainActor`-by-default UI code with explicit `@concurrent` escapes, `AVPlayerViewController` for playback (not a hand-rolled player), and clean actor boundaries around the UniFFI-generated Rust surface.

## Swift 6 strict concurrency and actor isolation

Xcode 26 creates new app targets with **Approachable Concurrency** and **default main-actor isolation** enabled. This is the posture to write for. The two build settings that define it:

```
SWIFT_APPROACHABLE_CONCURRENCY = YES     // umbrella; enables SE-0461 caller-runs behavior
SWIFT_DEFAULT_ACTOR_ISOLATION = MainActor // SE-0466; whole module defaults to @MainActor
```

The rationale, from SE-0466 ("Control default actor isolation inference"): a lot of app code is effectively single-threaded — it "start[s] running on the main actor and stay[s] there unless some part of the code does something concurrent," so under the old defaults "every concurrency diagnostic is necessarily a false positive." Default main-actor isolation removes that noise. Under this model your `App`, `View`s, view models, and plain classes are implicitly `@MainActor`. You do not annotate them. The app is effectively single-threaded until you *explicitly* leave the main actor. This is correct for a TV app: UI, focus, and playback control all belong on the main thread.

Critical insight: **SPM library targets do NOT inherit `defaultIsolation`** — a newly created Swift package has no default isolation set, so its code is `nonisolated` unless you say otherwise. Set it explicitly per target in `Package.swift` (see the SwiftPM section) so a feature module behaves like the app.

Escape to the background *on purpose* with `@concurrent` (Swift 6.2+) for CPU-bound work — image decoding, JSON parsing of a large catalog, thumbnail generation:

```swift
import Foundation

@MainActor
final class CatalogViewModel {
    private(set) var rows: [CatalogRow] = []

    func load() async throws {
        let data = try await api.fetchCatalog()      // network, suspends on main actor safely
        rows = try await Self.decode(data)           // hops to background, hops back
    }

    // Runs off the main actor. Inputs/outputs must be Sendable.
    @concurrent
    private static func decode(_ data: Data) async throws -> [CatalogRow] {
        try JSONDecoder().decode([CatalogRow].self, from: data)
    }
}
```

Use `actor` for mutable shared state that many tasks touch — a download coordinator, an in-memory key cache:

```swift
actor ContentKeyCache {
    private var keys: [String: Data] = [:]
    func key(for id: String) -> Data? { keys[id] }
    func store(_ data: Data, for id: String) { keys[id] = data }
}
```

`Sendable` is the currency of crossing isolation boundaries. Make model types `struct` with `Sendable` stored properties and they are `Sendable` for free. For a reference type that is immutable after init, prefer `final class Foo: Sendable` with only `let` properties. Swift 6.2 also allows `~Sendable` to explicitly opt a type out when it must stay isolated.

`nonisolated` marks members that touch no isolated state and can run anywhere — use it for pure helpers and for protocol conformances like `CustomStringConvertible` on a `@MainActor` type:

```swift
@MainActor
final class PlayerController {
    nonisolated let id: UUID = UUID()          // immutable, safe anywhere
    nonisolated var description: String { "Player" }  // touches no isolated state
    private var player: AVPlayer?              // main-actor isolated
}
```

Anti-habit to drop: do **not** sprinkle `Task { @MainActor in ... }` and `DispatchQueue.main.async` to "get back to the UI." Under default main-actor isolation you are already there. Use `Task { }` only to start async work from a synchronous context; it inherits the enclosing actor.

## SwiftUI app and scene architecture for tvOS

A tvOS app is a plain SwiftUI lifecycle app. There is no `AppDelegate` requirement.

```swift
import SwiftUI

@main
struct LivingRoomApp: App {
    var body: some Scene {
        WindowGroup {
            RootView()
        }
    }
}
```

Top-level navigation on tvOS is a **`TabView`** rendered as a top tab bar, or a sidebar via `.tabViewStyle(.sidebarAdaptable)` (tvOS 18+). Give each tab its own `NavigationStack` so navigation history is per-section and the tab bar persists:

```swift
struct RootView: View {
    var body: some View {
        TabView {
            Tab("Home", systemImage: "house") {
                NavigationStack { HomeView() }
            }
            Tab("Movies", systemImage: "film") {
                NavigationStack { CatalogView(kind: .movies) }
            }
            Tab("Search", systemImage: "magnifyingglass") {
                NavigationStack { SearchView() }
            }
        }
    }
}
```

Use **`NavigationStack`** with value-based destinations — never the deprecated `NavigationView`, and never `NavigationSplitView` with the default/prominent detail style on tvOS (it has a long-standing bug where the content column fails to render; if you must, force `.navigationSplitViewStyle(.balanced)`).

```swift
struct CatalogView: View {
    let kind: CatalogKind
    @State private var items: [Title] = []

    var body: some View {
        ScrollView {
            LazyVGrid(columns: Array(repeating: GridItem(.fixed(320), spacing: 40), count: 5),
                      spacing: 60) {
                ForEach(items) { title in
                    NavigationLink(value: title) { PosterCard(title: title) }
                        .buttonStyle(.card)
                }
            }
            .padding(EdgeInsets(top: 60, leading: 80, bottom: 60, trailing: 80)) // overscan-safe
        }
        .navigationDestination(for: Title.self) { DetailView(title: $0) }
    }
}
```

tvOS 26 adopts Apple's **Liquid Glass** design (WWDC 2025) — shared APIs like `.glassEffect(.regular, in: .capsule)`, `.buttonStyle(.glass)`, and `GlassEffectContainer` are available. Critical hardware caveat: the Liquid Glass redesign is only rendered on **Apple TV 4K (2nd generation and later)**; Apple TV HD and the 1st-gen Apple TV 4K run tvOS 26 but do not get the redesign. Do not assume glass visuals exist on all supported hardware.

## The tvOS focus engine

Focus is the whole game on tvOS. The engine picks the next focusable view by geometry (it traces from the *center* of the currently focused view), hierarchy, and your declared structure. Learn the vocabulary and cooperate with it.

`@FocusState` (tvOS 15+) tracks and programmatically drives focus. Always make it an `Optional` or `Bool`:

```swift
struct LoginView: View {
    enum Field: Hashable { case email, password, submit }
    @FocusState private var focus: Field?
    @State private var email = ""
    @State private var password = ""

    var body: some View {
        VStack(spacing: 40) {
            TextField("Email", text: $email).focused($focus, equals: .email)
            SecureField("Password", text: $password).focused($focus, equals: .password)
            Button("Sign In") { submit() }.focused($focus, equals: .submit)
        }
        .onAppear { focus = .email }        // set initial focus
        .onSubmit { focus = .password }     // move focus on submit
    }
}
```

**`focusSection()`** is the single most important tvOS focus modifier and the one agents omit. It groups focusable descendants into one target so the engine can jump between non-adjacent regions (e.g. from a left sidebar list to a right-side content grid) that pure geometry would otherwise skip. Apply it to each logical cluster — a sidebar, a button row, a card shelf:

```swift
HStack(spacing: 0) {
    SeasonSidebar(seasons: seasons, selection: $selectedSeason)
        .frame(width: 360)
        .focusSection()          // sidebar is one focus target

    EpisodeGrid(episodes: episodes)
        .focusSection()          // grid is another; focus can now cross the gap
}
```

`focusable()` makes an otherwise non-interactive view focusable (an `Image`, a custom card). `prefersDefaultFocus(_:in:)` paired with `@Namespace` and `focusScope(_:)` declares which view grabs focus by default within a scope. `focusEffectDisabled()` opts out of the system focus effect when you draw your own.

```swift
struct Shelf: View {
    @Namespace private var shelf
    var body: some View {
        ScrollView(.horizontal) {
            LazyHStack {
                ForEach(titles) { t in
                    PosterCard(title: t)
                        .prefersDefaultFocus(t == titles.first, in: shelf)
                }
            }
        }
        .focusScope(shelf)
    }
}
```

Respond to remote input with intent-based modifiers, not gestures. **`onMoveCommand`** reports directional swipes on the Siri Remote; **`onExitCommand`** handles the Menu/Back button (use it to dismiss custom overlays):

```swift
CustomOverlay()
    .onMoveCommand { direction in
        switch direction {
        case .left:  selectPrevious()
        case .right: selectNext()
        default: break
        }
    }
    .onExitCommand { dismissOverlay() }   // Menu button
```

Read whether the current view is focused with the `isFocused` environment value (or `.focused` state) to drive appearance:

```swift
struct PosterCard: View {
    let title: Title
    @Environment(\.isFocused) private var isFocused
    var body: some View {
        Image(title.poster)
            .scaleEffect(isFocused ? 1.1 : 1.0)
            .animation(.easeOut(duration: 0.2), value: isFocused)
    }
}
```

## Button styles and card interactions

Use **`.buttonStyle(.card)`** (`CardButtonStyle`, tvOS 14+) for content tiles. It handles the focus lift/raise and the directional parallax motion when the user drags on the Siri Remote — the native "TV feel" — with no code. It applies no padding and lets content go edge to edge.

```swift
Button { play(title) } label: {
    VStack(alignment: .leading) {
        Image(title.poster).resizable().aspectRatio(2/3, contentMode: .fit)
        Text(title.name).font(.caption)
    }
}
.buttonStyle(.card)
```

Do **not** use `.hoverEffect(.lift)` as your interaction model — hover effects are a pointer-device concept (iPadOS/macOS); on tvOS the *focus* engine provides the lift. Use `.buttonStyle(.card)` or `.borderedProminent` for standard buttons, and `.bordered`/`.plain` where appropriate. Context menus work on tvOS via long-press (hold Select):

```swift
PosterCard(title: title)
    .contextMenu {
        Button("Add to Watchlist") { add(title) }
        Button("Mark as Watched") { markWatched(title) }
    }
```

## AVFoundation and AVKit playback

For full-screen video, use **`AVPlayerViewController`** (from AVKit), not a hand-rolled `AVPlayer` + `AVPlayerLayer`. Apple explicitly recommends it for tvOS: it provides the transport bar, Info panel, subtitle/audio selection, chapter navigation, and interstitial UI for free. Wrap it for SwiftUI:

```swift
import SwiftUI
import AVKit

struct PlayerView: UIViewControllerRepresentable {
    let player: AVPlayer

    func makeUIViewController(context: Context) -> AVPlayerViewController {
        let vc = AVPlayerViewController()
        vc.player = player
        return vc
    }
    func updateUIViewController(_ vc: AVPlayerViewController, context: Context) {}
}
```

For a modern content pipeline, build the item from an `AVURLAsset` and load properties asynchronously (`AVAsset` synchronous property access is deprecated — use `load(_:)`):

```swift
func makePlayerItem(url: URL) async throws -> AVPlayerItem {
    let asset = AVURLAsset(url: url)
    _ = try await asset.load(.isPlayable, .duration)
    return AVPlayerItem(asset: asset)
}
```

**Interstitials (ads):** on tvOS, set `AVPlayerItem.interstitialTimeRanges` (or let them be synthesized from HLS `EXT-X-DATERANGE`). AVKit draws dots on the timeline and calls the delegate as playback enters/exits each range — use this to enforce business rules or capture analytics:

```swift
final class PlaybackCoordinator: NSObject, AVPlayerViewControllerDelegate {
    func playerViewController(_ pvc: AVPlayerViewController,
                             willPresent interstitial: AVInterstitialTimeRange) {
        pvc.requiresLinearPlayback = true   // block scrubbing during the ad
    }
    func playerViewController(_ pvc: AVPlayerViewController,
                             didPresent interstitial: AVInterstitialTimeRange) {
        pvc.requiresLinearPlayback = false
    }
}
```

Critical gotcha: **`requiresLinearPlayback` is the only API that disables the ±10s skip** on the Siri Remote. `isSkipForwardEnabled`/`isSkipBackwardEnabled` only affect non-default skipping behaviors and will NOT stop scrubbing — agents reach for them and are surprised.

Customize the transport bar with `transportBarCustomMenuItems` (add `UIMenu`/`UIAction` entries) and supply metadata via `AVPlayerItem.externalMetadata` and chapter markers via `navigationMarkerGroups`.

**Now Playing / background audio:** for audio-only or PiP-continued playback, configure the audio session and populate the Now Playing info + remote commands. On tvOS the audio category enables background audio:

```swift
import AVFAudio
import MediaPlayer

try AVAudioSession.sharedInstance().setCategory(.playback, mode: .moviePlayback)
try AVAudioSession.sharedInstance().setActive(true)

let center = MPRemoteCommandCenter.shared()
center.playCommand.addTarget { _ in player.play(); return .success }
center.pauseCommand.addTarget { _ in player.pause(); return .success }

MPNowPlayingInfoCenter.default().nowPlayingInfo = [
    MPMediaItemPropertyTitle: title.name,
    MPMediaItemPropertyPlaybackDuration: title.duration,
    MPNowPlayingInfoPropertyElapsedPlaybackTime: 0
]
```

Requires the **Audio, AirPlay, and Picture in Picture** background mode in the target's capabilities/Info.plist (`UIBackgroundModes` → `audio`).

## FairPlay Streaming and HLS key delivery

Use **`AVContentKeySession`** (the modern API, since iOS/tvOS 11) for FairPlay, not the older `AVAssetResourceLoaderDelegate` path. `AVContentKeySession` decouples key loading from the playback lifecycle and is the recommended approach for new code; the resource-loader path is the legacy pattern agents copy from old blog posts.

```swift
import AVFoundation

final class ContentKeyManager: NSObject, AVContentKeySessionDelegate {
    let session = AVContentKeySession(keySystem: .fairPlayStreaming)
    private let queue = DispatchQueue(label: "com.example.fairplay")

    override init() {
        super.init()
        session.setDelegate(self, queue: queue)
    }

    func prepareToPlay(_ asset: AVURLAsset) {
        session.addContentKeyRecipient(asset)
    }

    func contentKeySession(_ session: AVContentKeySession,
                           didProvide keyRequest: AVContentKeyRequest) {
        Task { await handle(keyRequest) }
    }

    private func handle(_ keyRequest: AVContentKeyRequest) async {
        do {
            let contentID = (keyRequest.identifier as? String) ?? ""
            let idData = Data(contentID.utf8)
            let appCert = try await fetchApplicationCertificate()
            let spc = try await keyRequest.makeStreamingContentKeyRequestData(
                forApp: appCert, contentIdentifier: idData)
            let ckc = try await fetchContentKey(spc: spc, contentID: contentID) // your KSM
            let response = AVContentKeyResponse(fairPlayStreamingKeyResponseData: ckc)
            keyRequest.processContentKeyResponse(response)
        } catch {
            keyRequest.processContentKeyResponseError(error)
        }
    }
}
```

For offline/persistent keys, respond to `AVPersistableContentKeyRequest` and store the result of `persistableContentKey(fromKeyVendorResponse:)`. Gotcha: **do not reuse one `AVContentKeySession` across rapid channel changes** — Apple documents against reuse, and rapid tune-ins cause stale key requests to arrive for a superseded asset. Also, calling `processContentKeyRequest` twice with the same identifier will not re-fire the delegate.

Picture in Picture on tvOS: set `AVPlayerViewController.allowsPictureInPicturePlayback = true`. FairPlay content plays over AirPlay out of the box when keys come via the resource-loader path; behavior differs for `AVContentKeySession` (check `canProvidePersistableContentKey` before using a persistent key for AirPlay).

## Top shelf extension

Use the modern **`TVTopShelfContentProvider`** (subclass it), not the deprecated `TVTopShelfProvider` protocol. Add a **TV Top Shelf Extension** target. There are four content styles: `TVTopShelfCarouselContent` (full-screen art/video, the richest), `TVTopShelfSectionedContent`, `TVTopShelfInsetContent`, and details carousel.

```swift
import TVServices

final class ContentProvider: TVTopShelfContentProvider {
    override func loadTopShelfContent(
        completionHandler: @escaping (TVTopShelfContent?) -> Void
    ) {
        Task {
            let featured = try? await MoviesClient.shared.featured()
            guard let featured else { completionHandler(nil); return }

            let items = featured.map { movie -> TVTopShelfSectionedItem in
                let item = TVTopShelfSectionedItem(identifier: movie.id)
                item.title = movie.name
                item.setImageURL(movie.posterURL, for: .screenScale2x)
                item.displayAction = TVTopShelfAction(
                    url: URL(string: "livingroom://title/\(movie.id)")!)
                return item
            }
            let collection = TVTopShelfItemCollection(items: items)
            collection.title = "Featured"
            completionHandler(TVTopShelfSectionedContent(sections: [collection]))
        }
    }
}
```

The extension's `Info.plist` must declare `NSExtensionPointIdentifier = com.apple.tv-top-shelf-provider` and `NSExtensionPrincipalClass = $(PRODUCT_MODULE_NAME).ContentProvider`. Deep-link URLs are handled in the main app via `.onOpenURL`. Call `TVTopShelfContentProvider.topShelfContentDidChange()` when your content updates. Keep the extension lightweight and share data with the app via an App Group.

## SwiftPM: manifest, modularization, and binary targets

Use `swift-tools-version` matching your toolchain floor. Set `defaultIsolation: MainActor` per target (SPM does not inherit the app's isolation) and enable strict concurrency features explicitly:

```swift
// swift-tools-version: 6.1
import PackageDescription

let package = Package(
    name: "LivingRoomKit",
    platforms: [.tvOS(.v26)],
    products: [
        .library(name: "CatalogFeature", targets: ["CatalogFeature"]),
        .library(name: "PlaybackFeature", targets: ["PlaybackFeature"]),
    ],
    dependencies: [
        .package(url: "https://github.com/example/rust-core-swift", from: "1.0.0"),
    ],
    targets: [
        .target(
            name: "CatalogFeature",
            dependencies: ["DesignSystem"],
            swiftSettings: [
                .defaultIsolation(MainActor.self),          // match the app
                .enableUpcomingFeature("NonisolatedNonsendingByDefault"),
            ]
        ),
        .target(name: "DesignSystem"),
        .target(
            name: "PlaybackFeature",
            dependencies: [
                .product(name: "RustCore", package: "rust-core-swift"),
            ]
        ),
        .testTarget(name: "CatalogFeatureTests", dependencies: ["CatalogFeature"]),
    ]
)
```

**Package traits** (SE-0450, "Package traits," review closed November 2024, shipped in Swift 6.1) let a package expose different feature sets per environment — Swift 6.1 added them to enable "different APIs and features for specific environments like Embedded Swift and WebAssembly." They require `swift-tools-version: 6.1` or higher (met by your 6.3 stack). Declare traits and a default set; a consumer enables them via `.package(..., traits:)`:

```swift
let package = Package(
    name: "RustCore",
    traits: [
        "Telemetry",
        .trait(name: "OfflineDownloads", enabledTraits: ["Telemetry"]),
        .default(enabledTraits: ["Telemetry"]),
    ],
    // ...
)

// In the consumer — use .default (singular), matching the shipped API and the Swift.org 6.1 blog:
.package(url: "https://github.com/example/rust-core-swift", from: "1.0.0",
         traits: [.default, "OfflineDownloads"])
```

Guard trait-gated code with plain `#if TraitName`. Traits must be additive (enabling one never removes API); removing a default trait is a SemVer-breaking change. Empty `traits: []` disables all traits including defaults.

**Binary targets** distribute a precompiled XCFramework (see UniFFI below). Reference remotely by URL+checksum, or locally by path during development:

```swift
.binaryTarget(
    name: "RustCoreFFI",
    url: "https://github.com/example/rust-core-swift/releases/download/1.0.0/RustCoreFFI.xcframework.zip",
    checksum: "a1b2c3..." // swift package compute-checksum RustCoreFFI.xcframework.zip
),
// or local:
.binaryTarget(name: "RustCoreFFI", path: "./Artifacts/RustCoreFFI.xcframework"),
```

Bundle resources with `.process("Resources")` or `.copy(...)`; access via `Bundle.module`. Prefer SwiftPM for all dependencies — **not** CocoaPods or Carthage, which are legacy for this stack.

## XcodeGen project.yml

XcodeGen generates the `.xcodeproj` from a checked-in `project.yml`, eliminating merge conflicts. Prefer it over Tuist for this stack unless you need Tuist's caching/generation graph. Regenerate with `xcodegen generate`; do not commit the `.xcodeproj`. A tvOS app with a top shelf extension and local package:

```yaml
name: LivingRoom
options:
  bundleIdPrefix: com.example
  deploymentTarget:
    tvOS: "26.0"
  createIntermediateGroups: true

packages:
  LivingRoomKit:
    path: ./LivingRoomKit

settings:
  base:
    SWIFT_VERSION: "6.0"
    SWIFT_STRICT_CONCURRENCY: complete
    SWIFT_APPROACHABLE_CONCURRENCY: YES
    SWIFT_DEFAULT_ACTOR_ISOLATION: MainActor

targets:
  LivingRoom:
    type: application
    platform: tvOS
    sources: [Sources/App]
    settings:
      PRODUCT_BUNDLE_IDENTIFIER: com.example.livingroom
      TARGETED_DEVICE_FAMILY: "3"      # 3 = Apple TV
    info:
      path: Sources/App/Info.plist
      properties:
        UILaunchScreen: {}
    dependencies:
      - package: LivingRoomKit
        product: CatalogFeature
      - package: LivingRoomKit
        product: PlaybackFeature
      - target: TopShelf

  TopShelf:
    type: tv-app-extension
    platform: tvOS
    sources: [Sources/TopShelf]
    settings:
      PRODUCT_BUNDLE_IDENTIFIER: com.example.livingroom.topshelf
    info:
      path: Sources/TopShelf/Info.plist
      properties:
        NSExtension:
          NSExtensionPointIdentifier: com.apple.tv-top-shelf-provider
          NSExtensionPrincipalClass: $(PRODUCT_MODULE_NAME).ContentProvider
```

`TARGETED_DEVICE_FAMILY = 3` is the Apple TV device family. Use `supportedDestinations`/`destinationFilters` if a target spans platforms.

## UniFFI: wrapping a Rust core safely

UniFFI (`mozilla/uniffi-rs`) generates idiomatic Swift bindings from a Rust crate. Use UniFFI for a shared Rust core exposing a defined API surface; `swift-bridge` is the alternative and is better for tight, hand-tuned Swift↔Rust interop, but UniFFI is the right default for a portable core shared with Android. The workflow: build the Rust static lib for each Apple arch, generate Swift with `uniffi-bindgen-swift` (library mode), assemble an XCFramework, and wrap it in a SwiftPM package.

Add a bindgen binary to the Rust crate:

```rust
// src/bin/uniffi-bindgen.rs
fn main() { uniffi::uniffi_bindgen_swift() }
```

Build, generate bindings, and package (build script sketch):

```bash
# Build for device + simulator (Apple Silicon)
cargo build --release --target aarch64-apple-tvos
cargo build --release --target aarch64-apple-tvos-sim

# Generate Swift sources, headers, and modulemap in library mode
cargo run --bin uniffi-bindgen -- generate \
  --library target/aarch64-apple-tvos/release/librustcore.a \
  --language swift --out-dir bindings

# module.modulemap must be renamed exactly (easy to miss)
mv bindings/RustCoreFFI.modulemap bindings/module.modulemap

# Assemble the XCFramework
xcodebuild -create-xcframework \
  -library target/aarch64-apple-tvos/release/librustcore.a -headers bindings \
  -library target/aarch64-apple-tvos-sim/release/librustcore.a -headers bindings \
  -output RustCoreFFI.xcframework
```

The Swift package has three layers: the `binaryTarget` (the XCFramework), a target holding the generated `.swift` files that links it, and an idiomatic Swift overlay:

```swift
.target(
    name: "RustCore",
    dependencies: ["RustCoreFFI"]   // generated bindings + binary
),
.binaryTarget(name: "RustCoreFFI", path: "./RustCoreFFI.xcframework"),
```

Concurrency is where agents get UniFFI wrong. Generated types are reference types backed by Rust; **do not assume they are `Sendable`**. Treat them as isolated and wrap access in an actor, converting to your own `Sendable` Swift models at the boundary:

```swift
import RustCore

// Rust-owned handle; confine it to an actor.
actor CoreEngine {
    private let inner = RustCore.Engine()          // generated type, not Sendable

    func recommendations(for profile: String) throws -> [Recommendation] {
        // Map generated types into your own Sendable structs before returning.
        try inner.recommend(profile: profile).map(Recommendation.init(core:))
    }
}

struct Recommendation: Sendable, Identifiable {
    let id: String
    let title: String
    init(core: RustCore.Recommendation) {
        self.id = core.id
        self.title = core.title
    }
}
```

Map Rust `Result`/errors: UniFFI generates a Swift `Error`-conforming enum for each Rust error type; catch it and translate to your domain error. UniFFI uses `Arc` for shared ownership, so passing a handle into Rust and back is cheap, but every call crosses the FFI — batch calls rather than looping across the boundary per item.

## Testing: Swift Testing and XCUITest

Use **Swift Testing** (`import Testing`, ships with Xcode 16+) for unit and logic tests. Tests are functions, suites are types, `#expect` and `#require` replace the 40+ `XCTAssert` variants, and tests run in parallel by default.

```swift
import Testing
@testable import CatalogFeature

@Suite("Catalog parsing")
struct CatalogTests {
    @Test func decodesFeatured() throws {
        let json = Data(#"[{"id":"1","name":"Dune"}]"#.utf8)
        let titles = try JSONDecoder().decode([Title].self, from: json)
        let first = try #require(titles.first)   // stop the test if nil
        #expect(first.name == "Dune")
    }

    @Test(arguments: [("movies", 5), ("shows", 3)])
    func rowCount(kind: String, expected: Int) {
        #expect(CatalogLayout(kind: kind).rowCount == expected)
    }
}
```

Each test in a suite gets a fresh instance; put setup in `init()` and teardown in `deinit`. Don't reach for `.serialized` when parallel tests fail — that almost always signals shared mutable state to fix. `@MainActor`-annotate a suite that touches UI-isolated code.

Keep **XCUITest on XCTest** — Swift Testing does not cover UI automation or performance (`XCTMetric`). For tvOS UI tests, drive the focus engine via the remote (`XCUIRemote`):

```swift
import XCTest

final class NavigationUITests: XCTestCase {
    func testPlayFromHome() {
        let app = XCUIApplication()
        app.launch()
        XCUIRemote.shared.press(.right)
        XCUIRemote.shared.press(.select)
        XCTAssertTrue(app.otherElements["PlayerView"].waitForExistence(timeout: 10))
    }
}
```

The two frameworks coexist in one target; never mix `#expect` into an `XCTestCase`.

## SwiftLint

Pin SwiftLint via SPM so every machine and CI runs the same version. The current release is **0.65.0** ("Fresh Folded Fixtures," 27 June 2026); it requires macOS 13+, and its SPM plugins work down to Swift 5.9 while the executable builds with a Swift 6 compiler. Run it as a **build tool plugin** for packages, but for the app target prefer an explicit Xcode **Run Script build phase** or a pre-commit hook — SPM build plugins can't do remote config and add clean-build cost. Never let SwiftLint run before compilation; it is designed to analyze compilable code.

A practical `.swiftlint.yml`:

```yaml
disabled_rules:
  - todo
opt_in_rules:
  - empty_count
  - first_where
  - force_unwrapping
  - explicit_init
  - redundant_type_annotation
  - unused_import
  - private_over_fileprivate
  - sorted_imports
analyzer_rules:
  - unused_declaration
  - unused_import
excluded:
  - .build
  - "**/*.generated.swift"   # UniFFI-generated bindings
  - DerivedData
line_length:
  warning: 120
  error: 200
type_body_length: [300, 400]
identifier_name:
  min_length: 2
  excluded: [id, x, y, vc]
```

Run lint on every build/CI; run the slower analyzer separately against a build log:

```bash
swiftlint lint --strict
xcodebuild -scheme LivingRoom clean build > build.log
swiftlint analyze --compiler-log-path build.log
```

Exclude the UniFFI-generated sources — linting machine-generated code is noise.

## Build, run, and test commands

```bash
# Generate the Xcode project
xcodegen generate

# Build a package
swift build

# Build the app for the tvOS simulator
xcodebuild -scheme LivingRoom \
  -destination 'platform=tvOS Simulator,name=Apple TV 4K (3rd generation)' \
  build

# Run tests
xcodebuild test -scheme LivingRoom \
  -destination 'platform=tvOS Simulator,name=Apple TV 4K (3rd generation)'

# Compute a binary target checksum
swift package compute-checksum RustCoreFFI.xcframework.zip
```

## Performance and memory on Apple TV

The Apple TV device is memory-constrained relative to its 4K output. Extensions (top shelf) run in a tight memory budget — keep them minimal. For catalog UIs:

- Use `LazyVGrid`/`LazyHStack` inside `ScrollView` so off-screen cells aren't realized. A `VStack` of hundreds of posters will blow memory.
- Downsample images to display size before handing them to SwiftUI — decoding full-resolution artwork for a 320pt card wastes megabytes each. Load remote posters at the target size.
- Respect the **overscan-safe area**: per Apple's tvOS layout guidance, inset primary content by **60 points top and bottom and 80 points on the sides** so content isn't clipped on TVs that overscan.
- Prefer a single `AVPlayer`/`AVPlayerViewController` and swap items; don't hold multiple decoded video pipelines alive.
- Cap concurrent artwork prefetch; a shelf that eagerly loads 200 images stalls scrolling.

## Anti-patterns to avoid

| Wrong (iOS/adjacent habit) | Right (tvOS 26 / Swift 6) |
| --- | --- |
| `.onTapGesture { }` / `DragGesture()` for interaction | `Button` + focus engine; `onMoveCommand`/`onExitCommand` for remote |
| `NavigationView` | `NavigationStack` with `navigationDestination` |
| `.hoverEffect(.lift)` as the interaction | `.buttonStyle(.card)` — focus provides the lift |
| Omitting `.focusSection()` on sidebars/rows | `focusSection()` per logical cluster so focus can cross gaps |
| `isSkipForwardEnabled = false` to block skipping | `requiresLinearPlayback = true` |
| Hand-rolled `AVPlayerLayer` UI | `AVPlayerViewController` (transport bar, subtitles, interstitials free) |
| `AVAssetResourceLoaderDelegate` for new FairPlay | `AVContentKeySession` + `AVContentKeySessionDelegate` |
| Deprecated `TVTopShelfProvider` | `TVTopShelfContentProvider` subclass |
| `DispatchQueue.main.async` to update UI | Already `@MainActor` under default isolation; just assign |
| Treating UniFFI types as `Sendable` | Confine to an `actor`; map to `Sendable` structs at the boundary |
| `VStack` of all posters | `LazyVGrid`/`LazyHStack` in a `ScrollView` |
| CocoaPods / Carthage | SwiftPM |
| Blocking `swiftc` before build with SwiftLint | Lint compilable code; run as plugin/Run Script/pre-commit |
| Mixing `#expect` into `XCTestCase` | Swift Testing for logic; XCTest only for XCUITest/perf |
| `AVAsset` synchronous property access | `try await asset.load(.isPlayable, .duration)` |

## Version & compatibility

| Component | Target | Notes |
| --- | --- | --- |
| Swift | 6.3 (6.3.1 stable, Apr 2026) | Swift 6.4 is preview-only — do not target |
| Xcode | 26.x | ships Swift 6.3; enables approachable concurrency for new targets |
| tvOS | 26 (26.5, May 2026) | supersedes "tvOS 18"; Liquid Glass only on Apple TV 4K 2nd-gen+ |
| Concurrency | SE-0466 default isolation, SE-0461 caller-runs | `SWIFT_DEFAULT_ACTOR_ISOLATION=MainActor`, `SWIFT_APPROACHABLE_CONCURRENCY=YES` |
| SwiftPM traits | SE-0450, Swift 6.1 | `swift-tools-version: 6.1`+; use `.default` (singular) |
| Focus API | `@FocusState` tvOS 15+, `focusSection()` tvOS 15+, `CardButtonStyle` tvOS 14+ | all current on tvOS 26 |
| AVContentKeySession | iOS/tvOS 11+ | modern FairPlay path |
| Swift Testing | Xcode 16+ | unit/logic; XCUITest stays on XCTest |
| SwiftLint | 0.65.0 (Jun 2026) | SPM-pinned; requires macOS 13+ |
| XcodeGen | current | `xcodegen generate`; don't commit `.xcodeproj` |
| UniFFI | mozilla/uniffi-rs, `uniffi-bindgen-swift` library mode | XCFramework + SwiftPM binary target |

- **Research date:** August 22, 2026
- **Research basis:** current official docs, release notes, specifications, changelogs, and primary repositories.
