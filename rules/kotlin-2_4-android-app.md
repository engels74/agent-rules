---
type: "agent_requested"
description: "Kotlin 2.4 + Jetpack Compose + Navigation 3 Android coding guidelines"
---
# Modern Android with Kotlin 2.4, Compose, Navigation 3, and Material 3 Adaptive

This stack is a single-activity, fully-Compose Android app targeting phones and tablets (minSdk 26), built on Kotlin 2.4 (K2-only), Jetpack Compose (BOM 2026.08.00), Material 3 with Material 3 Adaptive, Jetpack Navigation 3, and Hilt wired through KSP. Optimize for: unidirectional data flow with immutable state hoisted out of composables; a back stack you own as a plain observable list; and layouts that adapt to window size rather than to device type. The build uses Android Gradle Plugin 9's built-in Kotlin support, so there is no separate `org.jetbrains.kotlin.android` plugin to apply.

Agents most often write wrong-but-plausible code here by importing Navigation 2 habits (a `NavController`, string routes, `NavHost`) into Navigation 3, which has none of them; by defensively annotating everything `@Stable`/`@Immutable` under a compiler that already has strong skipping on by default; by collecting Flows with `collectAsState()` instead of `collectAsStateWithLifecycle()`; and by treating a new SDK as a reason to raise the deployment target. This document is the corrective.

## Build setup and toolchain

The build is driven by a Gradle version catalog. All exact pins live in `gradle/libs.versions.toml` and nowhere else.

```toml
# gradle/libs.versions.toml
[versions]
agp = "9.3.0"
kotlin = "2.4.10"
ksp = "2.3.10"                      # KSP2 versioning is decoupled from Kotlin since KSP 2.3.0
hilt = "2.57.1"
hiltNavigation = "1.3.0"
composeBom = "2026.08.00"          # core Compose artifacts resolve to 1.12.0
material3Adaptive = "1.3.0"
navigation3 = "1.1.6"
lifecycle = "2.11.0"
coroutines = "1.10.2"
serialization = "1.9.0"
room = "2.8.4"
datastore = "1.2.1"
coil = "3.6.1"
retrofit = "3.0.0"
okhttp = "5.1.0"
robolectric = "4.16.1"
mockk = "1.14.11"
turbine = "1.2.1"
junit = "4.13.2"
androidxTestExt = "1.3.0"
coreKtx = "1.17.0"
activityCompose = "1.11.0"

[libraries]
# Compose (versionless — governed by the BOM)
compose-bom = { module = "androidx.compose:compose-bom", version.ref = "composeBom" }
compose-ui = { module = "androidx.compose.ui:ui" }
compose-ui-graphics = { module = "androidx.compose.ui:ui-graphics" }
compose-ui-tooling = { module = "androidx.compose.ui:ui-tooling" }
compose-ui-tooling-preview = { module = "androidx.compose.ui:ui-tooling-preview" }
compose-material3 = { module = "androidx.compose.material3:material3" }
compose-material-icons = { module = "androidx.compose.material:material-icons-extended" }

# Material 3 Adaptive (not in the BOM)
material3-adaptive = { module = "androidx.compose.material3.adaptive:adaptive", version.ref = "material3Adaptive" }
material3-adaptive-layout = { module = "androidx.compose.material3.adaptive:adaptive-layout", version.ref = "material3Adaptive" }
material3-adaptive-navigation = { module = "androidx.compose.material3.adaptive:adaptive-navigation", version.ref = "material3Adaptive" }
material3-adaptive-navigation3 = { module = "androidx.compose.material3.adaptive:adaptive-navigation3", version.ref = "material3Adaptive" }
material3-adaptive-navigation-suite = { module = "androidx.compose.material3:material3-adaptive-navigation-suite" }
window-size-class = { module = "androidx.compose.material3:material3-window-size-class" }

# Navigation 3
navigation3-runtime = { module = "androidx.navigation3:navigation3-runtime", version.ref = "navigation3" }
navigation3-ui = { module = "androidx.navigation3:navigation3-ui", version.ref = "navigation3" }
lifecycle-viewmodel-navigation3 = { module = "androidx.lifecycle:lifecycle-viewmodel-navigation3", version.ref = "lifecycle" }

# Lifecycle
lifecycle-runtime-compose = { module = "androidx.lifecycle:lifecycle-runtime-compose", version.ref = "lifecycle" }
lifecycle-viewmodel-compose = { module = "androidx.lifecycle:lifecycle-viewmodel-compose", version.ref = "lifecycle" }

# Hilt
hilt-android = { module = "com.google.dagger:hilt-android", version.ref = "hilt" }
hilt-compiler = { module = "com.google.dagger:hilt-compiler", version.ref = "hilt" }
hilt-navigation-compose = { module = "androidx.hilt:hilt-navigation-compose", version.ref = "hiltNavigation" }

# Data + network + images
room-runtime = { module = "androidx.room:room-runtime", version.ref = "room" }
room-ktx = { module = "androidx.room:room-ktx", version.ref = "room" }
room-compiler = { module = "androidx.room:room-compiler", version.ref = "room" }
datastore-preferences = { module = "androidx.datastore:datastore-preferences", version.ref = "datastore" }
retrofit = { module = "com.squareup.retrofit2:retrofit", version.ref = "retrofit" }
retrofit-serialization = { module = "com.squareup.retrofit2:converter-kotlinx-serialization", version.ref = "retrofit" }
okhttp = { module = "com.squareup.okhttp3:okhttp", version.ref = "okhttp" }
okhttp-logging = { module = "com.squareup.okhttp3:logging-interceptor", version.ref = "okhttp" }
coil-compose = { module = "io.coil-kt.coil3:coil-compose", version.ref = "coil" }
coil-network-okhttp = { module = "io.coil-kt.coil3:coil-network-okhttp", version.ref = "coil" }
serialization-json = { module = "org.jetbrains.kotlinx:kotlinx-serialization-json", version.ref = "serialization" }
coroutines-android = { module = "org.jetbrains.kotlinx:kotlinx-coroutines-android", version.ref = "coroutines" }

androidx-core-ktx = { module = "androidx.core:core-ktx", version.ref = "coreKtx" }
androidx-activity-compose = { module = "androidx.activity:activity-compose", version.ref = "activityCompose" }

# Test
junit = { module = "junit:junit", version.ref = "junit" }
androidx-junit = { module = "androidx.test.ext:junit", version.ref = "androidxTestExt" }
compose-ui-test-junit4 = { module = "androidx.compose.ui:ui-test-junit4" }
compose-ui-test-manifest = { module = "androidx.compose.ui:ui-test-manifest" }
robolectric = { module = "org.robolectric:robolectric", version.ref = "robolectric" }
mockk = { module = "io.mockk:mockk", version.ref = "mockk" }
turbine = { module = "app.cash.turbine:turbine", version.ref = "turbine" }
coroutines-test = { module = "org.jetbrains.kotlinx:kotlinx-coroutines-test", version.ref = "coroutines" }

[plugins]
android-application = { id = "com.android.application", version.ref = "agp" }
kotlin-compose = { id = "org.jetbrains.kotlin.plugin.compose", version.ref = "kotlin" }
kotlin-serialization = { id = "org.jetbrains.kotlin.plugin.serialization", version.ref = "kotlin" }
ksp = { id = "com.google.devtools.ksp", version.ref = "ksp" }
hilt = { id = "com.google.dagger.hilt.android", version.ref = "hilt" }
```

Under AGP 9 the Kotlin toolchain is built into the Android plugin, so you no longer apply `org.jetbrains.kotlin.android`. You still apply the Compose compiler plugin (`org.jetbrains.kotlin.plugin.compose`), serialization, KSP, and Hilt.

```kotlin
// app/build.gradle.kts
plugins {
    alias(libs.plugins.android.application)
    alias(libs.plugins.kotlin.compose)
    alias(libs.plugins.kotlin.serialization)
    alias(libs.plugins.ksp)
    alias(libs.plugins.hilt)
}

android {
    namespace = "com.example.catalog"
    compileSdk = 36                       // Android 16 SDK; latest stable platform

    defaultConfig {
        applicationId = "com.example.catalog"
        minSdk = 26                        // constraint — build behind availability checks, do not raise
        targetSdk = 36                     // Play: new apps/updates must target API 36 from Aug 31, 2026
        versionCode = 1
        versionName = "1.0"
        testInstrumentationRunner = "androidx.test.runner.AndroidJUnitRunner"
    }

    buildFeatures { compose = true }

    compileOptions {
        sourceCompatibility = JavaVersion.VERSION_17
        targetCompatibility = JavaVersion.VERSION_17
    }
    kotlin { jvmToolchain(17) }            // AGP 9 requires JDK 17

    buildTypes {
        release {
            isMinifyEnabled = true
            isShrinkResources = true
            proguardFiles(getDefaultProguardFile("proguard-android-optimize.txt"), "proguard-rules.pro")
        }
    }
}

composeCompiler {
    // Emit stability/skippability reports on demand: ./gradlew assembleRelease -Pcompose.reports
    if (project.hasProperty("compose.reports")) {
        reportsDestination = layout.buildDirectory.dir("compose_compiler")
        metricsDestination = layout.buildDirectory.dir("compose_compiler")
    }
}

dependencies {
    implementation(platform(libs.compose.bom))
    androidTestImplementation(platform(libs.compose.bom))

    implementation(libs.compose.ui)
    implementation(libs.compose.ui.graphics)
    implementation(libs.compose.ui.tooling.preview)
    debugImplementation(libs.compose.ui.tooling)
    implementation(libs.compose.material3)
    implementation(libs.compose.material.icons)
    implementation(libs.window.size.class)

    implementation(libs.material3.adaptive)
    implementation(libs.material3.adaptive.layout)
    implementation(libs.material3.adaptive.navigation)
    implementation(libs.material3.adaptive.navigation3)
    implementation(libs.material3.adaptive.navigation.suite)

    implementation(libs.navigation3.runtime)
    implementation(libs.navigation3.ui)
    implementation(libs.lifecycle.viewmodel.navigation3)
    implementation(libs.lifecycle.runtime.compose)
    implementation(libs.lifecycle.viewmodel.compose)

    implementation(libs.hilt.android)
    ksp(libs.hilt.compiler)
    implementation(libs.hilt.navigation.compose)

    implementation(libs.room.runtime)
    implementation(libs.room.ktx)
    ksp(libs.room.compiler)
    implementation(libs.datastore.preferences)

    implementation(libs.retrofit)
    implementation(libs.retrofit.serialization)
    implementation(libs.okhttp)
    implementation(libs.okhttp.logging)
    implementation(libs.serialization.json)
    implementation(libs.coroutines.android)

    implementation(libs.coil.compose)
    implementation(libs.coil.network.okhttp)

    implementation(libs.androidx.core.ktx)
    implementation(libs.androidx.activity.compose)

    testImplementation(libs.junit)
    testImplementation(libs.robolectric)
    testImplementation(libs.mockk)
    testImplementation(libs.turbine)
    testImplementation(libs.coroutines.test)
    testImplementation(libs.compose.ui.test.junit4)
    debugImplementation(libs.compose.ui.test.manifest)
    androidTestImplementation(libs.androidx.junit)
    androidTestImplementation(libs.compose.ui.test.junit4)
}
```

The Gradle wrapper must be 9.6.0 or newer for AGP 9.3; AGP 9.x requires Gradle 9.1.0 as a hard floor that cannot be bypassed when coming from AGP 8.x. `compileSdk = 36` gives you Android 16 APIs; `targetSdk = 36` opts into Android 16 runtime behavior and satisfies the Google Play requirement that, from August 31, 2026, all new apps and updates target API level 36 or higher (a one-time extension is available until November 1, 2026). `minSdk = 26` is a constraint to build on, not a stale value: APIs newer than 26 belong behind `Build.VERSION.SDK_INT` checks, not behind a raised floor.

**R8 optimization DSL (AGP 9).** The release block above uses the classic `isMinifyEnabled`/`isShrinkResources` flags, which remain supported. The updated `optimization` DSL is equivalent and folds resource shrinking into one switch; either is correct. Keep app-specific keep rules in `proguard-rules.pro`. `kotlinx.serialization` classes are kept automatically by the plugin's bundled consumer rules; Room and Hilt likewise ship their own.

## Kotlin 2.4 language baseline

Kotlin 2.4 is **K2-only**: the K1 frontend is gone and `-language-version 1.9` is rejected outright, so the language floor is 2.0. Write idiomatic K2 Kotlin and do not carry `languageVersion`/`apiVersion` pins from older modules.

Two language features stabilized in 2.4 are worth adopting where they fit. **Context parameters** are stable in 2.4 (with the exception of context arguments and callable references, which remain experimental), and let a function declare an ambient dependency instead of threading it through every signature — useful for cross-cutting concerns like a logger or clock, not as a replacement for constructor injection:

```kotlin
import kotlin.time.Clock

context(clock: Clock)
fun Order.isExpired(ttlMillis: Long): Boolean =
    clock.now().toEpochMilliseconds() - createdAtMillis > ttlMillis
```

**Explicit backing fields** (stable since 2.4) remove the private-backing-property boilerplate for the "mutable inside, read-only outside" pattern — though in ViewModels the `MutableStateFlow`/`StateFlow` idiom below is still the norm:

```kotlin
val items: List<Item>
    field = mutableListOf()   // explicit backing field, exposed read-only
```

Use `kotlin.time.Clock` and `kotlin.time.Instant` (stable) rather than `System.currentTimeMillis()` for testable time. The standard `kotlin.uuid.Uuid` type is stable for identifiers, but minting random UUIDs from the standard library is still experimental — prefer `java.util.UUID.randomUUID()` on Android for generation.

## State, Compose, and stability

State flows down, events flow up. Composables are functions of their arguments; hoist state to a ViewModel and pass immutable snapshots plus lambdas.

```kotlin
import androidx.compose.runtime.Composable
import androidx.compose.runtime.getValue
import androidx.lifecycle.compose.collectAsStateWithLifecycle
import androidx.hilt.navigation.compose.hiltViewModel

@Composable
fun ProductListRoute(
    onProductClick: (String) -> Unit,
    viewModel: ProductListViewModel = hiltViewModel(),
) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()
    ProductListScreen(state = uiState, onProductClick = onProductClick)
}
```

Always collect Flows with `collectAsStateWithLifecycle()` (from `lifecycle-runtime-compose`), not `collectAsState()`. The former stops collecting when the app is not at least `STARTED`, which prevents pointless work and upstream calls while the UI is in the background; `collectAsState()` keeps collecting and is the wrong default on Android.

**Stability and strong skipping.** Strong skipping mode is enabled by default in Kotlin 2.0.20 and later. Under it, restartable composables are skippable regardless of parameter stability: as the Compose team documents it, "Composables with unstable parameters can be skipped. Unstable parameters are compared for equality via instance equality (`===`). Stable parameters continue to be compared for equality with `Object.equals()`. All lambdas in composable functions are automatically remembered." The practical consequences:

- Do **not** reflexively annotate every model `@Stable`/`@Immutable`. Enums and sealed classes are already inferred stable; annotating them is noise. Reach for `@Immutable` only when a type has an expensive `equals()` or when you pass freshly-mapped instances (e.g. new DTOs on every API response) and want the compiler to trust a stability contract.
- Lambda auto-memoization applies **inside** `@Composable` functions but **not** in `LazyListScope` (the `items { }` builder is not composable). Wrap callbacks captured in `item`/`items` blocks with `remember` keyed on their captures.
- Model UI state as a single immutable `data class` and expose it as `StateFlow<UiState>`. Prefer `kotlinx.collections.immutable` (`persistentListOf`) or accept that `List` is treated as stable at runtime under strong skipping.

```kotlin
data class ProductListUiState(
    val products: List<ProductUi> = emptyList(),
    val isLoading: Boolean = true,
    val errorMessage: String? = null,
)
```

Read the compiler's `*_composables.txt`/`*_classes.txt` reports (enabled by the `composeCompiler` block above) before hand-tuning stability, then confirm with recomposition counts in the profiler — the metrics say what the compiler thinks, the profiler says what matters.

## Edge-to-edge, theming, and the single Activity

There is one `Activity`, annotated `@AndroidEntryPoint`, calling `enableEdgeToEdge()` before `setContent`. Edge-to-edge is drawn under the system bars on every supported release; consume insets with `Scaffold` and `WindowInsets` rather than hardcoding padding.

```kotlin
import androidx.activity.ComponentActivity
import androidx.activity.compose.setContent
import androidx.activity.enableEdgeToEdge
import dagger.hilt.android.AndroidEntryPoint

@AndroidEntryPoint
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        enableEdgeToEdge()
        super.onCreate(savedInstanceState)
        setContent { CatalogTheme { CatalogApp() } }
    }
}
```

Add `android:enableOnBackInvokedCallback="true"` to the `<application>` element in `AndroidManifest.xml` to opt into predictive back. Navigation 3's `NavDisplay` handles the predictive-back preview animation for you when the back stack has entries; `BackHandler` still intercepts back for in-screen state. Use Material 3 dynamic color where available and a fixed color scheme below API 31:

```kotlin
@Composable
fun CatalogTheme(darkTheme: Boolean = isSystemInDarkTheme(), content: @Composable () -> Unit) {
    val context = LocalContext.current
    val colorScheme = when {
        Build.VERSION.SDK_INT >= Build.VERSION_CODES.S ->
            if (darkTheme) dynamicDarkColorScheme(context) else dynamicLightColorScheme(context)
        darkTheme -> darkColorScheme()
        else -> lightColorScheme()
    }
    MaterialTheme(colorScheme = colorScheme, typography = Typography, content = content)
}
```

## App architecture: Navigation 3

Navigation 3 has no `NavController`, no string routes, and no `NavHost`. The back stack **is** an observable `SnapshotStateList` of keys that you own. Navigating is list mutation: `add` to go forward, `removeLastOrNull` to go back.

Define destinations as `@Serializable` types implementing `NavKey`. Serialization is what lets `rememberNavBackStack` restore the stack across process death, so keep keys small and serializable — carry IDs, not objects.

```kotlin
import androidx.navigation3.runtime.NavKey
import kotlinx.serialization.Serializable

@Serializable data object ProductList : NavKey
@Serializable data class ProductDetail(val productId: String) : NavKey
@Serializable data object Settings : NavKey
```

`NavDisplay` observes the back stack and renders the top entry. `entryProvider` maps each key type to a `NavEntry`; the `entry<T> { }` DSL is type-safe and gives you the key instance. Two decorators are essential in production: `rememberSaveableStateHolderNavEntryDecorator()` preserves per-entry `rememberSaveable` state, and `rememberViewModelStoreNavEntryDecorator()` gives each entry its own `ViewModelStore` so ViewModels are scoped to the destination instance and cleared when it leaves the stack.

```kotlin
import androidx.compose.runtime.Composable
import androidx.navigation3.runtime.entryProvider
import androidx.navigation3.runtime.rememberNavBackStack
import androidx.navigation3.runtime.rememberSaveableStateHolderNavEntryDecorator
import androidx.navigation3.ui.NavDisplay
import androidx.lifecycle.viewmodel.navigation3.rememberViewModelStoreNavEntryDecorator

@Composable
fun CatalogApp() {
    val backStack = rememberNavBackStack<NavKey>(ProductList)

    NavDisplay(
        backStack = backStack,
        onBack = { count -> repeat(count) { backStack.removeLastOrNull() } },
        entryDecorators = listOf(
            rememberSaveableStateHolderNavEntryDecorator(),
            rememberViewModelStoreNavEntryDecorator(),
        ),
        entryProvider = entryProvider {
            entry<ProductList> {
                ProductListRoute(
                    onProductClick = { id -> backStack.add(ProductDetail(id)) },
                )
            }
            entry<ProductDetail> { key ->
                ProductDetailRoute(
                    productId = key.productId,
                    onBack = { backStack.removeLastOrNull() },
                )
            }
            entry<Settings> { SettingsRoute() }
        },
    )
}
```

The `onBack` lambda receives a count (predictive back may request removing several entries at once); honor it with `repeat`. Do not reintroduce a `NavController` wrapper — the whole point is that the stack is plain state you can inspect, log, and test.

**Multiple back stacks (bottom tabs).** Keep one `rememberNavBackStack` per tab and swap which one `NavDisplay` renders based on the selected tab; this preserves each tab's history independently.

## Adaptive layouts for phones, tablets, and foldables

Adapt to **window size**, never to a hardcoded "isTablet" boolean. Drive decisions from `WindowSizeClass`/`currentWindowAdaptiveInfo()`. For the canonical list-detail pattern, integrate Material 3 Adaptive with Navigation 3 through `ListDetailSceneStrategy`, which decides one, two, or three panes from the available width.

The scene strategy is passed to `NavDisplay` via `sceneStrategy`; entries opt into a pane role through metadata. These adaptive APIs require `@OptIn(ExperimentalMaterial3AdaptiveApi::class)`.

```kotlin
import androidx.compose.material3.adaptive.ExperimentalMaterial3AdaptiveApi
import androidx.compose.material3.adaptive.navigation3.ListDetailSceneStrategy
import androidx.compose.material3.adaptive.navigation3.rememberListDetailSceneStrategy
import androidx.compose.material3.Text
import androidx.navigation3.runtime.entryProvider
import androidx.navigation3.runtime.rememberNavBackStack
import androidx.navigation3.ui.NavDisplay

@OptIn(ExperimentalMaterial3AdaptiveApi::class)
@Composable
fun ConversationsApp() {
    val backStack = rememberNavBackStack<NavKey>(ConversationList)
    val listDetailStrategy = rememberListDetailSceneStrategy<NavKey>()

    NavDisplay(
        backStack = backStack,
        sceneStrategy = listDetailStrategy,
        onBack = { count -> repeat(count) { backStack.removeLastOrNull() } },
        entryDecorators = listOf(
            rememberSaveableStateHolderNavEntryDecorator(),
            rememberViewModelStoreNavEntryDecorator(),
        ),
        entryProvider = entryProvider {
            entry<ConversationList>(
                metadata = ListDetailSceneStrategy.listPane(
                    detailPlaceholder = { Text("Select a conversation") },
                ),
            ) {
                ConversationListRoute(
                    onConversationClick = { id -> backStack.add(ConversationDetail(id)) },
                )
            }
            entry<ConversationDetail>(
                metadata = ListDetailSceneStrategy.detailPane(),
            ) { key -> ConversationDetailRoute(conversationId = key.conversationId) }
        },
    )
}
```

On a phone the strategy shows the list, then the detail full-screen; on a tablet or unfolded foldable it shows them side by side, with `detailPlaceholder` filling the empty detail pane when nothing is selected. Because the list and detail share one back stack, back on a two-pane layout can collapse the selection rather than exiting — the `onBack` count handles this.

For top-level destination switching that itself adapts (bottom bar on compact width, navigation rail on medium, drawer on expanded), use `NavigationSuiteScaffold` from `material3-adaptive-navigation-suite`:

```kotlin
import androidx.compose.material3.adaptive.navigationsuite.NavigationSuiteScaffold

@Composable
fun TopLevelScaffold(selected: TopDest, onSelect: (TopDest) -> Unit, content: @Composable () -> Unit) {
    NavigationSuiteScaffold(
        navigationSuiteItems = {
            TopDest.entries.forEach { dest ->
                item(
                    selected = dest == selected,
                    onClick = { onSelect(dest) },
                    icon = { Icon(dest.icon, contentDescription = null) },
                    label = { Text(dest.label) },
                )
            }
        },
        content = content,
    )
}
```

## Dependency injection with Hilt

Hilt runs through KSP (never kapt). The `Application` is annotated `@HiltAndroidApp`; the single `Activity` is `@AndroidEntryPoint`; ViewModels are `@HiltViewModel` with constructor injection.

```kotlin
@HiltAndroidApp
class CatalogApplication : Application()
```

Provide app-scoped singletons in `@Module @InstallIn(SingletonComponent::class)` objects. Bind interfaces to implementations with `@Binds`; provide types you do not own (Retrofit, Room, OkHttp) with `@Provides`.

```kotlin
import dagger.Module
import dagger.Provides
import dagger.hilt.InstallIn
import dagger.hilt.components.SingletonComponent
import kotlinx.serialization.json.Json
import okhttp3.MediaType.Companion.toMediaType
import okhttp3.OkHttpClient
import okhttp3.logging.HttpLoggingInterceptor
import retrofit2.Retrofit
import retrofit2.converter.kotlinx.serialization.asConverterFactory
import javax.inject.Singleton

@Module
@InstallIn(SingletonComponent::class)
object NetworkModule {

    @Provides
    @Singleton
    fun provideJson(): Json = Json {
        ignoreUnknownKeys = true
        explicitNulls = false
    }

    @Provides
    @Singleton
    fun provideOkHttp(): OkHttpClient = OkHttpClient.Builder()
        .addInterceptor(HttpLoggingInterceptor().apply { level = HttpLoggingInterceptor.Level.BASIC })
        .build()

    @Provides
    @Singleton
    fun provideRetrofit(client: OkHttpClient, json: Json): Retrofit = Retrofit.Builder()
        .baseUrl("https://api.example.com/")
        .client(client)
        .addConverterFactory(json.asConverterFactory("application/json".toMediaType()))
        .build()

    @Provides
    @Singleton
    fun provideProductApi(retrofit: Retrofit): ProductApi = retrofit.create(ProductApi::class.java)
}
```

```kotlin
import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import dagger.hilt.android.lifecycle.HiltViewModel
import kotlinx.coroutines.flow.*
import javax.inject.Inject

@HiltViewModel
class ProductListViewModel @Inject constructor(
    private val repository: ProductRepository,
) : ViewModel() {

    val uiState: StateFlow<ProductListUiState> = repository.products
        .map { products -> ProductListUiState(products = products, isLoading = false) }
        .stateIn(
            scope = viewModelScope,
            started = SharingStarted.WhileSubscribed(5_000),
            initialValue = ProductListUiState(),
        )
}
```

In a composable inside a `NavDisplay` entry, obtain the ViewModel with `hiltViewModel()`. Combined with `rememberViewModelStoreNavEntryDecorator()`, each entry gets its own scope, so a `ProductDetail` ViewModel is created when you navigate in and cleared when the entry is popped. `hiltViewModel()` lives in `androidx.hilt:hilt-navigation-compose`; if you want it without a transitive Navigation dependency, `androidx.hilt:hilt-lifecycle-viewmodel-compose` exposes the same API. To pass navigation arguments into a ViewModel, read them from `SavedStateHandle` — but with Navigation 3 the simplest correct path is to pass the key's `productId` directly to the route composable and into an assisted-inject factory or the repository call, since the key already carries type-safe arguments.

## Data layer

**Room** is the local database; use the KSP compiler and suspend/`Flow` DAOs. Return `Flow` for observable reads and `suspend` for one-shot writes; Room moves both off the main thread.

```kotlin
import androidx.room.*
import kotlinx.coroutines.flow.Flow

@Entity(tableName = "products")
data class ProductEntity(
    @PrimaryKey val id: String,
    val name: String,
    val priceCents: Long,
)

@Dao
interface ProductDao {
    @Query("SELECT * FROM products ORDER BY name")
    fun observeAll(): Flow<List<ProductEntity>>

    @Upsert
    suspend fun upsertAll(products: List<ProductEntity>)

    @Query("DELETE FROM products")
    suspend fun clear()
}

@Database(entities = [ProductEntity::class], version = 1, exportSchema = true)
abstract class CatalogDatabase : RoomDatabase() {
    abstract fun productDao(): ProductDao
}
```

**DataStore Preferences** replaces `SharedPreferences` for key-value settings; it is fully async and exposes a `Flow`. Never call the deprecated `SharedPreferences` APIs for new code.

```kotlin
import android.content.Context
import androidx.datastore.preferences.core.*
import androidx.datastore.preferences.preferencesDataStore
import dagger.hilt.android.qualifiers.ApplicationContext
import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.flow.map
import javax.inject.Inject

private val Context.dataStore by preferencesDataStore(name = "settings")
private val DARK_THEME = booleanPreferencesKey("dark_theme")

class SettingsRepository @Inject constructor(@ApplicationContext private val context: Context) {
    val darkTheme: Flow<Boolean> = context.dataStore.data.map { it[DARK_THEME] ?: false }
    suspend fun setDarkTheme(enabled: Boolean) {
        context.dataStore.edit { it[DARK_THEME] = enabled }
    }
}
```

**Networking.** The modern default is Retrofit 3 with the first-party `kotlinx.serialization` converter over OkHttp 5, using `@Serializable` DTOs. Retrofit 3 depends on the Kotlin-based OkHttp and offers idiomatic coroutine `suspend` functions. Gson and Moshi are legacy choices here — prefer `kotlinx.serialization`, which you already need for `NavKey`. Ktor is the right pick only for Kotlin Multiplatform shared code; for an Android-only app Retrofit is leaner.

```kotlin
import kotlinx.serialization.SerialName
import kotlinx.serialization.Serializable
import retrofit2.http.GET
import retrofit2.http.Path

@Serializable
data class ProductDto(val id: String, val name: String, @SerialName("price_cents") val priceCents: Long)

interface ProductApi {
    @GET("products")
    suspend fun getProducts(): List<ProductDto>

    @GET("products/{id}")
    suspend fun getProduct(@Path("id") id: String): ProductDto
}
```

A repository owns the offline-first merge: expose the Room `Flow` as the source of truth and refresh from the network into the database.

```kotlin
class ProductRepository @Inject constructor(
    private val api: ProductApi,
    private val dao: ProductDao,
) {
    val products: Flow<List<ProductUi>> =
        dao.observeAll().map { entities -> entities.map(ProductEntity::toUi) }

    suspend fun refresh() {
        val remote = api.getProducts().map(ProductDto::toEntity)
        dao.upsertAll(remote)
    }
}
```

**Images.** Coil 3 (`io.coil-kt.coil3`) is the default async image loader for Compose. Add an explicit network backend — Coil 3 no longer bundles one — here `coil-network-okhttp`, which reuses your OkHttp stack.

```kotlin
import coil3.compose.AsyncImage

AsyncImage(
    model = product.imageUrl,
    contentDescription = product.name,
    modifier = Modifier.size(64.dp),
)
```

## Concurrency

All async work runs on coroutines and `Flow`. Business logic lives in the ViewModel and is launched in `viewModelScope`, which cancels automatically when the ViewModel is cleared. Never block; never leak a scope.

- Expose observable state as `StateFlow` produced with `stateIn(..., SharingStarted.WhileSubscribed(5_000), initial)`. The 5-second stop timeout keeps the upstream alive across configuration changes and short backgrounding without restarting collection.
- Do one-shot work (a pull-to-refresh, a button tap) in a `viewModelScope.launch { }` that calls a `suspend` repository function; surface failures into the `UiState`, not with uncaught exceptions.
- Inject a dispatcher for CPU-bound or blocking work and switch with `withContext(ioDispatcher)`; do not hardcode `Dispatchers.IO`, so tests can substitute a test dispatcher.
- Respect cancellation: suspend functions must be cooperative, and structured concurrency means a cancelled scope cancels its children. Do not wrap coroutines in `try/catch (e: Exception)` that swallows `CancellationException` — rethrow it.

```kotlin
fun refresh() {
    viewModelScope.launch {
        _uiState.update { it.copy(isLoading = true, errorMessage = null) }
        val result = runCatching { repository.refresh() }
        _uiState.update {
            it.copy(
                isLoading = false,
                errorMessage = result.exceptionOrNull()?.let { e -> "Couldn't refresh: ${e.message}" },
            )
        }
    }
}
```

`runCatching` here is safe because `refresh()` performs suspending I/O whose only failures are network/serialization errors; if you call it around code that itself launches children, prefer catching specific exceptions so cancellation propagates.

## Testing

Favor fast JVM tests. Unit-test ViewModels and repositories with JUnit4, `kotlinx-coroutines-test`, Turbine for Flow assertions, and MockK for doubles. Run Compose UI tests on the JVM with Robolectric to avoid an emulator; reserve on-device instrumented tests for genuinely device-dependent behavior (edge-to-edge, real gestures).

Compose's **v2 testing framework is the default** (BOM 2026.08.00): `createComposeRule()` now uses `StandardTestDispatcher`, so coroutines launched in a test are queued and run when the virtual clock advances rather than executing eagerly. Advance the clock (or rely on `waitForIdle`/`waitUntil`) instead of assuming immediate execution.

```kotlin
import app.cash.turbine.test
import io.mockk.coEvery
import io.mockk.mockk
import kotlinx.coroutines.flow.flowOf
import kotlinx.coroutines.test.runTest
import org.junit.Assert.assertEquals
import org.junit.Test

class ProductListViewModelTest {

    @Test
    fun `emits products after load`() = runTest {
        val repository = mockk<ProductRepository>()
        coEvery { repository.products } returns flowOf(listOf(ProductUi("1", "Widget", 199)))
        val viewModel = ProductListViewModel(repository)

        viewModel.uiState.test {
            assertEquals(true, awaitItem().isLoading)          // initial value
            val loaded = awaitItem()
            assertEquals(1, loaded.products.size)
            assertEquals(false, loaded.isLoading)
            cancelAndIgnoreRemainingEvents()
        }
    }
}
```

For a Robolectric-backed Compose UI test, enable Android resources in `testOptions { unitTests { isIncludeAndroidResources = true } }` and run with the Robolectric runner:

```kotlin
import androidx.compose.ui.test.assertIsDisplayed
import androidx.compose.ui.test.junit4.createComposeRule
import androidx.compose.ui.test.onNodeWithText
import org.junit.Rule
import org.junit.Test
import org.junit.runner.RunWith
import org.robolectric.RobolectricTestRunner
import org.robolectric.annotation.Config

@RunWith(RobolectricTestRunner::class)
@Config(sdk = [34])
class ProductListScreenTest {
    @get:Rule val composeRule = createComposeRule()

    @Test
    fun showsProductName() {
        composeRule.setContent {
            ProductListScreen(
                state = ProductListUiState(
                    products = listOf(ProductUi("1", "Widget", 199)),
                    isLoading = false,
                ),
                onProductClick = {},
            )
        }
        composeRule.onNodeWithText("Widget").assertIsDisplayed()
    }
}
```

## Tooling: format, lint, static analysis, performance

- **Format & style:** Spotless with the `ktlint` engine. One command formats the whole repo, honors `.editorconfig`, and fits CI. Configure ktlint to allow PascalCase composable names.
- **Static analysis:** detekt for complexity and design smells (it parses the Kotlin AST, so it understands coroutines and sealed types), complemented by the Compose-specific detekt/ktlint rule set for idiomatic Compose.
- **Android Lint:** run `./gradlew lint`; keep it enabled and treat new warnings as errors in CI (`lint { warningsAsErrors = true }`) where signal is high.
- **Baseline Profiles:** generate one with the Macrobenchmark module and ship it to improve cold-start and scroll jank; regenerate when hot paths change.

```kotlin
// build.gradle.kts (root) — Spotless
spotless {
    kotlin {
        target("**/*.kt")
        targetExclude("**/build/**")
        ktlint().editorConfigOverride(
            mapOf("ktlint_function_naming_ignore_when_annotated_with" to "Composable,Test"),
        )
    }
    kotlinGradle { target("**/*.gradle.kts"); ktlint() }
}
```

Commands that form the coherent setup:

```bash
./gradlew spotlessApply          # format
./gradlew spotlessCheck detekt   # verify style + static analysis
./gradlew lint                   # Android Lint
./gradlew testDebugUnitTest      # JVM unit + Robolectric UI tests
./gradlew connectedDebugAndroidTest   # on-device instrumented tests
./gradlew assembleRelease -Pcompose.reports   # emit Compose stability metrics
```

## Anti-patterns to avoid

| Wrong | Why | Right |
|---|---|---|
| Using a `NavController`, `NavHost`, or string routes | Navigation 3 has none of these; the back stack is an observable list you own | `rememberNavBackStack`, `NavDisplay`, `entryProvider`, `@Serializable` `NavKey` types |
| `collectAsState()` in a composable | Keeps collecting while backgrounded, doing needless work | `collectAsStateWithLifecycle()` from `lifecycle-runtime-compose` |
| Annotating every model `@Stable`/`@Immutable` | Strong skipping is on by default; most annotations are dead weight | Annotate only for expensive `equals()` or fresh-instance params; measure with compiler reports |
| `NavDisplay` without `rememberViewModelStoreNavEntryDecorator()` | ViewModels aren't scoped per entry and persist/leak across screens | Add the ViewModel-store and saveable-state decorators |
| Raising `minSdk` to use a newer API | A new SDK doesn't raise the deployment target; the floor is a constraint | Guard with `Build.VERSION.SDK_INT >= …`; keep `minSdk = 26` |
| kapt for Hilt/Room | kapt is legacy and slow on K2 | KSP (`ksp(...)`) for all annotation processors |
| Applying `org.jetbrains.kotlin.android` alongside AGP 9 | AGP 9 has built-in Kotlin support; the plugin conflicts | Apply only the Compose, serialization, KSP, and Hilt plugins |
| Branching layout on an "isTablet" flag | Breaks on foldables, split-screen, and resizable windows | Drive layout from `WindowSizeClass` / `ListDetailSceneStrategy` |
| `NavKey` holding non-serializable objects | Breaks `rememberNavBackStack` restoration after process death | Carry IDs; make every key `@Serializable` |
| Swallowing `CancellationException` in a broad `catch` | Breaks structured-concurrency cancellation | Use `runCatching` around leaf business calls and rethrow cancellation, or catch specific exceptions |
| `SharedPreferences` for new settings | Synchronous, main-thread-prone, superseded | DataStore Preferences with a `Flow` |
| Gson/Moshi for JSON | For a different era; you already need kotlinx.serialization for `NavKey` | `kotlinx.serialization` with the Retrofit first-party converter |

## Version & compatibility

| Component | Release line / version | Notes |
|---|---|---|
| Kotlin | 2.4 (2.4.10) | K2-only; `-language-version 1.9` rejected |
| Android Gradle Plugin | 9.3 | Built-in Kotlin support; supports up to API level 37 |
| Gradle | 9.6 (min 9.1.0) | 9.1.0 is a hard floor for AGP 9.x |
| JDK | 17 | Required by AGP 9 |
| compileSdk / targetSdk | 36 (Android 16) | Play requires targetSdk 36 for new apps/updates from Aug 31, 2026 |
| minSdk | 26 | Constraint; newer APIs behind `SDK_INT` checks |
| KSP | 2.3.10 | KSP2; version decoupled from Kotlin since KSP 2.3.0 |
| Compose BOM | 2026.08.00 | Core Compose artifacts resolve to 1.12; v2 test APIs default |
| Material 3 | via BOM (1.4 line) | 1.5.0 is alpha — do not use |
| Material 3 Adaptive | 1.3.0 | `adaptive`, `-layout`, `-navigation`, `-navigation3`; APIs need `@OptIn(ExperimentalMaterial3AdaptiveApi::class)` |
| Navigation 3 | 1.1 (1.1.6) | 1.2.0 is beta — do not use |
| lifecycle (incl. `-viewmodel-navigation3`, `-runtime-compose`) | 2.11 | Provides `collectAsStateWithLifecycle` |
| Hilt (Dagger) | 2.57.1 | KSP only |
| androidx.hilt (navigation-compose / lifecycle-viewmodel-compose) | 1.3.0 | 1.4.0 is alpha |
| Room | 2.8 (2.8.4) | KSP compiler |
| DataStore | 1.2 (1.2.1) | Preferences + Proto |
| kotlinx.coroutines | 1.10 (1.10.2) | |
| kotlinx.serialization | 1.9 (1.9.0) | JSON; also used for `NavKey` |
| Coil | 3 (3.6.1) | Add `coil-network-okhttp` explicitly |
| Retrofit / OkHttp | 3.0.0 / 5.1.0 | First-party kotlinx.serialization converter |
| Robolectric / MockK / Turbine | 4.16.1 / 1.14.11 / 1.2.1 | |

- **Research date:** 2026-09-05
