# Templates

Worked example: a **Create group** flow — `Home` (entry) → `GroupName` → `HouseSelection` → `Review` → `Result`.
Replace `CreateGroup` / `createGroup` / `CREATE_GROUP` with your flow's name. The ViewModels assume the team's
MVI base class (`BaseViewModel<Event, State, Effect>` with `setState`, `setEffect`, `viewState`,
`loadInitData()`); adapt the calls if yours differ.

## 0. Which form to write

Read the project's Gradle files and one existing Koin module first, then use the matching definitions:

| The project has | Definitions |
|---|---|
| Koin Compiler Plugin (`io.insert-koin.compiler.plugin`) and DSL modules | §3a |
| No Compiler Plugin — classic or constructor DSL | §3b |
| Koin Annotations (`@Singleton`, `@KoinViewModel`, `@Module`) | §3c |

Everything else on this page (§1, §2, §4–§10, the tests) is the same in every style; Annotations only add the
annotations shown in §3c.

## 1. FlowData — `feature/creategroup/domain/model/CreateGroupFlowData.kt`

```kotlin
/** Everything the Create group flow carries from one screen to the next. Lives only while the flow is open. */
data class CreateGroupFlowData(
    var groupName: String = "",
    var selectedHouse: HouseItem? = null,
    var pilgrimIds: List<String> = emptyList(),
    var createdGroup: CreatedGroup? = null,   // filled by the last API call, read by the Result screen
)
```

## 2. Scope type and owner — `feature/creategroup/di/CreateGroupFlow.kt`

```kotlin
import org.koin.core.scope.Scope

/** Scope type of the Create group flow. A marker: never instantiated, never injected. */
class CreateGroupFlowScope

private const val CREATE_GROUP_SCOPE_ID = "createGroupFlow"

/**
 * The one owner of the Create group flow scope — the only class that opens, replaces and closes it.
 * ViewModels receive it through their constructor and never touch Koin.
 */
class CreateGroupFlow(private val rootScope: Scope) {

    private val koin get() = rootScope.getKoin()

    /** True while a flow is open. */
    val isOpen: Boolean get() = koin.getScopeOrNull(CREATE_GROUP_SCOPE_ID) != null

    /** Entry point only. Drops whatever an abandoned flow left behind and returns empty data to fill. */
    fun start(): CreateGroupFlowData {
        close()
        return koin.createScope<CreateGroupFlowScope>(CREATE_GROUP_SCOPE_ID).get()
    }

    /** Steps. The open flow's data; after process death an empty one, so a step never crashes. */
    val data: CreateGroupFlowData
        get() = koin.getOrCreateScope<CreateGroupFlowScope>(CREATE_GROUP_SCOPE_ID).get()

    /** Finish, and every exit. Safe to call when nothing is open. */
    fun close() {
        koin.getScopeOrNull(CREATE_GROUP_SCOPE_ID)?.close()
    }
}
```

The owner keeps no reference to the scope or the data: it looks the scope up by ID each time, so the scope
registry is the single source of truth and nothing can go stale.

## 3. Definitions — `feature/creategroup/di/CreateGroupKoinModule.kt`

The app module `includes(...)` the feature module as it does for every feature.

The ViewModel definitions (`viewModel<T>()`, `viewModel { }`, `@KoinViewModel`) live in `koin-core-viewmodel`,
which `koin-compose-viewmodel` and `koin-android` already bring in. A shared module that so far had only
`koin-core` needs that artifact in `commonMain` — a Gradle change: version from the Koin BOM, and ask first.

### 3a. Compiler Plugin DSL

```kotlin
import org.koin.dsl.module
import org.koin.plugin.module.dsl.*

val createGroupKoinModule = module {
    scope<CreateGroupFlowScope> {
        scoped<CreateGroupFlowData>()
    }
    single<CreateGroupFlow>()

    viewModel<GroupNameViewModel>()
    viewModel<HouseSelectionViewModel>()
    viewModel<ReviewViewModel>()
    viewModel<CreateGroupResultViewModel>()
}
```

- The plugin passes the root `Scope` to `CreateGroupFlow` by itself — nothing to declare for it.
- `scoped<CreateGroupFlowData>()` calls the empty constructor because every property has a default
  (`skipDefaultValues`, on by default). A project that turned that option off gets
  `[KOIN-D001] Missing dependency: kotlin.String` → write `scoped { CreateGroupFlowData() }` there.

### 3b. Without the Compiler Plugin

```kotlin
import org.koin.core.module.dsl.viewModel
import org.koin.dsl.module

val createGroupKoinModule = module {
    scope<CreateGroupFlowScope> {
        scoped { CreateGroupFlowData() }
    }
    single { CreateGroupFlow(this) }   // `this` is the root Scope

    viewModel { GroupNameViewModel(get(), get()) }
    viewModel { HouseSelectionViewModel(get(), get()) }
    viewModel { ReviewViewModel(get(), get(), get()) }
    viewModel { CreateGroupResultViewModel(get(), get()) }
}
```

- A project on the constructor DSL writes its ViewModels as `viewModelOf(::HousePickerViewModel)`. The two flow
  lines stay lambdas: `singleOf(::CreateGroupFlow)` and `scopedOf(::CreateGroupFlowData)` both compile and both
  fail at runtime (`NoDefinitionFoundException` for `Scope` / for `String`).

### 3c. Annotations

No module code: annotate the classes, and the feature's `@ComponentScan` finds them.

```kotlin
// CreateGroupFlowData.kt
import org.koin.core.annotation.Scope
import org.koin.core.annotation.Scoped

@Scope(CreateGroupFlowScope::class)
@Scoped
data class CreateGroupFlowData(
```

```kotlin
// CreateGroupFlow.kt — org.koin.core.scope.Scope for the parameter, not the annotation
import org.koin.core.annotation.Singleton

@Singleton
class CreateGroupFlow(private val rootScope: Scope) {
```

```kotlin
// every ViewModel of the flow
import org.koin.core.annotation.KoinViewModel

@KoinViewModel
class GroupNameViewModel(
```

```kotlin
// only if the feature has no module yet
@Module
@Configuration                                   // on the Gradle module's root module only
@ComponentScan("com.acme.feature.creategroup")   // the feature's own package, never a broad root
class CreateGroupModule
```

## 4. Start the flow — entry screen ViewModel

```kotlin
class HomeViewModel(
    private val createGroupFlow: CreateGroupFlow,
    logger: AppLogger,
) : BaseViewModel<HomeContract.Event, HomeContract.State, HomeContract.Effect>(logger) {
```

```kotlin
is HomeContract.Event.OnCreateGroupClicked -> {
    // start() always returns empty data: an earlier, abandoned flow cannot leak in.
    createGroupFlow.start().apply {
        pilgrimIds = viewState.value.selectedPilgrimIds   // initial data, if any
    }
    setEffect { HomeContract.Effect.Navigation.NavigateToGroupName }
}
```

## 5. A step that collects input

```kotlin
class GroupNameViewModel(
    private val flow: CreateGroupFlow,
    logger: AppLogger,
) : BaseViewModel<GroupNameContract.Event, GroupNameContract.State, GroupNameContract.Effect>(logger) {

    // A getter, never a field: a stored reference would outlive the flow it came from.
    private val flowData: CreateGroupFlowData get() = flow.data

    override fun setInitialState() = GroupNameContract.State()

    override fun loadInitData() {
        // Coming back to this step keeps what was typed before.
        setState { copy(groupName = flowData.groupName, isNextEnabled = flowData.groupName.isNotBlank()) }
    }

    override fun handleEvents(event: GroupNameContract.Event) {
        when (event) {
            is GroupNameContract.Event.OnGroupNameChanged ->
                setState { copy(groupName = event.name, isNextEnabled = event.name.isNotBlank()) }

            GroupNameContract.Event.OnNextClicked -> {
                flowData.groupName = viewState.value.groupName.trim()
                setEffect { GroupNameContract.Effect.Navigation.NavigateToHouseSelection }
            }

            // First step: leaving it leaves the flow.
            GroupNameContract.Event.OnBackClicked -> {
                flow.close()
                setEffect { GroupNameContract.Effect.Navigation.Back }
            }
        }
    }
}
```

## 6. A step that needs earlier data (with the missing-data guard)

```kotlin
class ReviewViewModel(
    private val flow: CreateGroupFlow,
    private val createGroupUseCase: CreateGroupUseCase,
    logger: AppLogger,
) : BaseViewModel<ReviewContract.Event, ReviewContract.State, ReviewContract.Effect>(logger) {

    private val flowData: CreateGroupFlowData get() = flow.data

    override fun setInitialState() = ReviewContract.State()

    override fun loadInitData() {
        val name = flowData.groupName
        if (name.isBlank()) {
            // Process death or an unexpected entry point: the flow has no data.
            flow.close()
            setEffect { ReviewContract.Effect.Navigation.NavigateToFlowStart }
            return
        }
        setState { copy(groupName = name, house = flowData.selectedHouse, pilgrimCount = flowData.pilgrimIds.size) }
    }

    override fun handleEvents(event: ReviewContract.Event) {
        when (event) {
            ReviewContract.Event.OnSubmitClicked -> submit()
        }
    }

    private fun submit() {
        viewModelScope.launch {
            createGroupUseCase(flowData.groupName, flowData.selectedHouse!!.id, flowData.pilgrimIds).collect { result ->
                when (result) {
                    is ApiResult.Success -> {
                        flowData.createdGroup = result.value
                        setEffect { ReviewContract.Effect.Navigation.NavigateToResult }
                    }
                    is ApiResult.Error -> setEffect { ReviewContract.Effect.ShowError(result.message) }
                    is ApiResult.Loading -> setState { copy(isLoading = true) }
                }
            }
        }
    }
}
```

## 7. Finish — result screen

```kotlin
override fun loadInitData() {
    val group = flowData.createdGroup
    if (group == null) {
        flow.close()
        setEffect { CreateGroupResultContract.Effect.Navigation.BackToHome }
        return
    }
    // Copy everything the screen shows into State now: after close() the data is gone.
    setState { copy(groupId = group.id.toString(), groupName = group.name) }
}

override fun handleEvents(event: CreateGroupResultContract.Event) {
    when (event) {
        CreateGroupResultContract.Event.OnDoneClicked -> {
            flow.close()
            setEffect { CreateGroupResultContract.Effect.Navigation.BackToHome }
        }
        CreateGroupResultContract.Event.OnCreateAnotherClicked -> {
            flow.start()   // a new run with empty data, whichever GroupName ViewModel shows it
            setEffect { CreateGroupResultContract.Effect.Navigation.NavigateToGroupName }
        }
    }
}
```

## 8. Picker that returns a value to the previous step

Picker ViewModel:
```kotlin
is HousePickerContract.Event.OnHouseSelected -> {
    flowData.selectedHouse = event.house
    setEffect { HousePickerContract.Effect.Navigation.Back }
}
```

Previous step — re-read on resume:
```kotlin
// Contract
data object OnScreenResumed : Event
```
```kotlin
// ViewModel — back from the picker: re-read what it wrote.
HouseSelectionContract.Event.OnScreenResumed -> setState { copy(selectedHouse = flowData.selectedHouse) }
```

Android (Compose):
```kotlin
LifecycleResumeEffect(Unit) {
    viewModel.setEvent(HouseSelectionContract.Event.OnScreenResumed)
    onPauseOrDispose { }
}
```

iOS (SwiftUI):
```swift
.onAppear { viewModel.setEvent(event: HouseSelectionContractEventOnScreenResumed.shared) }
```

## 9. Hand-off to another flow

The source ViewModel receives the **target** flow's owner and starts it, exactly like an entry screen:

```kotlin
class EditGroupViewModel(
    private val addPilgrimsFlow: AddPilgrimsFlow,   // the target flow's owner, injected like any dependency
    logger: AppLogger,
) : BaseViewModel<EditGroupContract.Event, EditGroupContract.State, EditGroupContract.Effect>(logger) {
```

```kotlin
// Open the "add pilgrims" flow pre-filled with this group.
private fun openAddPilgrims() {
    addPilgrimsFlow.start().apply {
        groupId = viewState.value.groupId
        existingPilgrimIds = viewState.value.pilgrims.map { it.id }
    }
    setEffect { EditGroupContract.Effect.Navigation.NavigateToAddPilgrims }
}
```

## 10. Platforms

Android:
```kotlin
@Destination
@Composable
fun GroupNameScreen(
    navigator: DestinationsNavigator,
    viewModel: GroupNameViewModel = koinViewModel(),
) { /* render viewModel.viewState; handle effects; send events. No flow-scope code here. */ }
```

iOS — `KoinHelper` (iosMain), the bridge at the platform boundary. ViewModel getters only:
```kotlin
class KoinHelper : KoinComponent {
    fun getGroupNameViewModel(): GroupNameViewModel = get()
    fun getCreateGroupResultViewModel(): CreateGroupResultViewModel = get()
}
```

## Tests — `commonTest`

### The owner test — required for every flow

The build cannot check the owner's scoped lookup, so this test is what proves the scope is wired.

```kotlin
import org.koin.dsl.koinApplication
import kotlin.test.AfterTest
import kotlin.test.Test
import kotlin.test.assertEquals
import kotlin.test.assertFalse
import kotlin.test.assertSame
import kotlin.test.assertTrue

class CreateGroupFlowTest {
    // An isolated container, not startKoin: nothing global to stop between tests.
    private val koin = koinApplication { modules(appModule) }.koin
    private val flow: CreateGroupFlow = koin.get()

    @AfterTest fun tearDown() = koin.close()

    @Test fun steps_share_the_data_the_entry_point_filled() {
        flow.start().groupName = "A"

        assertEquals("A", flow.data.groupName)
        assertSame(flow.data, flow.data)
    }

    @Test fun starting_again_gives_fresh_data() {
        flow.start().groupName = "abandoned"

        assertEquals("", flow.start().groupName)
    }

    @Test fun closing_leaves_no_scope_open() {
        flow.start()
        assertTrue(flow.isOpen)

        flow.close()
        flow.close()   // a second close is harmless

        assertFalse(flow.isOpen)
    }

    @Test fun a_step_after_process_death_gets_empty_data_instead_of_crashing() {
        assertFalse(flow.isOpen)   // nothing started the flow in this process

        assertEquals("", flow.data.groupName)
    }
}
```

Which module to load:

- **`appModule`**, as above. With the Compiler Plugin a `koinApplication { }` is checked like an app entry
  point, so loading only `createGroupKoinModule` fails the build with `[KOIN-D001]` for everything its
  ViewModels get from other modules. Nothing platform-specific is created: the test resolves only the owner.
- **Annotations** → `koinApplication<KoinApp>().koin`, with `koinApplication` from `org.koin.plugin.module.dsl`
  and `KoinApp` the project's `@KoinApplication` type.
- **Without the Compiler Plugin** → `modules(createGroupKoinModule)` alone is enough.
- **The test cannot see the app module** (a feature Gradle module) → move the scope and owner definitions
  into their own `val createGroupFlowModule = module { … }`, `includes(createGroupFlowModule)` from the feature
  module, and load only that one in the test.

### ViewModel tests

Build the ViewModel directly: the owner from the same isolated container, its other dependencies by hand
(fakes for anything that touches the network or the platform).

```kotlin
class CreateGroupViewModelsTest {
    private val koin = koinApplication { modules(appModule) }.koin
    private val flow: CreateGroupFlow = koin.get()
    private val logger = AppLogger()

    @AfterTest fun tearDown() = koin.close()

    private fun groupName() = GroupNameViewModel(flow, logger).apply { loadInitData() }

    @Test fun next_writes_the_input_for_the_following_step() {
        flow.start()
        val step = groupName()

        step.setEvent(GroupNameContract.Event.OnGroupNameChanged("  Group A "))
        step.setEvent(GroupNameContract.Event.OnNextClicked)

        assertEquals("Group A", flow.data.groupName)
        assertEquals(GroupNameContract.Effect.Navigation.NavigateToHouseSelection, step.effects.last())
    }

    @Test fun back_on_the_first_step_closes_the_flow() {
        flow.start()

        groupName().setEvent(GroupNameContract.Event.OnBackClicked)

        assertFalse(flow.isOpen)
    }

    @Test fun a_step_without_its_data_goes_back_to_the_start_and_leaves_nothing_open() {
        val review = ReviewViewModel(flow, CreateGroupUseCase(), logger).apply { loadInitData() }   // as after process death

        assertEquals(ReviewContract.Effect.Navigation.NavigateToFlowStart, review.effects.last())
        assertFalse(flow.isOpen)
    }
}
```

`step.effects` stands for however the team's base class exposes emitted effects in tests.
