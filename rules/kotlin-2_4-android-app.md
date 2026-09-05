---
type: "agent_requested"
description: "Kotlin Android Compose (Material 3 Adaptive + Navigation 3 + Hilt) coding guidelines"
---
# Kotlin 2.4 + Jetpack Compose on Android 17: Adaptive, Navigation 3, and Hilt

This stack targets phones and tablets from a single adaptive codebase. Kotlin 2.4 (K2 only; the K1 frontend is removed) compiles against Android 17 (API 37) while deploying to minSdk 30. The UI is 100% Compose with Material 3, laid out adaptively with Material 3 Adaptive, navigated with the state-driven Navigation 3 library, and wired with Hilt over KSP. Optimize for: unidirectional data flow with hoisted state, `@Serializable` `NavKey` back stacks you own as plain lists, window-size-class–driven layout instead of hardcoded `sw600dp` branches, and lifecycle-aware Flow collection.

The biggest ways agents write wrong-but-plausible code here come from importing habits from older Android and adjacent ecosystems: reaching for `NavController`/`NavHost` and string routes (that is Navigation 2, a different library), applying the `kotlin-android` Gradle plugin (AGP 9 has built-in Kotlin and forbids it with the new DSL), using `kapt` for Hilt/Room (this stack is KSP-only), collecting Flows with `collectAsState()` instead of `collectAsStateWithLifecycle()`, branching layout on `Configuration.screenWidthDp` instead of `WindowSizeClass`, and treating tablets as a scaled-up phone instead of using list-detail scenes. Everything below assumes the exact version baseline in the final table.

## Project setup and the version catalog

AGP 9.1 provides **built-in Kotlin** — do not apply `org.jetbrains.kotlin.android`; it is incompatible with the new DSL and will fail the build. Because AGP 9.1 has a runtime dependency on Kotlin Gradle Plugin 2.2.10 for built-in Kotlin but this stack requires the Kotlin 2.4 line, override it by declaring the Kotlin-versioned plugins (Compose compiler, serialization) at 2.4.10 in the root `plugins` block; that pulls KGP 2.4.10 onto the build classpath for all modules. Keep every exact version in `gradle/libs.versions.toml`; never scatter versions through module files.

```toml
# gradle/libs.versions.toml
[versions]
kotlin = "2.4.10"
agp = "9.1.1"
ksp = "2.3.11"                       # KSP is versioned independently of Kotlin since 2.3.0
composeBom = "2026.08.00"           # Compose 1.12 (August '26 release)
material3Adaptive = "1.3.0"         # adaptive, adaptive-layout, adaptive-navigation, adaptive-navigation3
navigation3 = "1.1.7"
lifecycle = "2.10.0"
lifecycleNav3 = "2.10.0"            # androidx.lifecycle:lifecycle-viewmodel-navigation3
hilt = "2.60.1"                     # Dagger/Hilt; 2.59.0 has the ComponentTreeDeps bug on AGP 9 — do not use
androidxHilt = "1.3.0"             # hilt-navigation-compose, hilt-lifecycle-viewmodel-compose, hilt-compiler
room = "3.0.2"                      # androidx.room3 (KMP-ready, KSP-only, Kotlin codegen)
datastore = "1.1.7"
coroutines = "1.10.2"
serialization = "1.9.0"
retrofit = "3.0.0"
okhttp = "5.1.0"
coil = "3.6.1"
turbine = "1.2.1"
mockk = "1.14.2"
robolectric = "4.15.1"
junit = "4.13.2"
detekt = "1.23.8"
composeRules = "0.4.28"            # io.nlopez.compose.rules detekt plugin
ktfmt = "0.56"

[libraries]
# Compose (BOM-managed — no versions)
compose-bom = { group = "androidx.compose", name = "compose-bom", version.ref = "composeBom" }
compose-ui = { group = "androidx.compose.ui", name = "ui" }
compose-ui-tooling = { group = "androidx.compose.ui", name = "ui-tooling" }
compose-ui-tooling-preview = { group = "androidx.compose.ui", name = "ui-tooling-preview" }
compose-ui-test-junit4 = { group = "androidx.compose.ui", name = "ui-test-junit4" }
compose-ui-test-manifest = { group = "androidx.compose.ui", name = "ui-test-manifest" }
compose-material3 = { group = "androidx.compose.material3", name = "material3" }
compose-material-icons-extended = { group = "androidx.compose.material", name = "material-icons-extended" }
material3-adaptive-navigation-suite = { group = "androidx.compose.material3", name = "material3-adaptive-navigation-suite" }

# Material 3 Adaptive (own version group)
adaptive = { group = "androidx.compose.material3.adaptive", name = "adaptive", version.ref = "material3Adaptive" }
adaptive-layout = { group = "androidx.compose.material3.adaptive", name = "adaptive-layout", version.ref = "material3Adaptive" }
adaptive-navigation3 = { group = "androidx.compose.material3.adaptive", name = "adaptive-navigation3", version.ref = "material3Adaptive" }

# Navigation 3
navigation3-runtime = { group = "androidx.navigation3", name = "navigation3-runtime", version.ref = "navigation3" }
navigation3-ui = { group = "androidx.navigation3", name = "navigation3-ui", version.ref = "navigation3" }
lifecycle-viewmodel-navigation3 = { group = "androidx.lifecycle", name = "lifecycle-viewmodel-navigation3", version.ref = "lifecycleNav3" }

# Lifecycle / Activity
lifecycle-viewmodel-compose = { group = "androidx.lifecycle", name = "lifecycle-viewmodel-compose", version.ref = "lifecycle" }
lifecycle-runtime-compose = { group = "androidx.lifecycle", name = "lifecycle-runtime-compose", version.ref = "lifecycle" }
activity-compose = { group = "androidx.activity", name = "activity-compose", version = "1.11.0" }

# Hilt
hilt-android = { group = "com.google.dagger", name = "hilt-android", version.ref = "hilt" }
hilt-compiler = { group = "com.google.dagger", name = "hilt-compiler", version.ref = "hilt" }
hilt-navigation-compose = { group = "androidx.hilt", name = "hilt-navigation-compose", version.ref = "androidxHilt" }
hilt-lifecycle-viewmodel-compose = { group = "androidx.hilt", name = "hilt-lifecycle-viewmodel-compose", version.ref = "androidxHilt" }
androidx-hilt-compiler = { group = "androidx.hilt", name = "hilt-compiler", version.ref = "androidxHilt" }

# Data
room-runtime = { group = "androidx.room3", name = "room3-runtime", version.ref = "room" }
room-compiler = { group = "androidx.room3", name = "room3-compiler", version.ref = "room" }
datastore-preferences = { group = "androidx.datastore", name = "datastore-preferences", version.ref = "datastore" }
retrofit = { group = "com.squareup.retrofit2", name = "retrofit", version.ref = "retrofit" }
retrofit-serialization = { group = "com.squareup.retrofit2", name = "converter-kotlinx-serialization", version.ref = "retrofit" }
okhttp-logging = { group = "com.squareup.okhttp3", name = "logging-interceptor", version.ref = "okhttp" }
serialization-json = { group = "org.jetbrains.kotlinx", name = "kotlinx-serialization-json", version.ref = "serialization" }
coroutines-android = { group = "org.jetbrains.kotlinx", name = "kotlinx-coroutines-android", version.ref = "coroutines" }
coil-compose = { group = "io.coil-kt.coil3", name = "coil-compose", version.ref = "coil" }
coil-network-okhttp = { group = "io.coil-kt.coil3", name = "coil-network-okhttp", version.ref = "coil" }

# Test
junit = { group = "junit", name = "junit", version.ref = "junit" }
coroutines-test = { group = "org.jetbrains.kotlinx", name = "kotlinx-coroutines-test", version.ref = "coroutines" }
turbine = { group = "app.cash.turbine", name = "turbine", version.ref = "turbine" }
mockk = { group = "io.mockk", name = "mockk", version.ref = "mockk" }
robolectric = { group = "org.robolectric", name = "robolectric", version.ref = "robolectric" }
hilt-android-testing = { group = "com.google.dagger", name = "hilt-android-testing", version.ref = "hilt" }
compose-rules-detekt = { group = "io.nlopez.compose.rules", name = "detekt", version.ref = "composeRules" }

[plugins]
android-application = { id = "com.android.application", version.ref = "agp" }
android-library = { id = "com.android.library", version.ref = "agp" }
kotlin-compose = { id = "org.jetbrains.kotlin.plugin.compose", version.ref = "kotlin" }
kotlin-serialization = { id = "org.jetbrains.kotlin.plugin.serialization", version.ref = "kotlin" }
ksp = { id = "com.google.devtools.ksp", version.ref = "ksp" }
hilt = { id = "com.google.dagger.hilt.android", version.ref = "hilt" }
detekt = { id = "io.gitlab.arturbosch.detekt", version.ref = "detekt" }
```

```kotlin
// build.gradle.kts (root) — declaring the Kotlin-versioned plugins forces KGP 2.4.10 onto the classpath,
// overriding the KGP 2.2.10 that AGP 9.1 has as its runtime dependency for built-in Kotlin.
plugins {
    alias(libs.plugins.android.application) apply false
    alias(libs.plugins.android.library) apply false
    alias(libs.plugins.kotlin.compose) apply false
    alias(libs.plugins.kotlin.serialization) apply false
    alias(libs.plugins.ksp) apply false
    alias(libs.plugins.hilt) apply false
    alias(libs.plugins.detekt)
}
```

```kotlin
// app/build.gradle.kts — note: NO org.jetbrains.kotlin.android plugin.
plugins {
    alias(libs.plugins.android.application)
    alias(libs.plugins.kotlin.compose)
    alias(libs.plugins.kotlin.serialization)
    alias(libs.plugins.ksp)
    alias(libs.plugins.hilt)
}

android {
    namespace = "com.example.field"
    compileSdk = 37

    defaultConfig {
        applicationId = "com.example.field"
        minSdk = 30
        targetSdk = 37
        versionCode = 1
        versionName = "1.0"
        testInstrumentationRunner = "com.example.field.HiltTestRunner"
    }

    buildFeatures { compose = true }

    compileOptions {
        sourceCompatibility = JavaVersion.VERSION_17
        targetCompatibility = JavaVersion.VERSION_17
    }
    // Built-in Kotlin exposes the Kotlin DSL under `kotlin { }`:
    kotlin { jvmToolchain(17) }
}

dependencies {
    implementation(platform(libs.compose.bom))
    androidTestImplementation(platform(libs.compose.bom))

    implementation(libs.compose.ui)
    implementation(libs.compose.ui.tooling.preview)
    debugImplementation(libs.compose.ui.tooling)
    implementation(libs.compose.material3)
    implementation(libs.compose.material.icons.extended)
    implementation(libs.material3.adaptive.navigation.suite)

    implementation(libs.adaptive)
    implementation(libs.adaptive.layout)
    implementation(libs.adaptive.navigation3)

    implementation(libs.navigation3.runtime)
    implementation(libs.navigation3.ui)
    implementation(libs.lifecycle.viewmodel.navigation3)

    implementation(libs.lifecycle.viewmodel.compose)
    implementation(libs.lifecycle.runtime.compose)
    implementation(libs.activity.compose)

    implementation(libs.hilt.android)
    ksp(libs.hilt.compiler)
    implementation(libs.hilt.navigation.compose)
    implementation(libs.hilt.lifecycle.viewmodel.compose)
    ksp(libs.androidx.hilt.compiler)

    implementation(libs.room.runtime)
    ksp(libs.room.compiler)
    implementation(libs.datastore.preferences)
    implementation(libs.retrofit)
    implementation(libs.retrofit.serialization)
    implementation(libs.okhttp.logging)
    implementation(libs.serialization.json)
    implementation(libs.coroutines.android)
    implementation(libs.coil.compose)
    implementation(libs.coil.network.okhttp)

    testImplementation(libs.junit)
    testImplementation(libs.coroutines.test)
    testImplementation(libs.turbine)
    testImplementation(libs.mockk)
    testImplementation(libs.robolectric)
    androidTestImplementation(libs.compose.ui.test.junit4)
    debugImplementation(libs.compose.ui.test.manifest)
    androidTestImplementation(libs.hilt.android.testing)
    kspAndroidTest(libs.hilt.compiler)
}
```

Gradle must be 9.1+ (AGP 9.1 requires it) and the build JDK must be 17+. `compileSdk = 37` needs AGP 9.1.1 minimum, which Compose 1.12 (BOM `2026.08.00`) also requires. A new SDK does not move your deployment floor: `minSdk = 30` is the constraint, and any API above it goes behind a check.

## Composable fundamentals and state

State flows down, events flow up. Hoist state to the lowest common caller; keep composables that render given state separate from those that own it. Read `MutableState` with `by` delegation, and remember to key `remember` on the inputs the computation depends on so it recomputes when they change.

```kotlin
import androidx.compose.runtime.*
import androidx.compose.foundation.layout.*
import androidx.compose.material3.*
import androidx.compose.material.icons.Icons
import androidx.compose.material.icons.filled.Star
import androidx.compose.material.icons.outlined.StarBorder
import androidx.compose.ui.Modifier

// Stateless: fully driven by parameters; trivial to preview and test.
@Composable
fun SiteCard(
    site: Site,
    onToggleFavorite: (siteId: String) -> Unit,
    modifier: Modifier = Modifier,
) {
    ElevatedCard(modifier = modifier.fillMaxWidth()) {
        ListItem(
            headlineContent = { Text(site.name) },
            supportingContent = { Text(site.region) },
            trailingContent = {
                IconButton(onClick = { onToggleFavorite(site.id) }) {
                    Icon(
                        imageVector = if (site.isFavorite) Icons.Filled.Star else Icons.Outlined.StarBorder,
                        contentDescription = if (site.isFavorite) "Unfavorite" else "Favorite",
                    )
                }
            },
        )
    }
}
```

Rules that prevent the common defects:

- **`Modifier` is the first optional parameter, defaulting to `Modifier`, applied to the root layout node.** Never capture a `Modifier` and reuse it on multiple nodes. Modifier order is significant: `padding().background()` differs from `background().padding()`.
- Compose stability is a contract. Prefer stable, immutable model types (`data class` of `val`s over stable types). Types the compiler cannot prove stable — e.g. `List<T>` from `kotlin.collections` — cause extra recomposition; expose `kotlinx.collections.immutable` `ImmutableList` or `PersistentList` from state holders when a lambda closes over collections.
- Never do real work in the composition body. Side effects belong in `LaunchedEffect` (keyed on what should restart it), `DisposableEffect` (for cleanup), or `rememberCoroutineScope()` for event-driven launches. The August '26 release (Compose 1.12) added a keyed `SideEffect(key1, ...)` overload for one-shot effects that need neither a coroutine nor a dispose block.
- Every previewable screen gets `@Preview`; use `@PreviewScreenSizes` (or a multi-preview) to render phone and tablet widths at once.

## Adaptive layout: window size classes and panes

Never branch UI on raw `dp` from `Configuration`. The single source of truth is `currentWindowAdaptiveInfo()`, which yields a `WindowAdaptiveInfo` carrying the `WindowSizeClass` and the device `Posture` (foldable, tabletop). Use breakpoint checks, not hardcoded numbers.

```kotlin
import androidx.compose.material3.adaptive.currentWindowAdaptiveInfo
import androidx.window.core.layout.WindowSizeClass.Companion.WIDTH_DP_EXPANDED_LOWER_BOUND

@Composable
fun rememberIsExpandedWidth(): Boolean {
    val info = currentWindowAdaptiveInfo()
    return info.windowSizeClass.isWidthAtLeastBreakpoint(WIDTH_DP_EXPANDED_LOWER_BOUND)
}
```

For top-level navigation that must become a bar on compact widths and a rail/drawer on wider ones, use `NavigationSuiteScaffold` — it picks the component from the adaptive info automatically. Do not build your own bar/rail switch.

```kotlin
import androidx.compose.material3.adaptive.navigationsuite.NavigationSuiteScaffold
import androidx.compose.ui.res.stringResource

@Composable
fun FieldAppShell(
    selected: TopDestination,
    onSelect: (TopDestination) -> Unit,
    content: @Composable () -> Unit,
) {
    NavigationSuiteScaffold(
        navigationSuiteItems = {
            TopDestination.entries.forEach { dest ->
                item(
                    selected = dest == selected,
                    onClick = { onSelect(dest) },
                    icon = { Icon(dest.icon, contentDescription = null) },
                    label = { Text(stringResource(dest.labelRes)) },
                )
            }
        },
        content = content,
    )
}
```

Do **not** hand-roll a two-pane `Row` for tablets. The idiomatic list-detail pattern is delivered through Navigation 3 scenes (next section) via `ListDetailSceneStrategy`; when you need a self-contained pane layout outside the nav back stack, `ListDetailPaneScaffold` with `rememberListDetailPaneScaffoldNavigator` and `SupportingPaneScaffold` are the building blocks. `AnimatedPane` wraps each pane so pane show/hide animates correctly.

## Navigation 3

This is the part agents most often get wrong by importing Navigation 2 habits. There is no `NavController`, no `NavHost`, no string routes, no `NavGraphBuilder`. **The back stack is a plain observable `List` you own.** `NavDisplay` renders the top of it; you mutate it with `add`/`removeLastOrNull`.

Define routes as `@Serializable` types implementing `NavKey`. Serialization is required so `rememberNavBackStack` can persist the stack across configuration change and process death.

```kotlin
import androidx.navigation3.runtime.NavKey
import kotlinx.serialization.Serializable

@Serializable data object SiteList : NavKey
@Serializable data class SiteDetail(val siteId: String) : NavKey
@Serializable data class Inspection(val siteId: String, val inspectionId: String) : NavKey
```

A minimal, production-correct display wires the two decorators that make ViewModels and saved state work per-entry. `rememberViewModelStoreNavEntryDecorator` scopes a distinct `ViewModelStore` to each `NavEntry` (so `SiteDetail("a")` and `SiteDetail("b")` get separate ViewModels), and `rememberSaveableStateHolderNavEntryDecorator` preserves `rememberSaveable` state per entry.

```kotlin
import androidx.navigation3.runtime.entry
import androidx.navigation3.runtime.entryProvider
import androidx.navigation3.runtime.rememberNavBackStack
import androidx.navigation3.runtime.rememberSaveableStateHolderNavEntryDecorator
import androidx.navigation3.ui.NavDisplay
import androidx.lifecycle.viewmodel.navigation3.rememberViewModelStoreNavEntryDecorator
import androidx.hilt.lifecycle.viewmodel.compose.hiltViewModel
import androidx.compose.ui.Modifier

@Composable
fun FieldNavDisplay(modifier: Modifier = Modifier) {
    val backStack = rememberNavBackStack(SiteList)

    NavDisplay(
        backStack = backStack,
        modifier = modifier,
        onBack = { count -> repeat(count) { backStack.removeLastOrNull() } },
        entryDecorators = listOf(
            rememberSaveableStateHolderNavEntryDecorator(),
            rememberViewModelStoreNavEntryDecorator(),
        ),
        entryProvider = entryProvider {
            entry<SiteList> {
                val vm: SiteListViewModel = hiltViewModel()
                SiteListScreen(
                    viewModel = vm,
                    onOpenSite = { id -> backStack.add(SiteDetail(id)) },
                )
            }
            entry<SiteDetail> { key ->
                val vm: SiteDetailViewModel = hiltViewModel()
                SiteDetailScreen(
                    viewModel = vm,
                    onOpenInspection = { inspId ->
                        backStack.add(Inspection(key.siteId, inspId))
                    },
                    onBack = { backStack.removeLastOrNull() },
                )
            }
            entry<Inspection> { key ->
                InspectionScreen(inspectionId = key.inspectionId)
            }
        },
    )
}
```

Key behaviours to respect:

- `onBack` receives the number of entries the gesture requests popping; pop that many, do not assume one. Predictive back is handled by `NavDisplay` automatically once you pop from the list.
- `hiltViewModel()` inside an `entry` block resolves against the entry's `ViewModelStore` because of the decorator — you never pass a `key`.
- To survive process death, keep using `rememberNavBackStack`; a bare `remember { mutableStateListOf(...) }` only survives recomposition.

### Adaptive list-detail with a scene strategy

For list-detail that collapses to one pane on phones and shows two side-by-side on tablets, add `ListDetailSceneStrategy` and tag entries with pane metadata. The strategy reads window size and rearranges panes; you write one navigation graph for both form factors.

```kotlin
import androidx.compose.material3.adaptive.ExperimentalMaterial3AdaptiveApi
import androidx.compose.material3.adaptive.navigation3.ListDetailSceneStrategy
import androidx.compose.material3.adaptive.navigation3.rememberListDetailSceneStrategy

@OptIn(ExperimentalMaterial3AdaptiveApi::class)
@Composable
fun SitesListDetail(modifier: Modifier = Modifier) {
    val backStack = rememberNavBackStack(SiteList)
    val sceneStrategy = rememberListDetailSceneStrategy<NavKey>()

    NavDisplay(
        backStack = backStack,
        modifier = modifier,
        sceneStrategy = sceneStrategy,
        onBack = { count -> repeat(count) { backStack.removeLastOrNull() } },
        entryDecorators = listOf(
            rememberSaveableStateHolderNavEntryDecorator(),
            rememberViewModelStoreNavEntryDecorator(),
        ),
        entryProvider = entryProvider {
            entry<SiteList>(
                metadata = ListDetailSceneStrategy.listPane(
                    detailPlaceholder = { Text("Select a site") },
                ),
            ) {
                val vm: SiteListViewModel = hiltViewModel()
                SiteListScreen(vm, onOpenSite = { id -> backStack.add(SiteDetail(id)) })
            }
            entry<SiteDetail>(
                metadata = ListDetailSceneStrategy.detailPane(),
            ) { key ->
                val vm: SiteDetailViewModel = hiltViewModel()
                SiteDetailScreen(vm, onOpenInspection = {}, onBack = { backStack.removeLastOrNull() })
            }
        },
    )
}
```

On a two-pane layout the detail screen must not draw its own back arrow and must not go full-screen, because it sits beside the list. Guidance for the API itself: use `SupportingPaneSceneStrategy` (via `rememberSupportingPaneSceneStrategy`) for a main+supporting arrangement, and combine strategies by passing several to `NavDisplay` when a screen (e.g. a dialog) needs its own scene.

### Modular navigation with Hilt

In a multi-module app, let each feature contribute its `entry` blocks through Hilt multibindings so the app module never imports feature internals. Inject a shared navigator and the set of entry-provider installers.

```kotlin
// :core:navigation — a plain observable back stack owned as a singleton
import androidx.compose.runtime.mutableStateListOf
import androidx.navigation3.runtime.EntryProviderScope
import androidx.navigation3.runtime.NavKey
import javax.inject.Inject
import javax.inject.Singleton

@Singleton
class Navigator @Inject constructor() {
    val backStack = mutableStateListOf<NavKey>(SiteList)
    fun goTo(key: NavKey) { backStack.add(key) }
    fun goBack() { backStack.removeLastOrNull() }
}

// A function that adds entries to the builder scope.
typealias EntryProviderInstaller = EntryProviderScope<NavKey>.() -> Unit
```

```kotlin
// :feature:sites — contributes its entries without the app knowing about them
import dagger.Module
import dagger.Provides
import dagger.hilt.InstallIn
import dagger.hilt.android.components.ActivityRetainedComponent
import dagger.multibindings.IntoSet

@Module
@InstallIn(ActivityRetainedComponent::class)
object SitesNavModule {
    @Provides
    @IntoSet
    fun provideSitesEntries(navigator: Navigator): EntryProviderInstaller = {
        entry<SiteList> {
            val vm: SiteListViewModel = hiltViewModel()
            SiteListScreen(vm, onOpenSite = { navigator.goTo(SiteDetail(it)) })
        }
        entry<SiteDetail> { key ->
            SiteDetailScreen(hiltViewModel(), onOpenInspection = {}, onBack = navigator::goBack)
        }
    }
}
```

```kotlin
// :app
@AndroidEntryPoint
class MainActivity : ComponentActivity() {
    @Inject lateinit var navigator: Navigator
    @Inject lateinit var installers: Set<@JvmSuppressWildcards EntryProviderInstaller>

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
            FieldTheme {
                NavDisplay(
                    backStack = navigator.backStack,
                    onBack = { count -> repeat(count) { navigator.goBack() } },
                    entryDecorators = listOf(
                        rememberSaveableStateHolderNavEntryDecorator(),
                        rememberViewModelStoreNavEntryDecorator(),
                    ),
                    entryProvider = entryProvider { installers.forEach { it() } },
                )
            }
        }
    }
}
```

## Dependency injection with Hilt (KSP)

Hilt runs on KSP here — `kapt` is not used anywhere in this stack (Room 3 also dropped it entirely). Annotate the `Application` with `@HiltAndroidApp`, entry-point Android classes with `@AndroidEntryPoint`, and constructor-inject everywhere else. Bind interfaces with `@Binds` in an `abstract` module; use `@Provides` only for types you do not own.

```kotlin
@HiltAndroidApp
class FieldApp : Application()

@Module
@InstallIn(SingletonComponent::class)
abstract class RepositoryModule {
    @Binds
    abstract fun bindSiteRepository(impl: DefaultSiteRepository): SiteRepository
}

@Module
@InstallIn(SingletonComponent::class)
object NetworkModule {
    @Provides
    @Singleton
    fun provideJson(): Json = Json { ignoreUnknownKeys = true; explicitNulls = false }

    @Provides
    @Singleton
    fun provideOkHttp(): OkHttpClient =
        OkHttpClient.Builder()
            .addInterceptor(HttpLoggingInterceptor().apply { level = HttpLoggingInterceptor.Level.BASIC })
            .build()

    @Provides
    @Singleton
    fun provideRetrofit(client: OkHttpClient, json: Json): Retrofit =
        Retrofit.Builder()
            .baseUrl("https://api.example.com/")
            .client(client)
            .addConverterFactory(json.asConverterFactory("application/json".toMediaType()))
            .build()

    @Provides
    @Singleton
    fun provideSiteApi(retrofit: Retrofit): SiteApi = retrofit.create(SiteApi::class.java)
}
```

ViewModels use `@HiltViewModel` with constructor injection; `SavedStateHandle` is injectable and, in Navigation 3, is populated from the entry's `NavKey` fields.

```kotlin
@HiltViewModel
class SiteDetailViewModel @Inject constructor(
    savedStateHandle: SavedStateHandle,
    private val repository: SiteRepository,
) : ViewModel() {
    private val route = savedStateHandle.toRoute<SiteDetail>()
    // ... uiState below ...
}
```

When a ViewModel needs a runtime value that is not in the `NavKey`, use assisted injection: annotate the ViewModel `@HiltViewModel(assistedFactory = ...)`, mark the runtime parameter `@Assisted`, and pass a `creationCallback` to `hiltViewModel`. Prefer putting identifiers in the `NavKey` and reading them through `SavedStateHandle`; reserve assisted injection for genuinely non-serializable inputs.

## State holders, ViewModels, and UDF

Model each screen's state as a single immutable `sealed interface` or `data class`, expose it as a `StateFlow`, and turn cold repository flows into hot UI state with `stateIn`. `WhileSubscribed(5_000)` keeps the upstream alive briefly across configuration change without leaking it.

```kotlin
import kotlinx.collections.immutable.ImmutableList
import kotlinx.collections.immutable.toImmutableList
import kotlinx.coroutines.flow.*

sealed interface SiteDetailUiState {
    data object Loading : SiteDetailUiState
    data class Ready(val site: Site, val inspections: ImmutableList<Inspection>) : SiteDetailUiState
    data class Error(val message: String) : SiteDetailUiState
}

@HiltViewModel
class SiteDetailViewModel @Inject constructor(
    savedStateHandle: SavedStateHandle,
    repository: SiteRepository,
) : ViewModel() {
    private val route = savedStateHandle.toRoute<SiteDetail>()

    val uiState: StateFlow<SiteDetailUiState> =
        repository.observeSite(route.siteId)
            .map { site ->
                SiteDetailUiState.Ready(site, site.inspections.toImmutableList())
            }
            .catch { emit(SiteDetailUiState.Error(it.message ?: "Unknown error")) }
            .stateIn(
                scope = viewModelScope,
                started = SharingStarted.WhileSubscribed(5_000),
                initialValue = SiteDetailUiState.Loading,
            )
}
```

One-off events (navigation, snackbars) do not belong in the state object — a re-emitted `StateFlow` would replay them after rotation. Model them as explicit consumable state or a `Channel` exposed as a `receiveAsFlow()`, collected once inside a lifecycle-aware effect.

## Coroutines and Flow in the UI

Collect Flows with `collectAsStateWithLifecycle()`, never `collectAsState()` — the latter keeps collecting while the app is in the background, wasting work and risking stale UI. Launch UI-triggered coroutines from `rememberCoroutineScope()`, and never block the main thread.

```kotlin
import androidx.lifecycle.compose.collectAsStateWithLifecycle

@Composable
fun SiteDetailScreen(
    viewModel: SiteDetailViewModel,
    onOpenInspection: (String) -> Unit,
    onBack: () -> Unit,
) {
    val state by viewModel.uiState.collectAsStateWithLifecycle()

    when (val s = state) {
        SiteDetailUiState.Loading -> CircularProgressIndicator()
        is SiteDetailUiState.Error -> Text(s.message)
        is SiteDetailUiState.Ready -> SiteDetailContent(s.site, s.inspections, onOpenInspection)
    }
}
```

Repository suspend functions must be main-safe: switch to `Dispatchers.IO` internally rather than requiring callers to wrap them. Inject dispatchers (do not hardcode) so tests can substitute a test dispatcher. All work launched in `viewModelScope` is cancelled automatically when the ViewModel clears; do not manage that lifecycle by hand.

## Data layer

**Room 3** (`androidx.room3`) is the persistence default: it is coroutine-first, generates Kotlin, and is KSP-only. Expose reactive reads as `Flow`, and suspend one-shot writes.

```kotlin
import androidx.room3.*
import kotlinx.coroutines.flow.Flow

@Entity(tableName = "sites")
data class SiteEntity(
    @PrimaryKey val id: String,
    val name: String,
    val region: String,
    val isFavorite: Boolean,
)

@Dao
interface SiteDao {
    @Query("SELECT * FROM sites ORDER BY name")
    fun observeAll(): Flow<List<SiteEntity>>

    @Upsert
    suspend fun upsertAll(sites: List<SiteEntity>)

    @Query("UPDATE sites SET isFavorite = :favorite WHERE id = :id")
    suspend fun setFavorite(id: String, favorite: Boolean)
}

@Database(entities = [SiteEntity::class], version = 1)
abstract class FieldDatabase : RoomDatabase() {
    abstract fun siteDao(): SiteDao
}
```

```kotlin
// Room 3 is fully backed by the androidx.sqlite driver API. For Android, install AndroidSQLiteDriver.
import androidx.room3.Room
import androidx.sqlite.driver.AndroidSQLiteDriver

@Module
@InstallIn(SingletonComponent::class)
object DatabaseModule {
    @Provides
    @Singleton
    fun provideDatabase(@ApplicationContext context: Context): FieldDatabase =
        Room.databaseBuilder(context, FieldDatabase::class.java, "field.db")
            .setDriver(AndroidSQLiteDriver())
            .setQueryCoroutineContext(Dispatchers.IO)
            .build()

    @Provides fun provideSiteDao(db: FieldDatabase): SiteDao = db.siteDao()
}
```

Use **DataStore (Preferences)** for key-value settings — `SharedPreferences` is not the default on this stack. Networking is **Retrofit + OkHttp** with the **kotlinx.serialization** converter (declare `@Serializable` DTOs; keep them separate from domain and Room types). Load images with **Coil 3** (`io.coil-kt.coil3`), which needs an explicit network artifact (`coil-network-okhttp`); `AsyncImage` handles loading/error state and cancels the request when it leaves composition.

```kotlin
import coil3.compose.AsyncImage
import androidx.compose.ui.layout.ContentScale

@Serializable
data class SiteDto(val id: String, val name: String, val region: String)

interface SiteApi {
    @GET("sites")
    suspend fun getSites(): List<SiteDto>
}

@Composable
fun SiteThumbnail(url: String, modifier: Modifier = Modifier) {
    AsyncImage(
        model = url,
        contentDescription = null,
        modifier = modifier,
        contentScale = ContentScale.Crop,
    )
}
```

A repository composes these, mapping DTO → entity → domain and exposing a single `Flow` the UI trusts:

```kotlin
class DefaultSiteRepository @Inject constructor(
    private val api: SiteApi,
    private val dao: SiteDao,
    @Dispatcher(IO) private val io: CoroutineDispatcher,
) : SiteRepository {

    override fun observeSites(): Flow<List<Site>> =
        dao.observeAll().map { rows -> rows.map(SiteEntity::toDomain) }

    override suspend fun refresh() = withContext(io) {
        val remote = api.getSites().map(SiteDto::toEntity)
        dao.upsertAll(remote)
    }
}
```

## Testing

Unit-test ViewModels and repositories on the JVM with JUnit4, `kotlinx-coroutines-test`, MockK for collaborators, and Turbine for Flow assertions. Set the `Main` dispatcher with a rule and inject a `StandardTestDispatcher`.

```kotlin
import app.cash.turbine.test
import kotlinx.coroutines.test.runTest
import io.mockk.every
import io.mockk.mockk

class SiteDetailViewModelTest {
    @get:Rule val mainDispatcherRule = MainDispatcherRule()

    private val repository = mockk<SiteRepository>()

    @Test
    fun `emits Ready when site loads`() = runTest {
        every { repository.observeSite("s1") } returns flowOf(fakeSite(id = "s1"))
        val vm = SiteDetailViewModel(SavedStateHandle(mapOf("siteId" to "s1")), repository)

        vm.uiState.test {
            assertEquals(SiteDetailUiState.Loading, awaitItem())
            val ready = awaitItem() as SiteDetailUiState.Ready
            assertEquals("s1", ready.site.id)
            cancelAndConsumeRemainingEvents()
        }
    }
}
```

For Compose UI tests, use `createAndroidComposeRule` (or `createComposeRule`) and drive the tree via semantics. As of Compose 1.11 the v2 testing APIs are the default, and `runComposeUiTest` now uses `StandardTestDispatcher`, so a launched coroutine is queued and recompositions do not run until the virtual clock advances — advance them with `waitForIdle()`/`advanceUntilIdle()` rather than assuming immediate execution. Prefer Robolectric for fast on-JVM UI tests. For Hilt instrumentation tests, use a `HiltTestRunner` extending `AndroidJUnitRunner` that swaps in `HiltTestApplication`, annotate tests `@HiltAndroidTest`, and inject fakes via `@TestInstallIn`. Test navigation by asserting on the back stack list directly — it is ordinary state, so no test harness is required.

## Tooling: format, lint, static analysis

Format with **ktfmt** and lint with **detekt** plus the Compose lint rules; keep Android Lint for platform correctness. ktfmt is the maintained, non-configurable formatter (fewer style debates, auto-fixable); ktlint is the familiar incumbent but combines formatting and linting and is not preferred here. Configure both through version-catalog-wired plugins.

```kotlin
// build.gradle.kts (root)
detekt {
    buildUponDefaultConfig = true
    config.setFrom(files("$rootDir/config/detekt.yml"))
    autoCorrect = true
}
dependencies {
    detektPlugins(libs.compose.rules.detekt)
}
```

```yaml
# config/detekt.yml (excerpt) — enable the Compose rule set
Compose:
  ComposableNaming:
    active: true
  ComposableParamOrder:
    active: true
  ModifierMissing:
    active: true
  ModifierNotUsedAtRoot:
    active: true
```

Commands for the coherent setup:

```bash
./gradlew ktfmtFormat                  # apply formatting
./gradlew detekt                       # static analysis + Compose rules
./gradlew lint                         # Android Lint
./gradlew testDebugUnitTest            # JVM unit tests
./gradlew connectedDebugAndroidTest    # instrumented + Compose UI tests
./gradlew assembleRelease
```

## Anti-patterns to avoid

| Wrong | Why | Right |
|---|---|---|
| Applying `org.jetbrains.kotlin.android` in a module | AGP 9.1 has built-in Kotlin and the plugin is incompatible with the new DSL | Apply only AGP + the Kotlin-versioned plugins (compose, serialization); override KGP via the root `plugins` block |
| `NavController` + `NavHost` + string routes | That is Navigation 2, a different library | Own a `List<NavKey>`; render with `NavDisplay` + `entryProvider` |
| `kapt` for Hilt/Room | KSP-only stack; Room 3 dropped kapt entirely; kapt is slower | `ksp(...)` for every annotation processor |
| `collectAsState()` in composables | Keeps collecting in the background, wastes work and shows stale data | `collectAsStateWithLifecycle()` |
| Branching layout on `Configuration.screenWidthDp` | Ignores foldable posture and multi-window; brittle | `currentWindowAdaptiveInfo()` + `WindowSizeClass` breakpoints |
| Hand-built `Row` two-pane for tablets | Reinvents adaptive logic; breaks on resize and back | `ListDetailSceneStrategy` (Nav 3) or `ListDetailPaneScaffold` |
| `remember { mutableStateListOf(...) }` for the back stack | Lost on process death | `rememberNavBackStack(startKey)` with `@Serializable` `NavKey`s |
| Non-`@Serializable` `NavKey` | `rememberNavBackStack` cannot persist/restore it | Annotate every route `@Serializable` |
| Omitting the ViewModel/saved-state decorators on `NavDisplay` | ViewModels leak across entries or aren't scoped; state lost | Add `rememberViewModelStoreNavEntryDecorator()` + `rememberSaveableStateHolderNavEntryDecorator()` |
| `Modifier` reused on several nodes / not the first optional param | Padding/click regions land on the wrong node | One `Modifier` param defaulting to `Modifier`, applied to the root |
| Emitting navigation/snackbar events inside a `StateFlow` | Replays after rotation | Consumable event state or a `Channel`/`receiveAsFlow()` |
| Raw `List<T>` in Compose state | Compiler cannot prove stability → extra recomposition | Expose `ImmutableList`/`PersistentList` |
| `SharedPreferences` for settings | Superseded here | DataStore Preferences |
| Hilt 2.59.0 with AGP 9 | Missing `dagger.hilt.internal.componenttreedeps.ComponentTreeDeps` runtime class breaks the build | Hilt 2.60.1 |

## Version & compatibility

| Component | Release line / version | Notes |
|---|---|---|
| Kotlin | 2.4.10 | K2 only; K1 frontend removed. Context parameters stable |
| AGP | 9.1.1 | Built-in Kotlin; requires Gradle 9.1+ and JDK 17; supports compileSdk 37 |
| Gradle | 9.1+ | Mandatory for AGP 9.1 |
| JDK (build) | 17 | Minimum for AGP 9 |
| KSP | 2.3.11 | Versioned independently of Kotlin since 2.3.0; works with Kotlin 2.4.x |
| compileSdk / targetSdk | 37 (Android 17) | Availability floor for target-gated behavior changes |
| minSdk | 30 | Deployment floor; higher APIs go behind checks |
| Compose BOM | 2026.08.00 | Maps core Compose modules to 1.12 |
| Material 3 Adaptive | 1.3.0 | `adaptive`, `adaptive-layout`, `adaptive-navigation`, `adaptive-navigation3` (stable) |
| material3-adaptive-navigation-suite | via Compose Material 3 | Versioned with Material 3 (BOM-managed) |
| Navigation 3 | 1.1.7 | `navigation3-runtime`, `navigation3-ui`; 1.2.0 line still beta |
| lifecycle-viewmodel-navigation3 | 2.10.0 | Nav 3 ViewModel scoping decorator |
| Hilt (Dagger) | 2.60.1 | Requires AGP 9 (since 2.59); 2.59.0 has the ComponentTreeDeps bug |
| androidx.hilt (nav-compose, viewmodel-compose, compiler) | 1.3.0 | `hiltViewModel()` moved to `hilt-lifecycle-viewmodel-compose` |
| Room | 3.0.2 (`androidx.room3`) | Coroutine-first, Kotlin codegen, KSP-only |
| DataStore | 1.1.7 | Preferences + Proto |
| Retrofit / OkHttp | 3.0.0 / 5.1.0 | kotlinx.serialization converter |
| Coil | 3.6.1 | Needs explicit `coil-network-okhttp` |
| detekt / ktfmt | 1.23.8 / 0.56 | + Compose rules 0.4.28; Android Lint retained |

- **Research date:** September 5, 2026
