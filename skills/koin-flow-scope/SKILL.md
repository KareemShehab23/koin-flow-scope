---
name: koin-flow-scope
description: Team standard for passing data between screens in a Kotlin Multiplatform app (shared ViewModels + Koin, Compose on Android, SwiftUI on iOS) using a Koin "flow scope" — one Koin scope per multi-screen flow that holds a {Feature}FlowData object every step reads and writes. Covers creating the scope ID/qualifier and scoped definition, starting a flow with fresh data, reading in loadInitData, writing before navigation, returning a result to the previous screen, handing off to another flow, closing on finish AND on cancel, process death, iOS access through KoinHelper, unit tests, and migrating screens from navigation arguments. ALWAYS read this before adding a new multi-step flow or wizard, passing data from one screen/ViewModel to the next, refactoring navigation arguments, or touching getOrCreateScope / createScope / getScope / FlowData / *_SCOPE_ID code. Trigger on: "pass data between screens", "send data to next screen", "share data between view models", "wizard", "multi-step flow", "navigation arguments", "nav args", "koin flow", "koin scope", "flow data", "result back to previous screen".
---

# Koin flow scope — passing data between screens

One rule for the whole team: **data that travels through a multi-screen flow lives in a `{Feature}FlowData`
object inside a Koin scope that exists only while that flow is open.** Screens never pass that data to each
other through navigation arguments, singletons, or the UI layer.

```
Entry screen ──start()──▶ Step 1 ──▶ Step 2 ──▶ … ──▶ Result screen ──close()
      │                    │  ▲         │                    │
      │                    ▼  │ read / write                 │
      └──────────────▶ Koin scope "{feature}Scope" ──▶ {Feature}FlowData
                                     ▲
                       Back / ✕ on any step ──close()
```

## 1. Pick the right tool first

| Situation | Use |
|---|---|
| A screen needs one or two IDs to open, and it can be opened on its own (deep link, list → details) | Navigation argument / `parametersOf(id)` |
| Two or more screens in a row build up or share data (wizard, create/edit flow, scan → validate → confirm) | **Flow scope (this skill)** |
| A screen picks something and returns it to the screen before it, inside a flow | Flow scope + re-read on resume (§5.4) |
| App-wide state (session, user, feature flags) | A `single` repository/manager — not this skill |

If in doubt: more than one hop, or more than two values → flow scope.

## 2. The four pieces

Every flow has exactly these, all in the **shared** module. File locations follow the feature package
convention (`feature/<feature>/di`, `feature/<feature>/domain/model`, `feature/<feature>/<step>/presentation`).

1. **`{Feature}FlowData`** — `domain/model/{Feature}FlowData.kt`. A `data class` with `var` properties, every one
   with a default. Plain data only: no flows, no coroutines, no Android/iOS types, no callbacks.
2. **Scope ID + qualifier + scoped definition** — in the feature's Koin module file:
   `const val {FEATURE}_SCOPE_ID`, `val {FEATURE}_SCOPE_QUALIFIER = named("{feature}")`,
   `scope({FEATURE}_SCOPE_QUALIFIER) { scoped { {Feature}FlowData() } }`.
3. **ViewModels take `koin: Koin`** as a constructor parameter and are registered as
   `factory { {Step}ViewModel(getKoin(), …) }`. They are the only code that touches the scope.
4. **A lifecycle**: one place starts the flow, every step reads/writes, the last step and every exit close it.

Copy-paste templates for all four: [references/templates.md](references/templates.md).

## 3. Lifecycle rules (the part people get wrong)

| Moment | Who | Code | Why |
|---|---|---|---|
| **Start** | The event that launches the flow (button on the entry screen) | `koin.getScopeOrNull(ID)?.close()` then `koin.createScope(ID, QUALIFIER)`, fill the initial fields, then `setEffect { Navigate… }` | A flow abandoned earlier must not leak its half-filled data into the new one. `getOrCreateScope` here is the #1 bug. |
| **Each step** | Step ViewModel | `private val flowScope = koin.getOrCreateScope(ID, QUALIFIER)` and `private val flowData: {Feature}FlowData get() = flowScope.get()` | `getOrCreateScope` (not `getScope`) so a step never crashes after process death. |
| **Read** | `loadInitData()` | copy what the screen shows from `flowData` into `State` | UI renders `State`, never `flowData`. |
| **Write** | the event handler, right before `setEffect { Navigate… }` | `flowData.x = viewState.value.x` | The next step reads it in its own `loadInitData()`. |
| **Finish** | Result / last step | `setEffect { … }` then `flowScope.close()` | Frees the data; the next start is clean. |
| **Cancel** | Back/✕ on the **first** step, and every "exit flow" action on any step | `flowScope.close()` then `setEffect { Back }` | Without this the scope lives until the app dies. |
| **Missing data** | any step whose required field is null/empty | `setEffect { NavigateToFlowStart }` (or show an error) | Happens after process death or a bad deep link. |

Back from step N to step N-1 does **not** close the scope — the flow is still open.

## 4. What never to do

- ❌ `single { {Feature}FlowData() }` — data survives across flows and users.
- ❌ Reading or writing the scope from a Composable, a SwiftUI view, or `KoinHelper`. Only ViewModels.
- ❌ Rendering `flowData` directly or treating it as observable state. Copy into `State`.
- ❌ `koin.getScope(ID)` in a step — throws `ScopeNotCreatedException` after process death. Use `getOrCreateScope`.
- ❌ `getOrCreateScope` in the **start** event — reuses stale data. Close then `createScope`.
- ❌ Two flows sharing one scope ID, or a scope ID string reused across features. One ID per flow, named after it.
- ❌ Putting large/binary data (images, files) in FlowData — store a cache path/URI instead.
- ❌ Passing the same data both as a navigation argument and in FlowData. Pick one (§1).

## 5. Recipes

Full code for each is in [references/templates.md](references/templates.md).

1. **New flow from scratch** — create the four pieces (§2), wire start/finish/cancel (§3), register the
   ViewModels in the feature module and in iOS `KoinHelper`.
2. **Step that collects input** — `OnNextClicked`: validate, write to `flowData`, navigate.
3. **Result screen** — read in `loadInitData()`, close in every exit event.
4. **Picker returns a value to the previous step** — the picker writes `flowData.selected = item` and goes back;
   the previous screen sends an `OnScreenResumed` event (Android: `LifecycleResumeEffect`/`ON_RESUME`;
   iOS: `.onAppear`) and its ViewModel re-reads `flowData` into `State`.
5. **Hand-off to another flow** (e.g. edit screen opens the "add" flow) — the source ViewModel *starts* the target
   flow exactly like an entry screen (close + create + fill), then navigates. Never write into another flow's
   scope that is already open.
6. **Edit flow pre-filled from a details screen** — a separate `Edit{Feature}FlowData` and scope; the details
   screen starts it with the loaded model.

## 6. Platforms

- **Android**: `viewModel: {Step}ViewModel = koinViewModel()` (or `koinViewModel { parametersOf(…) }` for real
  navigation arguments). Nothing flow-related in the Composable.
- **iOS**: expose `fun get{Step}ViewModel() = get<{Step}ViewModel>()` in `KoinHelper`; SwiftUI holds the
  ViewModel. Same scope, same data — iOS needs no extra code to share data between steps.

## 7. Migrating existing screens (refactor checklist)

For each flow, in this order — one PR per flow:

1. List the screens of the flow and every value passed between them (nav args, `parametersOf`, shared
   singletons, saved-state handles, `rememberSharedViewModel`).
2. Create `{Feature}FlowData` with those values, plus scope ID/qualifier/`scoped` (§2).
3. Find the **start** event; add close + create + fill (§3).
4. In each step ViewModel: add `koin: Koin`, `flowScope`, `flowData`; move reads to `loadInitData()` and writes
   to the event before navigation. Remove the now-unused constructor params / nav args.
5. Add close on finish **and** on every exit (first-step Back, ✕, "cancel" dialogs).
6. Update `factory { … }` registrations and iOS `KoinHelper` getters; remove the old args from Android
   destinations and iOS initialisers.
7. Unit-test the flow (templates §Tests): start → steps → finish leaves no scope; cancel leaves no scope;
   starting twice gives fresh data.
8. Run it on both platforms: complete once, cancel halfway and start again (must be empty).

## 8. Review checklist (paste into PRs)

- [ ] One `{FEATURE}_SCOPE_ID` + qualifier per flow, defined in the feature Koin module
- [ ] FlowData: `data class`, `var`s with defaults, plain data only
- [ ] Start event: `getScopeOrNull(ID)?.close()` + `createScope(ID, QUALIFIER)`
- [ ] Steps: `getOrCreateScope`, read in `loadInitData()`, write before navigating
- [ ] Closed on finish and on every exit path
- [ ] No scope access outside ViewModels; UI renders `State` only
- [ ] Required-data check with a way back to the start
- [ ] Android destinations / iOS initialisers no longer pass flow data as arguments
- [ ] Tests for finish, cancel and restart
