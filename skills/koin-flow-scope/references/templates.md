# Templates

Worked example: a **Create group** flow — `Home` (entry) → `GroupName` → `HouseSelection` → `Review` → `Result`.
Replace `CreateGroup` / `createGroup` / `CREATE_GROUP` with your flow's name. The ViewModels assume the team's
MVI base class (`BaseViewModel<Event, State, Effect>` with `setState`, `setEffect`, `viewState`,
`loadInitData()`); adapt the calls if yours differ.

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

## 2. Koin module — `feature/creategroup/di/CreateGroupKoinModule.kt`

```kotlin
const val CREATE_GROUP_SCOPE_ID = "createGroupScope"
val CREATE_GROUP_SCOPE_QUALIFIER = named("createGroup")

val createGroupKoinModule = module {
    scope(CREATE_GROUP_SCOPE_QUALIFIER) {
        scoped { CreateGroupFlowData() }
    }

    factory { GroupNameViewModel(getKoin(), get()) }
    factory { HouseSelectionViewModel(getKoin(), get(), get()) }
    factory { ReviewViewModel(getKoin(), get(), get()) }
    factory { CreateGroupResultViewModel(getKoin(), get()) }
}
```

## 3. Start the flow — entry screen ViewModel

```kotlin
is HomeContract.Event.OnCreateGroupClicked -> {
    // Always start with fresh data: an earlier, abandoned flow may still be open.
    koin.getScopeOrNull(CREATE_GROUP_SCOPE_ID)?.close()
    val flowData: CreateGroupFlowData =
        koin.createScope(CREATE_GROUP_SCOPE_ID, CREATE_GROUP_SCOPE_QUALIFIER).get()
    flowData.pilgrimIds = viewState.value.selectedPilgrimIds   // initial data, if any
    setEffect { HomeContract.Effect.Navigation.NavigateToGroupName }
}
```

## 4. A step that collects input

```kotlin
class GroupNameViewModel(
    private val koin: Koin,
    logger: AppLogger,
) : BaseViewModel<GroupNameContract.Event, GroupNameContract.State, GroupNameContract.Effect>(logger) {

    private val flowScope = koin.getOrCreateScope(CREATE_GROUP_SCOPE_ID, CREATE_GROUP_SCOPE_QUALIFIER)
    private val flowData: CreateGroupFlowData get() = flowScope.get()

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
                flowScope.close()
                setEffect { GroupNameContract.Effect.Navigation.Back }
            }
        }
    }
}
```

## 5. A step that needs earlier data (with the missing-data guard)

```kotlin
override fun loadInitData() {
    val name = flowData.groupName
    if (name.isBlank()) {
        // Process death or an unexpected entry point: the flow has no data.
        setEffect { ReviewContract.Effect.Navigation.NavigateToFlowStart }
        return
    }
    setState { copy(groupName = name, house = flowData.selectedHouse, pilgrimCount = flowData.pilgrimIds.size) }
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
```

## 6. Finish — result screen

```kotlin
override fun loadInitData() {
    val group = flowData.createdGroup ?: return setEffect { CreateGroupResultContract.Effect.Navigation.BackToHome }
    setState { copy(groupId = group.id.toString(), groupName = group.name) }
}

override fun handleEvents(event: CreateGroupResultContract.Event) {
    when (event) {
        CreateGroupResultContract.Event.OnDoneClicked -> {
            setEffect { CreateGroupResultContract.Effect.Navigation.BackToHome }
            flowScope.close()
        }
        CreateGroupResultContract.Event.OnCreateAnotherClicked -> {
            setEffect { CreateGroupResultContract.Effect.Navigation.NavigateToGroupName }
            flowScope.close()   // the next GroupName step starts a new, empty scope
        }
    }
}
```

## 7. Picker that returns a value to the previous step

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

// ViewModel
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

## 8. Hand-off to another flow

```kotlin
// In EditGroupViewModel: open the "add pilgrims" flow pre-filled with this group.
private fun openAddPilgrims() {
    koin.getScopeOrNull(ADD_PILGRIMS_SCOPE_ID)?.close()
    val target: AddPilgrimsFlowData =
        koin.createScope(ADD_PILGRIMS_SCOPE_ID, ADD_PILGRIMS_SCOPE_QUALIFIER).get()
    target.groupId = viewState.value.groupId
    target.existingPilgrimIds = viewState.value.pilgrims.map { it.id }
    setEffect { EditGroupContract.Effect.Navigation.NavigateToAddPilgrims }
}
```

## 9. Platforms

Android:
```kotlin
@Destination
@Composable
fun GroupNameScreen(
    navigator: DestinationsNavigator,
    viewModel: GroupNameViewModel = koinViewModel(),
) { /* render viewModel.viewState; handle effects; send events. No flow-scope code here. */ }
```

iOS — `KoinHelper` (iosMain):
```kotlin
fun getGroupNameViewModel() = get<GroupNameViewModel>()
fun getCreateGroupResultViewModel() = get<CreateGroupResultViewModel>()
```

## Tests — `commonTest`

No global Koin and no `koin-test` dependency needed:

```kotlin
class CreateGroupFlowScopeTest {
    private fun newKoin() = koinApplication {
        modules(module { scope(CREATE_GROUP_SCOPE_QUALIFIER) { scoped { CreateGroupFlowData() } } })
    }.koin

    private fun Koin.startFlow(): CreateGroupFlowData {
        getScopeOrNull(CREATE_GROUP_SCOPE_ID)?.close()
        return createScope(CREATE_GROUP_SCOPE_ID, CREATE_GROUP_SCOPE_QUALIFIER).get()
    }

    @Test
    fun stepsShareOneInstance() {
        val koin = newKoin()
        koin.startFlow().groupName = "A"
        val step2: CreateGroupFlowData =
            koin.getOrCreateScope(CREATE_GROUP_SCOPE_ID, CREATE_GROUP_SCOPE_QUALIFIER).get()
        assertEquals("A", step2.groupName)
    }

    @Test
    fun startingAgainGivesFreshData() {
        val koin = newKoin()
        koin.startFlow().groupName = "abandoned"
        assertEquals("", koin.startFlow().groupName)
    }

    @Test
    fun closingRemovesTheScope() {
        val koin = newKoin()
        koin.startFlow()
        koin.getScope(CREATE_GROUP_SCOPE_ID).close()
        assertNull(koin.getScopeOrNull(CREATE_GROUP_SCOPE_ID))
    }
}
```

To test a ViewModel, pass the same `koin` into its constructor and drive it with events.
