---
type: "agent_requested"
description: "Kotlin + Android TV + Jetpack Compose (androidx.tv:tv-material) coding guidelines"
---
# Android TV with Jetpack Compose & tv-material: The Living-Room Kotlin Reference

Android TV development in 2026 is a **Compose-first, focus-first** discipline. The living room has no touchscreen, no scroll gestures, and no mouse — every interaction flows through a D-pad, and the single hardest thing to get right is **focus management**. Build UI from `androidx.tv:tv-material` (TV-tuned Material 3 components with built-in focus scaling, glow, and border indication), lay content out with **standard** `LazyRow`/`LazyColumn`/`LazyVerticalGrid` from `androidx.compose.foundation` plus `Modifier.focusRestorer()`, and play media through **AndroidX Media3 (ExoPlayer)**. Target the latest stable toolchain: Kotlin 2.4, Compose BOM 2026.08.00 (Compose 1.12), AGP 9.x on Gradle 9.x, Java 17 toolchain.

The ways agents write wrong-but-plausible TV code are predictable, and almost all of them come from importing phone-Compose habits: reaching for `androidx.compose.material3.Button`/`Card` instead of `androidx.tv.material3` equivalents; using `Modifier.clickable` with no focus indication; using the **removed** `androidx.tv:tv-foundation` `TvLazyRow`/`TvLazyColumn`; using the **removed** `ImmersiveList`; using **deprecated** `androidx.leanback` fragments; using the **end-of-life** `com.google.android.exoplayer2`; forgetting the `LEANBACK_LAUNCHER` intent category; assuming touch/scroll gestures exist; hardcoding sizes with no overscan margin; and using `kapt` + `kotlinCompilerExtensionVersion` instead of KSP + the Compose compiler Gradle plugin. This document shows the one correct, current way to do each of these.

## Stack snapshot

- **Research date:** August 22, 2026
- **Research basis:** current official docs, release notes, specifications, changelogs, and primary repositories.

| Component | Version | Notes |
|---|---|---|
| Kotlin | 2.4.0 (2.3.20 also current stable) | K2-only; context parameters & explicit backing fields stable |
| Jetpack Compose BOM | 2026.08.00 | Compose core modules 1.12.0; requires compileSdk 37, AGP 9.1.1+ |
| androidx.tv:tv-material | 1.0.1 stable (1.1.0-alpha01 preview) | `tv-foundation` removed; use standard Lazy* + `focusRestorer` |
| androidx.media3 | 1.9.0 stable | `com.google.android.exoplayer2` is end-of-life — never use |
| Navigation 3 (androidx.navigation3) | 1.0.0 stable | Compose-first, state-based; recommended for new code |
| Hilt | 2.56.x (androidx.hilt 1.2.0) | KSP only, never kapt |
| Android Gradle Plugin | 9.x | Built-in Kotlin support; Gradle 9.x; JDK 17 |
| Room | 2.7.x | KSP; KMP-capable |
| Coil | 3.x | Compose + multiplatform image loading |

## Dependencies: version catalog

`gradle/libs.versions.toml` — the single source of truth. Never hardcode versions in module files.

```toml
[versions]
agp = "9.1.1"
kotlin = "2.4.0"
ksp = "2.4.0-2.0.0"
composeBom = "2026.08.00"
tvMaterial = "1.0.1"
media3 = "1.9.0"
hilt = "2.56.2"
hiltNavigationCompose = "1.2.0"
navigation3 = "1.0.0"
lifecycle = "2.9.0"
coil = "3.0.4"
room = "2.7.1"
coroutines = "1.9.0"

[libraries]
# Compose (BOM-managed — no versions on these)
compose-bom = { module = "androidx.compose:compose-bom", version.ref = "composeBom" }
compose-ui = { module = "androidx.compose.ui:ui" }
compose-ui-tooling = { module = "androidx.compose.ui:ui-tooling" }
compose-ui-tooling-preview = { module = "androidx.compose.ui:ui-tooling-preview" }
compose-ui-test-junit4 = { module = "androidx.compose.ui:ui-test-junit4" }
compose-ui-test-manifest = { module = "androidx.compose.ui:ui-test-manifest" }
# TV
androidx-tv-material = { module = "androidx.tv:tv-material", version.ref = "tvMaterial" }
androidx-activity-compose = { module = "androidx.activity:activity-compose", version = "1.13.0" }
# Lifecycle / ViewModel
lifecycle-runtime-compose = { module = "androidx.lifecycle:lifecycle-runtime-compose", version.ref = "lifecycle" }
lifecycle-viewmodel-compose = { module = "androidx.lifecycle:lifecycle-viewmodel-compose", version.ref = "lifecycle" }
# Navigation 3
navigation3-runtime = { module = "androidx.navigation3:navigation3-runtime", version.ref = "navigation3" }
navigation3-ui = { module = "androidx.navigation3:navigation3-ui", version.ref = "navigation3" }
lifecycle-viewmodel-navigation3 = { module = "androidx.lifecycle:lifecycle-viewmodel-navigation3", version.ref = "lifecycle" }
# Media3
media3-exoplayer = { module = "androidx.media3:media3-exoplayer", version.ref = "media3" }
media3-exoplayer-hls = { module = "androidx.media3:media3-exoplayer-hls", version.ref = "media3" }
media3-exoplayer-dash = { module = "androidx.media3:media3-exoplayer-dash", version.ref = "media3" }
media3-ui-compose = { module = "androidx.media3:media3-ui-compose", version.ref = "media3" }
media3-session = { module = "androidx.media3:media3-session", version.ref = "media3" }
# DI
hilt-android = { module = "com.google.dagger:hilt-android", version.ref = "hilt" }
hilt-compiler = { module = "com.google.dagger:hilt-compiler", version.ref = "hilt" }
hilt-navigation-compose = { module = "androidx.hilt:hilt-navigation-compose", version.ref = "hiltNavigationCompose" }
# Data
room-runtime = { module = "androidx.room:room-runtime", version.ref = "room" }
room-ktx = { module = "androidx.room:room-ktx", version.ref = "room" }
room-compiler = { module = "androidx.room:room-compiler", version.ref = "room" }
coil-compose = { module = "io.coil-kt.coil3:coil-compose", version.ref = "coil" }
coil-network-okhttp = { module = "io.coil-kt.coil3:coil-network-okhttp", version.ref = "coil" }

[plugins]
android-application = { id = "com.android.application", version.ref = "agp" }
kotlin-android = { id = "org.jetbrains.kotlin.android", version.ref = "kotlin" }
kotlin-compose = { id = "org.jetbrains.kotlin.plugin.compose", version.ref = "kotlin" }
kotlin-serialization = { id = "org.jetbrains.kotlin.plugin.serialization", version.ref = "kotlin" }
ksp = { id = "com.google.devtools.ksp", version.ref = "ksp" }
hilt = { id = "com.google.dagger.hilt.android", version.ref = "hilt" }
```

**Critical:** the Compose compiler is the `org.jetbrains.kotlin.plugin.compose` Gradle plugin (required since Kotlin 2.0). Do **not** set `composeOptions { kotlinCompilerExtensionVersion = ... }` — that key is obsolete and applying it is a red flag for stale code.

## Module build file

`app/build.gradle.kts`:

```kotlin
plugins {
    alias(libs.plugins.android.application)
    alias(libs.plugins.kotlin.android)
    alias(libs.plugins.kotlin.compose)
    alias(libs.plugins.kotlin.serialization)
    alias(libs.plugins.ksp)
    alias(libs.plugins.hilt)
}

android {
    namespace = "com.example.tv"
    compileSdk = 37

    defaultConfig {
        applicationId = "com.example.tv"
        minSdk = 23          // TV: covers Android TV devices in the field; 21 also possible
        targetSdk = 36       // Android 16; meets Play requirement for new apps
        versionCode = 1
        versionName = "1.0"
    }

    buildTypes {
        release {
            isMinifyEnabled = true
            isShrinkResources = true
            proguardFiles(
                getDefaultProguardFile("proguard-android-optimize.txt"),
                "proguard-rules.pro"
            )
        }
    }

    buildFeatures { compose = true }

    // Java 17 toolchain; do not use compileOptions/kotlinOptions blocks anymore.
}

kotlin {
    jvmToolchain(17)
    compilerOptions {
        // Opt-ins if needed, e.g. optIn.add("androidx.tv.material3.ExperimentalTvMaterial3Api")
    }
}

dependencies {
    implementation(platform(libs.compose.bom))
    androidTestImplementation(platform(libs.compose.bom))

    implementation(libs.compose.ui)
    implementation(libs.compose.ui.tooling.preview)
    debugImplementation(libs.compose.ui.tooling)
    debugImplementation(libs.compose.ui.test.manifest)

    implementation(libs.androidx.tv.material)
    implementation(libs.androidx.activity.compose)
    implementation(libs.lifecycle.runtime.compose)
    implementation(libs.lifecycle.viewmodel.compose)

    implementation(libs.navigation3.runtime)
    implementation(libs.navigation3.ui)
    implementation(libs.lifecycle.viewmodel.navigation3)

    implementation(libs.media3.exoplayer)
    implementation(libs.media3.exoplayer.hls)
    implementation(libs.media3.ui.compose)
    implementation(libs.media3.session)

    implementation(libs.hilt.android)
    ksp(libs.hilt.compiler)
    implementation(libs.hilt.navigation.compose)

    implementation(libs.room.runtime)
    implementation(libs.room.ktx)
    ksp(libs.room.compiler)

    implementation(libs.coil.compose)
    implementation(libs.coil.network.okhttp)

    testImplementation("junit:junit:4.13.2")
    testImplementation("app.cash.turbine:turbine:1.2.0")
    androidTestImplementation(libs.compose.ui.test.junit4)
}
```

Enable Gradle configuration cache and parallel builds in `gradle.properties`:

```properties
org.gradle.configuration-cache=true
org.gradle.parallel=true
org.gradle.caching=true
android.useAndroidX=true
kotlin.code.style=official
```

## AndroidManifest for TV

The manifest is what makes Google Play recognize the app as a TV app and what makes it appear on the Android TV home screen. Get these five things right:

```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android">

    <!-- TVs have no touchscreen: must be required=false or Play rejects TV eligibility -->
    <uses-feature
        android:name="android.hardware.touchscreen"
        android:required="false" />

    <!-- Declares TV support (Leanback). required=false lets the same APK ship to phones too -->
    <uses-feature
        android:name="android.software.leanback"
        android:required="false" />

    <application
        android:banner="@drawable/tv_banner"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:theme="@style/Theme.App">

        <activity
            android:name=".MainActivity"
            android:exported="true"
            android:configChanges="keyboard|keyboardHidden|navigation|screenSize"
            android:theme="@style/Theme.App">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <!-- LEANBACK_LAUNCHER makes the app appear on the Android TV home screen -->
                <category android:name="android.intent.category.LEANBACK_LAUNCHER" />
                <!-- Include LAUNCHER too only if the same APK also targets phones -->
            </intent-filter>
        </activity>
    </application>
</manifest>
```

Rules:
- The **banner** (`android:banner`) is mandatory for any app with a `LEANBACK_LAUNCHER` activity; it is the home-screen entry point. Per Android Developers "Create and run a TV app": *"For the banner, use an xhdpi resource with a size of 320 x 180 px. Text must be included in the image."*
- Add `android:isGame="true"` on `<application>` only for games — it moves the app into the Games row.
- If the app is TV-only, omit `LAUNCHER` and keep only `LEANBACK_LAUNCHER`.
- Do **not** request permissions or hardware that exclude TVs (e.g. don't make `android.hardware.camera` required).

## Theming with tv-material

Use `androidx.tv.material3` for the theme, colors, and typography. It provides `MaterialTheme`, `darkColorScheme`, `lightColorScheme`, `Text`, and `Surface`. TV apps are dark-first (10-foot viewing, dim rooms).

```kotlin
import androidx.tv.material3.MaterialTheme
import androidx.tv.material3.darkColorScheme
import androidx.tv.material3.Typography

private val TvDarkColors = darkColorScheme(
    primary = Color(0xFFB4C5FF),
    onPrimary = Color(0xFF001A41),
    surface = Color(0xFF121316),
    onSurface = Color(0xFFE3E2E6),
)

@Composable
fun TvAppTheme(content: @Composable () -> Unit) {
    MaterialTheme(
        colorScheme = TvDarkColors,
        typography = Typography(),
        content = content,
    )
}
```

**Interop rule:** import `Text`, `Surface`, `Button`, `Card`, `Icon`, `MaterialTheme` from `androidx.tv.material3`. Layout primitives (`Row`, `Column`, `Box`, `Spacer`, `LazyRow`, `LazyColumn`, `LazyVerticalGrid`), all `Modifier`s, and `androidx.compose.foundation` come from base Compose — there is no TV-specific fork of those anymore. It is a compile-time bug waiting to happen to accidentally import `androidx.compose.material3.Button` in a TV app: it has no focus scaling or glow and looks dead on a TV.

## What is in tv-material (and what was removed)

tv-material 1.0.1 provides (all under `androidx.tv.material3`): `Surface`, `Button`, `OutlinedButton`, `WideButton`, `IconButton`, `OutlinedIconButton`, `Card`, `ClassicCard`, `CompactCard`, `WideClassicCard`, `StandardCardContainer`, `WideCardContainer`, `Carousel` (+ `rememberCarouselState`), `TabRow`/`Tab`, `NavigationDrawer`/`ModalNavigationDrawer`/`NavigationDrawerItem` (+ `rememberDrawerState`), `ListItem`/`DenseListItem`, `Checkbox`/`TriStateCheckbox`, `Switch`, `RadioButton`, `AssistChip`/`FilterChip`/`InputChip`/`SuggestionChip`, `Text`, `Icon`, `MaterialTheme`, and color scheme helpers. (Note the naming from the 1.0.0 stabilization: `NonInteractiveSurfaceDefaults`→`SurfaceDefaults`, `NonInteractiveSurfaceColors`→`SurfaceColors`, and `StandardCardLayout`/`WideCardLayout`→`StandardCardContainer`/`WideCardContainer`; `NavigationDrawer`/`NavigationDrawerScope`/`NavigationDrawerItem` are stable.)

| Concern | Use | Do NOT use |
|---|---|---|
| Buttons, cards, surfaces | `androidx.tv.material3.*` | `androidx.compose.material3.*` |
| Horizontal/vertical/grid lists | `LazyRow`/`LazyColumn`/`LazyVerticalGrid` (`androidx.compose.foundation`) + `Modifier.focusRestorer()` | `androidx.tv:tv-foundation` `TvLazyRow`/`TvLazyColumn` — **removed** |
| Hero rotator | `Carousel` (tv-material) | — |
| Featured immersive row | Hand-rolled (`LazyRow` + track focused index + `AnimatedContent` background) | `ImmersiveList` — **removed from tv-material** |
| Top navigation | `TabRow` (tv-material) | phone `TabRow` |
| Side navigation | `NavigationDrawer`/`ModalNavigationDrawer` (tv-material) | phone navigation rail/drawer |

**`androidx.tv:tv-foundation` is gone.** Per Paul Lammertsma's "Migrating Jetpack Compose for TV from alpha to stable" (Android Developers): *"TvLazyRow, TvLazyColumn, TvLazyHorizontalGrid and TvLazyVerticalGrid have been removed because their functionality has been incorporated into the scrollable containers in compose-foundation version 1.7.0-beta02."* The supporting `TvLazyListState`/`rememberTvLazyListState` classes were replaced by their foundation equivalents. Migrate by swapping `TvLazy*` → `Lazy*` and `rememberTvLazy*State` → `rememberLazy*State`. **`ImmersiveList` was also removed** — build immersive backgrounds yourself (pattern below).

## Focus management — the core TV skill

Everything about TV interaction is focus. All focus modifiers live in **`androidx.compose.ui.focus`** (module `androidx.compose.ui:ui`) — not in `androidx.compose.foundation`.

### Focusable rows with focus restoration

`Modifier.focusRestorer()` saves the last-focused child of a focus group and restores it when focus re-enters — this is what makes a D-pad user return to the same card after moving away. Current stable signature:

```kotlin
fun Modifier.focusRestorer(fallback: FocusRequester = FocusRequester.Default): Modifier
```

The lambda-based `focusRestorer(onRestoreFailed)` overload is **deprecated** (WARNING level) — use the `FocusRequester` version. Call `focusRestorer()` **before** `focusGroup()`.

```kotlin
import androidx.compose.foundation.lazy.LazyRow
import androidx.compose.foundation.lazy.items
import androidx.compose.ui.focus.focusRestorer
import androidx.tv.material3.Card
import androidx.tv.material3.Text

@Composable
fun MovieRow(movies: List<Movie>, onClick: (Movie) -> Unit) {
    LazyRow(
        horizontalArrangement = Arrangement.spacedBy(12.dp),
        contentPadding = PaddingValues(horizontal = 48.dp), // overscan margin
        modifier = Modifier.focusRestorer(),
    ) {
        items(movies, key = { it.id }) { movie ->
            Card(onClick = { onClick(movie) }, modifier = Modifier.width(160.dp)) {
                Text(movie.title, Modifier.padding(8.dp))
            }
        }
    }
}
```

**Critical behavioral note:** recent Compose `focusRestorer` no longer pins the previously focused item — give lazy items a stable `key` so the restored item has the same composition hash, or restoration silently fails.

### Requesting initial focus

A TV screen must place focus somewhere on entry or the D-pad does nothing. Use a `FocusRequester` + `LaunchedEffect`:

```kotlin
import androidx.compose.ui.focus.focusRequester
import androidx.compose.ui.focus.FocusRequester

@Composable
fun HomeScreen() {
    val firstItem = remember { FocusRequester() }
    LaunchedEffect(Unit) { firstItem.requestFocus() }

    Column {
        Button(onClick = {}, modifier = Modifier.focusRequester(firstItem)) {
            Text("Play")
        }
        // ...
    }
}
```

### Directing focus traversal with focusProperties

`Modifier.focusProperties { }` overrides D-pad traversal. In current Compose the enter/exit callbacks were renamed: per the Compose for TV release notes, *"FocusProperties.enter and FocusProperties.exit have been replaced with onEnter and onExit, respectively, using a receiver scope instead of FocusDirection parameter."* Inside those scopes the direction is available as `requestedFocusDirection`, and you redirect with `FocusRequester.requestFocus()` or cancel with `cancelFocusChange()`. Directional overrides remain `FocusRequester`-typed properties: `up`, `down`, `left`, `right`, `start`, `end`, `next`, `previous`.

```kotlin
import androidx.compose.ui.focus.focusProperties
import androidx.compose.ui.focus.FocusRequester

val menu = remember { FocusRequester() }
val content = remember { FocusRequester() }

Row {
    Column(Modifier.focusProperties { right = content }) { /* menu */ }
    Column(
        Modifier.focusProperties {
            left = menu
            up = FocusRequester.Cancel   // block focus escaping upward
        }
    ) { /* content grid */ }
}
```

### Save/restore focus across navigation

For screens where `focusRestorer()` is not enough (e.g. returning from a detail screen), use `FocusRequester.saveFocusedChild()` before navigating away and `restoreFocusedChild()` on return, wired through `focusProperties { onEnter/onExit }`. Treat `saveFocusedChild()`/`restoreFocusedChild()` as still requiring `@OptIn(ExperimentalComposeUiApi::class)`.

```kotlin
@OptIn(ExperimentalComposeUiApi::class)
val parent = remember { FocusRequester() }

Box(
    Modifier
        .focusRequester(parent)
        .focusProperties {
            onExit = { parent.saveFocusedChild() }
            onEnter = { if (!parent.restoreFocusedChild()) { /* fallback */ } }
        }
) { /* grid */ }
```

### Hand-rolled immersive list (ImmersiveList replacement)

Since `ImmersiveList` was removed, track the focused item and cross-fade a background:

```kotlin
@Composable
fun ImmersiveRow(items: List<Media>) {
    var focusedIndex by remember { mutableIntStateOf(0) }

    Box(Modifier.fillMaxSize()) {
        AnimatedContent(targetState = focusedIndex, label = "bg") { index ->
            AsyncImage(
                model = items[index].backdropUrl,
                contentDescription = null,
                contentScale = ContentScale.Crop,
                modifier = Modifier.fillMaxSize(),
            )
        }
        LazyRow(
            Modifier
                .align(Alignment.BottomStart)
                .focusRestorer(),
            contentPadding = PaddingValues(horizontal = 48.dp),
        ) {
            itemsIndexed(items, key = { _, m -> m.id }) { index, media ->
                Card(
                    onClick = {},
                    modifier = Modifier.onFocusChanged { if (it.isFocused) focusedIndex = index },
                ) { Text(media.title) }
            }
        }
    }
}
```

## D-pad key handling

For media transport keys and custom D-pad behavior, use `Modifier.onPreviewKeyEvent` (fires top-down, ancestors first) or `Modifier.onKeyEvent` (bottom-up). Both live in `androidx.compose.ui.input.key` and are stable. Return `true` to consume. A physical press produces a `KeyDown` then `KeyUp`; filter on `KeyEventType` to avoid double-handling.

```kotlin
import androidx.compose.ui.input.key.*

Box(
    Modifier.onPreviewKeyEvent { event ->
        if (event.type != KeyEventType.KeyUp) return@onPreviewKeyEvent false
        when (event.key) {
            Key.MediaPlayPause -> { player.togglePlay(); true }
            Key.MediaFastForward -> { player.seekForward(); true }
            Key.MediaRewind -> { player.seekBack(); true }
            Key.DirectionCenter, Key.Enter -> { toggleControls(); true }
            else -> false
        }
    }
) { /* player surface */ }
```

Never rely on `KEYCODE_MENU`, `KEYCODE_SEARCH`, or `KEYCODE_BUTTON_START` for essential navigation — not all remotes have them.

## Back handling

`BackHandler` and `PredictiveBackHandler` are stable and live in `androidx.activity.compose` (`PredictiveBackHandler` introduced with activity-compose 1.8.0). On TV, the Back button is the universal "up/out" affordance; handle it explicitly rather than letting the activity finish unexpectedly.

```kotlin
import androidx.activity.compose.BackHandler

BackHandler(enabled = drawerOpen) { drawerOpen = false }
```

`PredictiveBackHandler` collects a `Flow<BackEventCompat>` for progress-driven animations:

```kotlin
import androidx.activity.compose.PredictiveBackHandler

PredictiveBackHandler(enabled = true) { progress ->
    try {
        progress.collect { event -> offset = event.progress * 100f }
        onBackConfirmed()
    } catch (e: CancellationException) {
        offset = 0f
    }
}
```

## Overscan and the 10-foot UI

TVs crop edges (overscan). Content near the border can be clipped. Apply a safe-area margin to the root. Per Android Developers "Layouts in the Leanback UI toolkit": *"Adding a 5% margin of 48 dp on the left and right edges and 27 dp on the top and bottom edges to a layout helps ensure that screen elements in the layout are within the overscan-safe area."* Size everything for viewing from ~3 meters: large type, generous spacing, clear focus indication.

```kotlin
@Composable
fun TvScaffold(content: @Composable () -> Unit) {
    Box(
        Modifier
            .fillMaxSize()
            .background(MaterialTheme.colorScheme.surface)
            .padding(horizontal = 48.dp, vertical = 27.dp)
    ) { content() }
}
```

Use `Configuration.uiMode and Configuration.UI_MODE_TYPE_MASK == Configuration.UI_MODE_TYPE_TELEVISION` to detect TV when a shared codebase needs a runtime branch.

## Media playback with Media3

**Use AndroidX Media3 (`androidx.media3:*`, stable 1.9.0). The legacy `com.google.android.exoplayer2` is end-of-life — never add it to a new project.** Build an `ExoPlayer`, drive video through the Compose-native `PlayerSurface` from `media3-ui-compose`, and manage lifecycle in a `ViewModel` or with lifecycle-aware effects.

```kotlin
import androidx.media3.exoplayer.ExoPlayer
import androidx.media3.common.MediaItem
import androidx.media3.ui.compose.PlayerSurface
import androidx.media3.common.util.UnstableApi

@OptIn(UnstableApi::class)
@Composable
fun VideoPlayer(url: String, modifier: Modifier = Modifier) {
    val context = LocalContext.current
    val player = remember {
        ExoPlayer.Builder(context).build().apply {
            setMediaItem(MediaItem.fromUri(url))
            prepare()
            playWhenReady = true
        }
    }
    DisposableEffect(Unit) { onDispose { player.release() } }

    PlayerSurface(player = player, modifier = modifier.fillMaxSize())
}
```

Notes:
- `PlayerSurface` and the Media3 Compose state holders (`rememberPresentationState`, `rememberPlayPauseButtonState`, etc.) shipped in `media3-ui-compose` (since Media3 1.6.0); they are currently annotated `@UnstableApi`, so opt in. Material 3-themed transport controls live in the newer `media3-ui-compose-material3` module.
- For HLS add `media3-exoplayer-hls`; for DASH add `media3-exoplayer-dash`. Widevine DRM is configured via `MediaItem.DrmConfiguration`.
- For background audio and system transport controls (and Assistant/Cast integration), run playback in a `MediaSessionService` with a `MediaSession` from `media3-session`.
- Per the "Media3 1.9.0 – What's new" blog (Dec 2025), 1.9.0 added `media3-inspector` (`MetadataRetriever` for reading duration/format/thumbnails without playback) and `PreloadManager` caching, where *"you can now choose PreloadStatus.specifiedRangeCached(0, 5000) as a target state for preloaded items"* — smoother catalog browsing on TV.

## TV home-screen channels

Publish content to the Android TV home screen (recommendation channels and Watch Next) with `androidx.tvprovider` (`PreviewChannelHelper`, `PreviewChannel`, `PreviewProgram`, `WatchNextProgram`). This is the current supported path; the old Leanback recommendation APIs are deprecated along with the rest of Leanback.

## Architecture: UI / domain / data

Standard Android app architecture applies unchanged on TV. UI state is exposed by a `ViewModel` as `StateFlow` and collected with `collectAsStateWithLifecycle()` from `androidx.lifecycle:lifecycle-runtime-compose`. Use explicit backing fields (stable in Kotlin 2.4) to collapse the `_state`/`state` pair.

```kotlin
@HiltViewModel
class HomeViewModel @Inject constructor(
    private val repo: CatalogRepository,
) : ViewModel() {

    // Kotlin 2.4 explicit backing field: expose read-only StateFlow, mutate the wider type internally
    val uiState: StateFlow<HomeUiState>
        field = MutableStateFlow(HomeUiState.Loading)

    init {
        viewModelScope.launch {
            repo.rows().collect { rows -> uiState.value = HomeUiState.Ready(rows) }
        }
    }
}

sealed interface HomeUiState {
    data object Loading : HomeUiState
    data class Ready(val rows: List<CatalogRow>) : HomeUiState
}
```

```kotlin
@Composable
fun HomeRoute(viewModel: HomeViewModel = hiltViewModel()) {
    val state by viewModel.uiState.collectAsStateWithLifecycle()
    when (val s = state) {
        HomeUiState.Loading -> LoadingScreen()
        is HomeUiState.Ready -> HomeScreen(s.rows)
    }
}
```

**Never** collect a `Flow` with a bare `collectAsState()` on Android — `collectAsStateWithLifecycle()` stops collection when the app is not in the foreground, which matters for TV's leanback lifecycle.

## Dependency injection with Hilt

Hilt with **KSP** (never kapt). Annotate the `Application`, provide dependencies in modules, inject `ViewModel`s with `@HiltViewModel`.

```kotlin
@HiltAndroidApp
class TvApp : Application()

@Module
@InstallIn(SingletonComponent::class)
object NetworkModule {
    @Provides @Singleton
    fun okHttp(): OkHttpClient = OkHttpClient.Builder().build()

    @Provides @Singleton
    fun retrofit(client: OkHttpClient): Retrofit =
        Retrofit.Builder()
            .baseUrl("https://api.example.com/")
            .client(client)
            .addConverterFactory(
                Json.asConverterFactory("application/json".toMediaType())
            )
            .build()
}
```

## Navigation

For new TV codebases, **Navigation 3 (`androidx.navigation3`, stable 1.0.0)** is the recommended, Compose-first choice: the back stack is a plain observable `List` of serializable keys that you own. It composes naturally with TV's single-activity model.

```kotlin
import androidx.navigation3.runtime.NavKey
import androidx.navigation3.runtime.rememberNavBackStack
import androidx.navigation3.runtime.entry
import androidx.navigation3.ui.NavDisplay
import kotlinx.serialization.Serializable

@Serializable data object Home : NavKey
@Serializable data class Detail(val id: String) : NavKey

@Composable
fun TvNavHost() {
    val backStack = rememberNavBackStack(Home)
    NavDisplay(
        backStack = backStack,
        onBack = { backStack.removeLastOrNull() },
        entryProvider = entryProvider {
            entry<Home> { HomeRoute(onOpen = { backStack.add(Detail(it)) }) }
            entry<Detail> { key -> DetailRoute(key.id) }
        },
    )
}
```

If a team is standardized on **Navigation Compose (Nav2, `androidx.navigation:navigation-compose`)**, that remains fully supported and stable; use type-safe routes with `@Serializable` classes (Navigation 2.8+). Pick one and be consistent — do not mix.

## Data: Room, DataStore, networking, images

- **Room 2.7** with KSP; expose queries as `Flow` and suspend functions. Room is KMP-capable now.
- **DataStore** for key-value/user prefs (Proto or Preferences) — never `SharedPreferences` in new code.
- **Networking:** Retrofit + OkHttp + `kotlinx.serialization`, or Ktor client for KMP. Use `kotlinx-serialization-json`, not Gson.
- **Images:** **Coil 3** (`io.coil-kt.coil3`) — Compose-native, multiplatform, with `AsyncImage`. Prefer it over Glide/Picasso, which are View-era and awkward in Compose.

```kotlin
import coil3.compose.AsyncImage

AsyncImage(
    model = movie.posterUrl,
    contentDescription = movie.title,
    contentScale = ContentScale.Crop,
    modifier = Modifier.width(160.dp).aspectRatio(2f / 3f),
)
```

## Compose performance for TV

Focus-heavy TV UIs recompose a lot; keep it cheap:
- Make UI-state classes `@Immutable`/`@Stable`; prefer `data class`es of stable types and `kotlinx.collections.immutable` lists so **strong skipping** can skip unchanged composables. (Strong skipping is default since the Compose compiler shipped with Kotlin 2.0.20.)
- Always pass a stable `key` to `items {}` in lazy lists — required for correct focus restoration *and* recomposition.
- Read state as late as possible; pass lambdas (`onClick = { vm.select(id) }`), not captured values, into leaf composables to defer reads.
- Use `derivedStateOf` when deriving a value from frequently changing state; `remember(key)` to scope caching.
- Use `Modifier.animateItem()` for list reordering and `Modifier.animateContentSize()` for focus-driven size changes.
- Ship a **Baseline Profile** (macrobenchmark module) — measurable cold-start and scroll-jank wins on low-power TV SoCs.

## Previews on TV

Preview at TV resolution with the built-in device specs.

```kotlin
import androidx.compose.ui.tooling.preview.Preview
import androidx.compose.ui.tooling.preview.Devices

@Preview(device = Devices.TV_1080p, showBackground = true)
@Composable
private fun MovieRowPreview() {
    TvAppTheme { MovieRow(sampleMovies, onClick = {}) }
}
```

`Devices.TV_1080p` and `Devices.TV_720p` render the 10-foot canvas. Wrap previews in the TV theme so tv-material components show focus/color correctly.

## Testing

- **UI tests:** `createComposeRule()` / `createAndroidComposeRule<MainActivity>()` with semantics matchers and test tags.
- **D-pad injection:** drive focus with key input.

```kotlin
@get:Rule val rule = createComposeRule()

@Test fun dpadRightMovesFocus() {
    rule.setContent { TvAppTheme { MovieRow(sampleMovies, onClick = {}) } }
    rule.onNodeWithTag("row").performKeyInput {
        pressKey(Key.DirectionRight)
    }
    rule.onNodeWithTag("movie-1").assertIsFocused()
}
```

- **Flow tests:** Turbine.
- **Screenshot tests:** Roborazzi (JVM, Robolectric-based, supports post-action snapshots) or Paparazzi (JVM, fastest); both render Compose without a device. For preview-driven snapshots use Compose Preview Screenshot Testing. Configure a TV device spec for correct 1080p rendering.
- **Note (Compose 1.11+):** the v2 Compose testing APIs are now the default and use `StandardTestDispatcher` — launched coroutines don't execute until the virtual clock advances. Advance the clock in tests that used to rely on immediate execution.
- **Startup/scroll perf:** Macrobenchmark + Baseline Profiles; UiAutomator for end-to-end TV flows.

## Kotlin language features to use (2.4)

- **Context parameters** (stable in 2.4): thread implicit dependencies (a logger, a session) without passing them through every call. Note: naming a context argument at the call site is still behind `-Xexplicit-context-arguments` in 2.4.
- **Explicit backing fields** (stable in 2.4): the `_state`/`state` StateFlow pattern collapses to one property (shown above).
- **Guard conditions in `when`** (stable since 2.1): `is Ready if state.rows.isEmpty() -> ...`.
- **`kotlin.time.Duration`** for timeouts/seek offsets instead of raw `Long` millis.
- K2 is the only compiler in 2.4 (K1 removed) — this is transparent but means all code must be K2-clean.

## Code style tooling

Pick one formatter and enforce in CI. **ktlint** (via the Spotless or ktlint-gradle plugin) is the low-config default; **detekt** adds static analysis on top. A minimal detekt config:

```yaml
# config/detekt/detekt.yml
build:
  maxIssues: 0
complexity:
  LongMethod:
    threshold: 60
style:
  MagicNumber:
    ignoreNumbers: ['-1', '0', '1', '2']
    active: false   # dp/sizing literals are pervasive in Compose
formatting:
  active: true
  Indentation:
    indentSize: 4
```

```kotlin
// build.gradle.kts (root)
plugins {
    id("io.gitlab.arturbosch.detekt") version "1.23.7"
}
detekt { config.setFrom("config/detekt/detekt.yml") }
```

## R8 / ProGuard

R8 full mode is on by default with AGP 9 when `isMinifyEnabled = true`. Keep rules for reflection-driven libraries: kotlinx.serialization generates serializers, and Media3/Room mostly ship their own consumer rules. For serializable Navigation 3 keys and DTOs:

```proguard
# kotlinx.serialization
-keepattributes *Annotation*, InnerClasses
-dontnote kotlinx.serialization.**
-keepclassmembers class **$$serializer { *; }
-keepclasseswithmembers class * {
    kotlinx.serialization.KSerializer serializer(...);
}
```

## Anti-patterns to avoid

| Anti-pattern | Why it's wrong | Do instead |
|---|---|---|
| `androidx.compose.material3.Button`/`Card` in TV UI | No D-pad focus scaling/glow; looks dead on TV | `androidx.tv.material3.*` |
| `Modifier.clickable` with no focus visual | Invisible to D-pad users | tv-material components (built-in focus indication) or add `onFocusChanged` + visual |
| `androidx.tv:tv-foundation` `TvLazyRow`/`TvLazyColumn` | Library removed | `LazyRow`/`LazyColumn` (foundation) + `Modifier.focusRestorer()` |
| `ImmersiveList` from tv-material | Removed | Hand-roll: track focused index + `AnimatedContent` |
| `androidx.leanback` fragments/`BrowseSupportFragment` | Deprecated; Material 1, View-based | Compose for TV |
| `com.google.android.exoplayer2` | End-of-life | `androidx.media3:media3-exoplayer` |
| Missing `LEANBACK_LAUNCHER` category / banner | App won't appear on TV home screen | Add both in manifest |
| `touchscreen` feature `required="true"` (default) | Play won't treat it as a TV app | `required="false"` |
| No initial focus on screen entry | D-pad does nothing | `FocusRequester` + `LaunchedEffect { requestFocus() }` |
| Hardcoded edge sizing, no overscan | Content clipped on real TVs | 48dp/27dp safe-area padding |
| Assuming scroll/swipe gestures | No touch on TV | D-pad focus traversal only |
| `kapt` for Hilt/Room | Slower, legacy | `ksp` |
| `composeOptions { kotlinCompilerExtensionVersion }` | Obsolete since Kotlin 2.0 | `org.jetbrains.kotlin.plugin.compose` plugin |
| `collectAsState()` on Android | Keeps collecting in background | `collectAsStateWithLifecycle()` |
| Gson + Glide/Picasso | View-era, non-Compose | kotlinx.serialization + Coil 3 |
| Importing focus APIs from `androidx.compose.foundation` | Wrong package | `androidx.compose.ui.focus` |

## Version & compatibility quick reference

| Layer | Setting |
|---|---|
| Kotlin | 2.4.0 (K2-only) |
| JVM toolchain | Java 17 |
| compileSdk | 37 (required by Compose 1.12) |
| targetSdk | 36 (new apps on Play); TV existing-app floor is API 34 |
| minSdk | 23 (typical TV floor; 21 possible) |
| AGP / Gradle | 9.1.1+ / 9.x |
| Compose BOM | 2026.08.00 (core 1.12.0) |
| Compose compiler | `org.jetbrains.kotlin.plugin.compose` (matches Kotlin version) |
| tv-material | 1.0.1 stable |
| Media3 | 1.9.0 |
| Navigation | Navigation 3 1.0.0 (recommended) or Navigation Compose 2.8+ |
| Hilt / KSP | 2.56.x / KSP2 |