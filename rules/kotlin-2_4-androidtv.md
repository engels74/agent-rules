---
type: "agent_requested"
description: "Kotlin 2.4 + Android TV + Jetpack Compose coding guidelines"
---
# Android TV with Jetpack Compose and Kotlin 2.4

This stack builds living-room apps in pure Compose: `androidx.tv.material3` components on top of the standard Compose runtime, driven by a D-pad rather than a touchscreen, compiled with the K2 Kotlin 2.4 toolchain. It is exceptional at declarative, focus-first UI (rows of cards, carousels, immersive backgrounds) and at Compose-native video via Media3's `PlayerSurface`. Optimize for **focus correctness** above all else — on TV there is no touch, so every interactive element must be focusable, must show a visible focus state (scale/glow/border, not a ripple), and must restore focus predictably when the user scrolls away and returns.

The biggest way agents write wrong-but-plausible code here is importing phone habits: using `androidx.compose.material3` `Button`/`Card` for interactive elements (they render no D-pad focus indication and their ripple is invisible without touch), reaching for the deprecated `androidx.tv.foundation` `TvLazyRow`/`TvLazyColumn` (the lazy layouts moved back into `androidx.compose.foundation`), forgetting `focusRestorer()` on rows (focus snaps to the first item every time), and writing Compose UI tests that silently pass in touch mode when the feature only works under D-pad input. Get focus, lists, and lifecycle right and the rest of the stack is ordinary modern Android.

## Project and build setup

Use a Gradle version catalog as the single source of truth for versions; keep exact pins here and nowhere else. AGP 9.0+ ships built-in Kotlin support, so you no longer apply `org.jetbrains.kotlin.android` — the Compose compiler is applied through its own plugin `org.jetbrains.kotlin.plugin.compose`.

`gradle/libs.versions.toml`:

```toml
[versions]
kotlin = "2.4.10"              # Kotlin 2.4 language line, K2 compiler
ksp = "2.3.10"                 # KSP2 uses decoupled versioning; this line is compatible with Kotlin 2.4.x
agp = "9.2.0"                  # supports compileSdk 37
composeBom = "2026.08.00"      # Compose 1.12 line
tvMaterial = "1.1.0"
media3 = "1.11.0"
coil = "3.6.1"
hilt = "2.59.2"
hiltAndroidx = "1.4.0"
room = "2.8.4"
coroutines = "1.11.0"
serialization = "1.11.0"
lifecycle = "2.11.0"           # requires AGP 9.2+
activityCompose = "1.13.0"
datastore = "1.2.1"
navigation3 = "1.1.6"
coreKtx = "1.18.0"
turbine = "1.2.1"
mockk = "1.14.7"
robolectric = "4.16.1"
junit = "4.13.2"
androidxJunit = "1.2.1"

[libraries]
compose-bom = { module = "androidx.compose:compose-bom", version.ref = "composeBom" }
compose-foundation = { module = "androidx.compose.foundation:foundation" }
compose-ui = { module = "androidx.compose.ui:ui" }
compose-ui-tooling = { module = "androidx.compose.ui:ui-tooling" }
compose-ui-tooling-preview = { module = "androidx.compose.ui:ui-tooling-preview" }
compose-ui-test-junit4 = { module = "androidx.compose.ui:ui-test-junit4" }
compose-ui-test-manifest = { module = "androidx.compose.ui:ui-test-manifest" }
tv-material = { module = "androidx.tv:tv-material", version.ref = "tvMaterial" }
activity-compose = { module = "androidx.activity:activity-compose", version.ref = "activityCompose" }
core-ktx = { module = "androidx.core:core-ktx", version.ref = "coreKtx" }
lifecycle-runtime-compose = { module = "androidx.lifecycle:lifecycle-runtime-compose", version.ref = "lifecycle" }
lifecycle-viewmodel-compose = { module = "androidx.lifecycle:lifecycle-viewmodel-compose", version.ref = "lifecycle" }
lifecycle-viewmodel-navigation3 = { module = "androidx.lifecycle:lifecycle-viewmodel-navigation3", version.ref = "lifecycle" }
navigation3-runtime = { module = "androidx.navigation3:navigation3-runtime", version.ref = "navigation3" }
navigation3-ui = { module = "androidx.navigation3:navigation3-ui", version.ref = "navigation3" }
hilt-android = { module = "com.google.dagger:hilt-android", version.ref = "hilt" }
hilt-compiler = { module = "com.google.dagger:hilt-compiler", version.ref = "hilt" }
hilt-navigation-compose = { module = "androidx.hilt:hilt-navigation-compose", version.ref = "hiltAndroidx" }
room-runtime = { module = "androidx.room:room-runtime", version.ref = "room" }
room-compiler = { module = "androidx.room:room-compiler", version.ref = "room" }
room-ktx = { module = "androidx.room:room-ktx", version.ref = "room" }
datastore-preferences = { module = "androidx.datastore:datastore-preferences", version.ref = "datastore" }
coroutines-android = { module = "org.jetbrains.kotlinx:kotlinx-coroutines-android", version.ref = "coroutines" }
serialization-json = { module = "org.jetbrains.kotlinx:kotlinx-serialization-json", version.ref = "serialization" }
coil-compose = { module = "io.coil-kt.coil3:coil-compose", version.ref = "coil" }
coil-network-okhttp = { module = "io.coil-kt.coil3:coil-network-okhttp", version.ref = "coil" }
media3-exoplayer = { module = "androidx.media3:media3-exoplayer", version.ref = "media3" }
media3-ui-compose = { module = "androidx.media3:media3-ui-compose", version.ref = "media3" }
media3-session = { module = "androidx.media3:media3-session", version.ref = "media3" }
media3-exoplayer-hls = { module = "androidx.media3:media3-exoplayer-hls", version.ref = "media3" }
turbine = { module = "app.cash.turbine:turbine", version.ref = "turbine" }
mockk = { module = "io.mockk:mockk", version.ref = "mockk" }
robolectric = { module = "org.robolectric:robolectric", version.ref = "robolectric" }
junit = { module = "junit:junit", version.ref = "junit" }
androidx-junit = { module = "androidx.test.ext:junit", version.ref = "androidxJunit" }
coroutines-test = { module = "org.jetbrains.kotlinx:kotlinx-coroutines-test", version.ref = "coroutines" }

[plugins]
android-application = { id = "com.android.application", version.ref = "agp" }
kotlin-android = { id = "org.jetbrains.kotlin.android", version.ref = "kotlin" }
compose-compiler = { id = "org.jetbrains.kotlin.plugin.compose", version.ref = "kotlin" }
kotlin-serialization = { id = "org.jetbrains.kotlin.plugin.serialization", version.ref = "kotlin" }
ksp = { id = "com.google.devtools.ksp", version.ref = "ksp" }
hilt = { id = "com.google.dagger.hilt.android", version.ref = "hilt" }
```

Set the Gradle wrapper to 9.5 (`gradle/wrapper/gradle-wrapper.properties`: `distributionUrl=https\://services.gradle.org/distributions/gradle-9.5-bin.zip`). The Kotlin Gradle plugin 2.4.10 requires a minimum of Gradle 7.6.3 and is fully supported through Gradle 9.5.0.

`app/build.gradle.kts`:

```kotlin
plugins {
    alias(libs.plugins.android.application)
    alias(libs.plugins.kotlin.android)
    alias(libs.plugins.compose.compiler)
    alias(libs.plugins.kotlin.serialization)
    alias(libs.plugins.ksp)
    alias(libs.plugins.hilt)
}

android {
    namespace = "com.example.tvapp"
    compileSdk = 37

    defaultConfig {
        applicationId = "com.example.tvapp"
        minSdk = 23          // AndroidX libraries floor; realistic Android TV floor
        targetSdk = 36       // TV Play floor is API 34; 36 is a safe, current choice
        versionCode = 1
        versionName = "1.0"
        testInstrumentationRunner = "androidx.test.runner.AndroidJUnitRunner"
    }

    buildTypes {
        release {
            isMinifyEnabled = true
            isShrinkResources = true
            proguardFiles(getDefaultProguardFile("proguard-android-optimize.txt"), "proguard-rules.pro")
        }
    }

    buildFeatures { compose = true }
    kotlin { jvmToolchain(21) }
}

composeCompiler {
    // Strong skipping is on by default. Add a stability config for third-party
    // types you cannot annotate @Stable/@Immutable:
    stabilityConfigurationFiles.add(rootProject.layout.projectDirectory.file("compose_stability.conf"))
    // Enable only in a dedicated benchmark build type, not release:
    // reportsDestination = layout.buildDirectory.dir("compose_compiler")
}

dependencies {
    implementation(platform(libs.compose.bom))
    androidTestImplementation(platform(libs.compose.bom))

    implementation(libs.compose.foundation)
    implementation(libs.compose.ui)
    implementation(libs.compose.ui.tooling.preview)
    debugImplementation(libs.compose.ui.tooling)
    implementation(libs.tv.material)
    implementation(libs.activity.compose)
    implementation(libs.core.ktx)

    implementation(libs.lifecycle.runtime.compose)
    implementation(libs.lifecycle.viewmodel.compose)
    implementation(libs.lifecycle.viewmodel.navigation3)
    implementation(libs.navigation3.runtime)
    implementation(libs.navigation3.ui)

    implementation(libs.hilt.android)
    ksp(libs.hilt.compiler)
    implementation(libs.hilt.navigation.compose)

    implementation(libs.room.runtime)
    implementation(libs.room.ktx)
    ksp(libs.room.compiler)
    implementation(libs.datastore.preferences)

    implementation(libs.coroutines.android)
    implementation(libs.serialization.json)
    implementation(libs.coil.compose)
    implementation(libs.coil.network.okhttp)

    implementation(libs.media3.exoplayer)
    implementation(libs.media3.exoplayer.hls)
    implementation(libs.media3.ui.compose)
    implementation(libs.media3.session)

    testImplementation(libs.junit)
    testImplementation(libs.coroutines.test)
    testImplementation(libs.turbine)
    testImplementation(libs.mockk)
    testImplementation(libs.robolectric)
    androidTestImplementation(libs.androidx.junit)
    androidTestImplementation(libs.compose.ui.test.junit4)
    debugImplementation(libs.compose.ui.test.manifest)
}
```

Always use **KSP**, never `kapt`, for Room and Hilt — kapt runs a Java stub-generation pass and is markedly slower; both libraries generate their processors through KSP.

### TV manifest

A TV app must announce itself with the Leanback launcher category and declare that touch is not required, or it will not appear in Google Play on TV devices. It must also ship a home-screen banner for each supported language: a 320×180 px image, placed in `drawable-xhdpi`, with localized text identifying the app — omit it and Android Lint fails the build with `MissingTvBanner`.

```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android">

    <uses-feature android:name="android.software.leanback" android:required="false" />
    <uses-feature android:name="android.hardware.touchscreen" android:required="false" />

    <application
        android:name=".TvApp"
        android:banner="@drawable/app_banner"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:theme="@style/Theme.TvApp">

        <activity
            android:name=".MainActivity"
            android:banner="@drawable/app_banner"
            android:exported="true"
            android:screenOrientation="landscape"
            android:theme="@style/Theme.TvApp">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LEANBACK_LAUNCHER" />
            </intent-filter>
        </activity>
    </application>
</manifest>
```

Set `required="false"` on `android.software.leanback` if the same APK also runs on phones; use `required="true"` only for a TV-exclusive build. The `MainActivity` extends `ComponentActivity` and calls `setContent { }` as on any Compose app.

## Kotlin 2.4 language features

Kotlin 2.4 runs the K2 compiler and stabilizes a handful of features worth using. **Context parameters** are now stable (except context arguments and callable references): they let a group of related functions share an implicit dependency instead of threading it through every signature. Use them for domain/service code, not for `@Composable` functions.

```kotlin
context(logger: Logger)
suspend fun refreshCatalog(repo: CatalogRepository): List<Movie> {
    logger.info("refreshing catalog")
    return repo.fetchAll()
}

// call site — `logger` supplied from the surrounding context:
context(consoleLogger)
suspend fun boot(repo: CatalogRepository) = refreshCatalog(repo)
```

**Explicit backing fields** (stable) remove the private-mutable/public-immutable backing-property boilerplate — ideal for exposing a read-only `StateFlow` from a mutable one:

```kotlin
class SearchViewModel : ViewModel() {
    val results: StateFlow<List<Movie>>
        field = MutableStateFlow(emptyList())   // `field` is the mutable backing state

    fun update(movies: List<Movie>) { results.value = movies } // still writable internally
}
```

The **`kotlin.uuid.Uuid`** API is stable and multiplatform-safe; prefer it over `java.util.UUID` for new code (`Uuid.random()`, `Uuid.parse(...)`). Use **guard conditions in `when`** to fold a nested `if` into a branch:

```kotlin
when (state) {
    is Playback.Active if state.isBuffering -> showSpinner()
    is Playback.Active -> showControls()
    Playback.Idle -> showPoster()
}
```

Do **not** rely on **name-based destructuring** in 2.4 — it ships as a preview whose syntax may still change; keep positional destructuring or explicit property access. Exclude other preview/experimental compiler features from production code.

## Compose for TV: surfaces, theming, components

Use `androidx.tv.material3` for anything the user focuses or clicks. Its components carry TV-specific `scale`, `glow`, and `border` interaction states that make focus visible from ten feet away; the phone `androidx.compose.material3` components do not, and their ripple is invisible without touch. Many TV Material APIs require opting in with `@OptIn(ExperimentalTvMaterial3Api::class)` — this opt-in is expected, not a warning to suppress blindly.

`Surface` is the TV building block; the clickable overload wires focus visuals for you:

```kotlin
import androidx.tv.material3.*

@OptIn(ExperimentalTvMaterial3Api::class)
@Composable
fun MovieCard(movie: Movie, onClick: () -> Unit, modifier: Modifier = Modifier) {
    Surface(
        onClick = onClick,
        modifier = modifier.size(width = 240.dp, height = 135.dp),
        shape = ClickableSurfaceDefaults.shape(shape = RoundedCornerShape(12.dp)),
        scale = ClickableSurfaceDefaults.scale(focusedScale = 1.1f),
        border = ClickableSurfaceDefaults.border(
            focusedBorder = Border(BorderStroke(3.dp, MaterialTheme.colorScheme.onSurface)),
        ),
        colors = ClickableSurfaceDefaults.colors(
            containerColor = MaterialTheme.colorScheme.surfaceVariant,
            focusedContainerColor = MaterialTheme.colorScheme.surfaceVariant,
        ),
    ) {
        AsyncImage(
            model = movie.posterUrl,
            contentDescription = movie.title,
            contentScale = ContentScale.Crop,
            modifier = Modifier.fillMaxSize(),
        )
    }
}
```

Note the `glow` visual is unavailable below API 28 and is silently disabled there; do not depend on it as the only focus cue — always pair it with scale or border.

Wrap the app in the TV `MaterialTheme` (from `androidx.tv.material3`), which provides a TV-tuned `ColorScheme`, `Typography`, and `Shapes`. Common TV components and their roles:

| Component | Use for |
| --- | --- |
| `Surface` / `WideButton` / `Button` | focusable containers and actions |
| `Card`, `ClassicCard`, `CompactCard`, `WideClassicCard` | content tiles in rows |
| `Carousel` | auto-rotating hero/featured content at the top of a screen |
| `NavigationDrawer` / `ModalNavigationDrawer` | collapsible left-edge navigation |
| `TabRow` + `Tab` | top navigation that loads content on tab focus |
| `ListItem`, `DenseListItem` | settings and detail lists |

`Carousel` for a featured row (handles Back-to-exit and focus retention across fast key presses):

```kotlin
@OptIn(ExperimentalTvMaterial3Api::class)
@Composable
fun FeaturedCarousel(items: List<Movie>, onPlay: (Movie) -> Unit) {
    Carousel(
        itemCount = items.size,
        modifier = Modifier.fillMaxWidth().height(340.dp),
    ) { index ->
        val movie = items[index]
        Box(Modifier.fillMaxSize()) {
            AsyncImage(
                model = movie.backdropUrl,
                contentDescription = null,
                contentScale = ContentScale.Crop,
                modifier = Modifier.fillMaxSize(),
            )
            Column(Modifier.align(Alignment.BottomStart).padding(48.dp)) {
                Text(movie.title, style = MaterialTheme.typography.headlineLarge)
                Spacer(Modifier.height(12.dp))
                Button(onClick = { onPlay(movie) }) { Text("Play") }
            }
        }
    }
}
```

`TabRow` loads content when a tab gains focus, which is the expected TV convention (no click needed):

```kotlin
@OptIn(ExperimentalTvMaterial3Api::class)
@Composable
fun TopNav(tabs: List<String>, selected: Int, onFocusTab: (Int) -> Unit) {
    TabRow(selectedTabIndex = selected) {
        tabs.forEachIndexed { i, title ->
            Tab(selected = selected == i, onFocus = { onFocusTab(i) }) {
                Text(title, Modifier.padding(horizontal = 16.dp, vertical = 8.dp))
            }
        }
    }
}
```

## Focus and D-pad navigation

This is where TV apps live or die. Focus travels by declaration order by default; you shape it with modifiers.

**Restore focus on rows.** When focus leaves a horizontally scrolling row and comes back, it should land on the item the user left, not the first one. Chain `focusRestorer()` **before** `focusGroup()`:

```kotlin
@Composable
fun MovieRow(
    title: String,
    movies: List<Movie>,
    onClick: (Movie) -> Unit,
    modifier: Modifier = Modifier,
) {
    Column(modifier) {
        Text(title, style = MaterialTheme.typography.titleLarge, modifier = Modifier.padding(start = 48.dp))
        LazyRow(
            horizontalArrangement = Arrangement.spacedBy(16.dp),
            contentPadding = PaddingValues(horizontal = 48.dp),
            modifier = Modifier
                .focusRestorer()   // must come before focusGroup()
                .focusGroup(),
        ) {
            items(movies, key = { it.id }, contentType = { "movie" }) { movie ->
                MovieCard(movie = movie, onClick = { onClick(movie) })
            }
        }
    }
}
```

**Set initial focus** so the screen is usable the instant it appears. Create the `FocusRequester`, attach it, and request focus in a `LaunchedEffect` — never call `requestFocus()` synchronously during composition, because the node is not yet attached and it will throw.

```kotlin
@Composable
fun HomeScreen(rows: List<Row>) {
    val firstItem = remember { FocusRequester() }
    LaunchedEffect(Unit) { firstItem.requestFocus() }

    LazyColumn(verticalArrangement = Arrangement.spacedBy(24.dp)) {
        itemsIndexed(rows, key = { _, r -> r.id }) { index, row ->
            MovieRow(
                title = row.title,
                movies = row.movies,
                onClick = { /* navigate */ },
                modifier = if (index == 0) Modifier.focusRequester(firstItem) else Modifier,
            )
        }
    }
}
```

**Constrain focus escape** with `focusProperties`. For example, prevent Down from leaving the top navigation bar into empty content, or redirect Enter/Center. Use `onExit`/`onEnter` to veto or redirect a directional move:

```kotlin
Row(
    Modifier
        .focusGroup()
        .focusProperties {
            onExit = {
                if (requestedFocusDirection == FocusDirection.Down && contentIsEmpty) {
                    cancelFocusChange()   // trap focus rather than dumping it into nothing
                }
            }
        }
) { /* tabs */ }
```

Handle D-pad center/Enter explicitly only for custom focusables that are not TV Material components; the Material `Surface` `onClick` already responds to both. For a bespoke focusable, use `Modifier.onKeyEvent` and filter `Key.DirectionCenter` and `Key.Enter`, emitting a `PressInteraction` to the interaction source for visual feedback.

## Lists and rendering performance

Lazy lists are the core of a TV UI, so their contracts matter. **Always provide a stable `key`** — without it, focus and scroll position are lost when the backing list changes, and Compose cannot skip unchanged items. **Provide `contentType`** when a list mixes item shapes so Compose can reuse compositions of the same type. Hoist immutable data (`List` is treated as unstable unless wrapped; prefer `kotlinx.collections.immutable`'s `ImmutableList`/`persistentListOf` for parameters that must be stable, or annotate your data holders `@Immutable`).

For values that are derived from other state and read during scrolling, wrap them in `derivedStateOf` so recomposition only fires when the derived result changes:

```kotlin
val showScrollToTop by remember {
    derivedStateOf { listState.firstVisibleItemIndex > 3 }
}
```

Use the primitive state factories (`mutableIntStateOf`, `mutableLongStateOf`, `mutableFloatStateOf`) for `Int`/`Long`/`Float`/`Double` to avoid autoboxing; `mutableStateOf` for those types boxes on every write. Ship a **baseline profile** for TV startup and scroll: apply the `androidx.baselineprofile` plugin with a Macrobenchmark module and generate a profile exercising the home screen and a row scroll — TV hardware is frequently low-powered, so JIT warmup shows up as visible jank on first launch.

> Excerpt: the row/card composables referenced above (`MovieCard`, `MovieRow`) are defined in earlier sections; `Row`/`Movie` are app model types.

## Navigation

Navigation 3 (`androidx.navigation3`) is stable and is the Compose-first navigation model: the back stack is ordinary observable state you own (a `SnapshotStateList` of `@Serializable` keys), and `NavDisplay` renders it. This matches Compose's declarative model and gives compile-time-typed destinations.

```kotlin
import androidx.navigation3.runtime.*
import androidx.navigation3.ui.NavDisplay
import androidx.lifecycle.viewmodel.navigation3.rememberViewModelStoreNavEntryDecorator
import kotlinx.serialization.Serializable

@Serializable data object Home
@Serializable data class Details(val movieId: String)
@Serializable data class Player(val movieId: String)

@Composable
fun TvNavHost() {
    val backStack = rememberNavBackStack(Home)

    NavDisplay(
        backStack = backStack,
        onBack = { backStack.removeLastOrNull() },
        entryDecorators = listOf(
            rememberSceneSetupNavEntryDecorator(),
            rememberSavedStateNavEntryDecorator(),
            rememberViewModelStoreNavEntryDecorator(),   // per-entry ViewModelStore for hiltViewModel()
        ),
        entryProvider = entryProvider {
            entry<Home> {
                HomeScreen(onOpen = { backStack.add(Details(it.id)) })
            }
            entry<Details> { key ->
                DetailsScreen(
                    movieId = key.movieId,
                    onPlay = { backStack.add(Player(key.movieId)) },
                )
            }
            entry<Player> { key -> PlayerScreen(movieId = key.movieId) }
        },
    )
}
```

The `rememberViewModelStoreNavEntryDecorator` scopes a `ViewModelStore` to each back-stack entry so `hiltViewModel()` returns a ViewModel tied to that destination's lifecycle. Ensure destination keys are `@Serializable` (apply the `kotlin-serialization` plugin) so state survives process death. On TV, remember that the **Back button (`KEYCODE_BACK`)** is the primary "up" affordance and Carousel/immersive components consume it to exit their sub-state before it pops the back stack.

## State, ViewModels, and the data layer

Expose UI state as a single immutable `StateFlow` from a `ViewModel`; collect it with **`collectAsStateWithLifecycle()`**, never `collectAsState()`. On TV the app is frequently backgrounded (user switches to another input or app); `collectAsStateWithLifecycle` stops collecting when the lifecycle drops below STARTED, which avoids wasted work and background crashes.

```kotlin
@HiltViewModel
class HomeViewModel @Inject constructor(
    private val repo: CatalogRepository,
) : ViewModel() {

    val uiState: StateFlow<HomeUiState> = repo.rows()
        .map { HomeUiState.Ready(it) }
        .stateIn(
            scope = viewModelScope,
            started = SharingStarted.WhileSubscribed(5_000),
            initialValue = HomeUiState.Loading,
        )
}

sealed interface HomeUiState {
    data object Loading : HomeUiState
    data class Ready(val rows: List<Row>) : HomeUiState
}

@Composable
fun HomeRoute(viewModel: HomeViewModel = hiltViewModel()) {
    val state by viewModel.uiState.collectAsStateWithLifecycle()
    when (val s = state) {
        HomeUiState.Loading -> LoadingScreen()
        is HomeUiState.Ready -> HomeScreen(rows = s.rows)
    }
}
```

Launch all coroutines in `viewModelScope` (or a lifecycle scope) — never `GlobalScope`, which leaks past the screen. `SharingStarted.WhileSubscribed(5_000)` keeps the upstream alive for 5 seconds after the last collector leaves, so a quick Back-then-forward does not restart the flow.

**Room** with KSP, coroutines, and `Flow`:

```kotlin
@Entity(tableName = "movies")
data class MovieEntity(
    @PrimaryKey val id: String,
    val title: String,
    val posterUrl: String,
)

@Dao
interface MovieDao {
    @Query("SELECT * FROM movies ORDER BY title")
    fun observeAll(): Flow<List<MovieEntity>>

    @Upsert
    suspend fun upsertAll(movies: List<MovieEntity>)
}

@Database(entities = [MovieEntity::class], version = 1, exportSchema = true)
abstract class AppDatabase : RoomDatabase() {
    abstract fun movieDao(): MovieDao
}
```

`Flow`-returning DAO queries emit on every table change; `suspend` write functions run off the main thread on Room's own executor. Keep the DAO's `Flow` cold and let `stateIn` in the ViewModel own the hot conversion. For small key/value settings (last-played position, preferences) use **DataStore Preferences**, not `SharedPreferences`:

```kotlin
val Context.dataStore by preferencesDataStore(name = "settings")
val SUBTITLES = booleanPreferencesKey("subtitles_on")

val subtitlesOn: Flow<Boolean> = context.dataStore.data.map { it[SUBTITLES] ?: false }
suspend fun setSubtitles(on: Boolean) { context.dataStore.edit { it[SUBTITLES] = on } }
```

`androidx.room` 2.x is in maintenance mode; a separate `androidx.room3` package is the KMP-focused, KSP-only successor. Stay on `androidx.room` 2.8.x unless you are deliberately adopting the multiplatform Room3 line.

## Dependency injection with Hilt

Hilt is the DI default. It runs on KSP; do not use kapt. Annotate the `Application`, the entry-point `Activity`, and ViewModels; bind implementations in modules.

```kotlin
@HiltAndroidApp
class TvApp : Application()

@AndroidEntryPoint
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent { TvAppTheme { TvNavHost() } }
    }
}

@Module
@InstallIn(SingletonComponent::class)
object DataModule {
    @Provides @Singleton
    fun provideDatabase(@ApplicationContext ctx: Context): AppDatabase =
        Room.databaseBuilder(ctx, AppDatabase::class.java, "tv.db").build()

    @Provides
    fun provideMovieDao(db: AppDatabase): MovieDao = db.movieDao()
}
```

`hiltViewModel()` (from `androidx.hilt:hilt-navigation-compose`) resolves ViewModels against the current `ViewModelStoreOwner`, which is the per-entry owner provided by the Navigation 3 decorator above.

## Media playback with Media3

Use **Media3 ExoPlayer** with the Compose-native `PlayerSurface` (`media3-ui-compose`); do not wrap the legacy `PlayerView` in `AndroidView` for new Compose UIs. The standalone `com.google.android.exoplayer2` library is discontinued — all development is in `androidx.media3`. The single hard rule: `ExoPlayer` owns a codec and buffers that **must be released**, and playback must pause when the screen leaves the foreground.

```kotlin
import androidx.media3.exoplayer.ExoPlayer
import androidx.media3.common.MediaItem
import androidx.media3.common.util.UnstableApi
import androidx.media3.ui.compose.PlayerSurface
import androidx.media3.ui.compose.SURFACE_TYPE_SURFACE_VIEW

@OptIn(UnstableApi::class)
@Composable
fun PlayerScreen(url: String) {
    val context = LocalContext.current
    val lifecycleOwner = LocalLifecycleOwner.current

    val player = remember {
        ExoPlayer.Builder(context).build().apply {
            setMediaItem(MediaItem.fromUri(url))
            prepare()
            playWhenReady = true
        }
    }

    // Pause/resume with the lifecycle; TV apps are backgrounded often.
    DisposableEffect(lifecycleOwner) {
        val observer = LifecycleEventObserver { _, event ->
            when (event) {
                Lifecycle.Event.ON_STOP -> player.pause()
                else -> Unit
            }
        }
        lifecycleOwner.lifecycle.addObserver(observer)
        onDispose { lifecycleOwner.lifecycle.removeObserver(observer) }
    }

    // Release exactly once, when the composable leaves composition for good.
    DisposableEffect(Unit) {
        onDispose { player.release() }
    }

    PlayerSurface(
        player = player,
        surfaceType = SURFACE_TYPE_SURFACE_VIEW,
        modifier = Modifier.fillMaxSize(),
    )
}
```

For anything beyond a throwaway screen, own the `ExoPlayer` in a `ViewModel` (released in `onCleared()`) so it survives configuration changes and you can drive state cleanly; the `PlayerScreen` above is the minimal correct excerpt. Use `SURFACE_TYPE_SURFACE_VIEW` for full-screen video (best power/quality); pair `PlayerSurface` with `rememberPresentationState()` to show a shutter until the first frame renders and to size to the video's aspect ratio. Add a **`MediaSession`** (from `media3-session`) so the system and remote transport keys (play/pause on the D-pad, external controllers) drive playback. For HLS/DASH streams add the matching `media3-exoplayer-hls`/`-dash` module; `MediaItem.fromUri` auto-detects the type.

## Images with Coil 3

Coil 3 is the image loader; use `AsyncImage` with a Compose `ContentScale`. Register a network backend (OkHttp shown in the catalog). Coil handles memory/disk caching and cancels the request when the composable leaves composition, so no manual cleanup is needed. Always pass `contentDescription` (or `null` for decorative art) and `ContentScale.Crop` for poster tiles so images fill focus-scaled cards without distortion.

## Testing

Write instrumented Compose UI tests with `createComposeRule()`. **The critical TV gotcha:** the default `ComposeUiTestConfig` now sets the initial input mode to `InputMode.Touch` at the start of each test, so a test that exercises D-pad focus will not behave like a real remote unless you request keyboard/D-pad mode explicitly:

```kotlin
import androidx.compose.ui.test.*
import androidx.compose.ui.test.junit4.createComposeRule
import androidx.compose.ui.input.InputMode
import androidx.compose.ui.input.key.Key

class HomeScreenTest {
    @get:Rule
    val rule = createComposeRule(ComposeUiTestConfig(inputMode = InputMode.Keyboard))

    @OptIn(ExperimentalTestApi::class)
    @Test
    fun dpadRightMovesFocusAcrossRow() {
        rule.setContent { TvAppTheme { MovieRow("Trending", fakeMovies, onClick = {}) } }

        rule.onNodeWithText("Movie 1").requestFocus()
        rule.onNodeWithText("Movie 1").performKeyInput {
            pressKey(Key.DirectionRight)
            pressKey(Key.DirectionRight)
        }
        rule.onNodeWithText("Movie 3").assertIsFocused()
    }
}
```

Note the Compose v2 test dispatcher (now the default) is a `StandardTestDispatcher`: recompositions queue rather than flushing immediately, so advance with `rule.waitForIdle()`/`runOnIdle` after injecting input. Test `Flow` emissions with **Turbine**:

```kotlin
@Test
fun emitsReadyAfterLoading() = runTest {
    val vm = HomeViewModel(fakeRepo)
    vm.uiState.test {
        assertEquals(HomeUiState.Loading, awaitItem())
        assertTrue(awaitItem() is HomeUiState.Ready)
        cancelAndIgnoreRemainingEvents()
    }
}
```

Use **MockK** for Kotlin-idiomatic mocking (`mockk`, `coEvery`, `coVerify` for suspend functions) and **Robolectric** for JVM-side Android unit tests. Robolectric 4.16 supports Android SDK 36 but requires JDK 21 to run tests targeting it, and it drops support for SDK 21–22 — which is fine given a minSdk of 23. Keep JUnit 4 as the runner — it is what Compose's `createComposeRule` and AndroidX test infra target.

## Tooling: lint, formatting, and shrinking

Two complementary tools cover static analysis:

- **Android Lint** (built into AGP) is the correctness backbone; keep it in the build and do not disable TV checks — `MissingTvBanner` and touchscreen/leanback checks catch manifest mistakes that make the app invisible on Play for TV. Run `./gradlew lint`.
- **ktlint** with the **compose-rules** ruleset (`io.nlopez.compose.rules:ktlint`) for formatting plus Compose-specific checks (missing `remember`, `MutableState` as a parameter, modifier ordering, composable naming). Configure through `.editorconfig`; the current compose-rules release tracks ktlint 1.8.x. detekt is a viable alternative, but its 2.x line — the one the newest compose-rules targets — is still in alpha, so ktlint is the stable default here.

`.editorconfig` (excerpt):

```editorconfig
[*.{kt,kts}]
ktlint_function_naming_ignore_when_annotated_with = Composable
ktlint_compose_compositionlocal-allowlist = disabled
```

Enable **R8** in release (`isMinifyEnabled = true`, `isShrinkResources = true`) with `proguard-android-optimize.txt`. Room needs no extra rules when using KSP; kotlinx-serialization ships its own consumer rules, so `@Serializable` classes survive shrinking without hand-written keep rules. Common commands:

```bash
./gradlew :app:assembleDebug          # build
./gradlew ktlintFormat                # format
./gradlew ktlintCheck lint            # style + correctness
./gradlew testDebugUnitTest           # JVM unit + Robolectric tests
./gradlew connectedDebugAndroidTest   # instrumented Compose UI tests on a TV emulator
```

## Anti-patterns to avoid

| Wrong | Why | Right |
| --- | --- | --- |
| `androidx.compose.material3.Button`/`Card` for interactive TV elements | No D-pad focus scale/glow/border; ripple is invisible without touch | `androidx.tv.material3` `Surface`/`Button`/`Card` with `Clickable*Defaults` |
| `androidx.tv.foundation.TvLazyRow` / `TvLazyColumn` | Deprecated; the TV lazy layouts moved into `androidx.compose.foundation` | `LazyRow`/`LazyColumn` + `Modifier.focusRestorer().focusGroup()` |
| `LazyRow` without `focusRestorer()` | Focus snaps back to the first item every time the row is re-entered | Chain `.focusRestorer()` **before** `.focusGroup()` |
| `items(list)` with no `key` | Focus and scroll position lost on data change; no skipping | `items(list, key = { it.id }, contentType = { ... })` |
| `focusRequester.requestFocus()` during composition | Node not attached yet — throws | Request inside `LaunchedEffect(Unit)` |
| `createComposeRule()` (default) for D-pad tests | Defaults to `InputMode.Touch`; D-pad behavior untested | `createComposeRule(ComposeUiTestConfig(inputMode = InputMode.Keyboard))` |
| `collectAsState()` in a screen | Keeps collecting while backgrounded; wasted work/crashes | `collectAsStateWithLifecycle()` |
| Not releasing `ExoPlayer` | Codec/buffer leak, ANRs | `DisposableEffect { onDispose { player.release() } }` |
| `PlayerView` in `AndroidView` for new Compose UI | Legacy view interop, extra lifecycle glue | `PlayerSurface` from `media3-ui-compose` |
| `mutableStateOf(0)` for `Int`/`Float` | Autoboxing on every write | `mutableIntStateOf(0)` / `mutableFloatStateOf(0f)` |
| Passing `MutableState<T>` into a composable | Split state ownership | Hoist: pass value down, events up |
| `kapt` for Room/Hilt | Slow Java stub pass | `ksp(...)` |
| `GlobalScope.launch { }` | Leaks past the screen | `viewModelScope` / lifecycle scope |
| Missing banner or `touchscreen required="false"` | App absent from Play on TV; `MissingTvBanner` lint failure | 320×180 localized `android:banner` in `drawable-xhdpi` + touchscreen not required + `LEANBACK_LAUNCHER` |
| Target API below 34 for a TV build | Play blocks new/updated TV apps | `targetSdk = 34`+ (36 recommended) |

## Version & compatibility

| Component | Release line / version | Notes |
| --- | --- | --- |
| Kotlin | 2.4 (2.4.10) | K2 compiler; context parameters, explicit backing fields, UUID API stable |
| AGP | 9.2 | Supports compileSdk 37; built-in Kotlin (no `kotlin-android` apply needed) |
| Gradle | 9.5 | Kotlin 2.4.10 min 7.6.3, fully supported through 9.5.0 |
| JDK toolchain | 21 | Required to run Robolectric tests targeting SDK 36 |
| compileSdk / targetSdk / minSdk | 37 / 36 / 23 | TV Play floor is targetSdk 34; 36 is a safe current choice. minSdk 23 is the AndroidX floor |
| Compose BOM | 2026.08.00 (Compose 1.12) | Requires AGP 9.1.1+; v2 test APIs are the default |
| androidx.tv:tv-material | 1.1.0 | tv-foundation lazy layouts deprecated → use `compose.foundation` |
| Media3 (ExoPlayer, ui-compose, session) | 1.11.0 | `PlayerSurface` Compose UI; standalone ExoPlayer2 discontinued |
| Navigation 3 | 1.1.6 | Stable, Compose-first; keys must be `@Serializable` |
| Lifecycle | 2.11.0 | Requires AGP 9.2+; provides `viewmodel-navigation3` decorator. Drop to 2.9.x if you must stay below AGP 9.2 |
| Hilt | 2.59.2 + androidx.hilt 1.4.0 | KSP only |
| KSP | 2.3.10 | KSP2 uses decoupled versioning; no `2.4.0-x.x.x` stable exists — this line is compatible with Kotlin 2.4.x |
| Room | 2.8.4 | Classic `androidx.room` + KSP; `androidx.room3` is the separate KMP successor |
| Coroutines / Serialization | 1.11.0 / 1.11.0 | |
| Coil | 3.6.1 | `AsyncImage`; register a network backend |
| DataStore | 1.2.1 | Preferences over `SharedPreferences` |
| Testing | JUnit 4.13.2, Turbine 1.2.1, MockK 1.14.7, Robolectric 4.16.1 | Compose test v2 default dispatcher is `StandardTestDispatcher` |
| Lint/format | Android Lint (AGP) + ktlint 1.8.x + compose-rules | detekt 2.x still alpha — not the default here |

- **Research date:** September 5, 2026
