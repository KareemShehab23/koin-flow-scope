---
name: koin-flow-scope
description: >
  Team standard for passing data between screens in a Kotlin Multiplatform app
  (shared ViewModels, Koin, Compose on Android, SwiftUI on iOS): one Koin scope
  per multi-screen flow holding a {Feature}FlowData object, opened and closed by
  one owner class that ViewModels receive through their constructor. Use it for
  any task that adds a wizard or multi-step flow, passes data from one screen or
  ViewModel to the next, returns a result to the previous screen, hands off to
  another flow, replaces navigation arguments, or touches {Feature}Flow,
  FlowData, createScope, getOrCreateScope or SCOPE_ID code, even when the
  request never says "scope". Load it together with the official koin skill:
  this skill decides where flow data lives and when the scope opens and closes,
  koin owns the Koin syntax. Not for one screen's own state or app-wide state.
---

# Koin flow scope — passing data between screens

One rule for the whole team: **data that travels through a multi-screen flow lives in a `{Feature}FlowData`
object inside a Koin scope that exists only while that flow is open, and one class — the flow's owner,
`{Feature}Flow` — opens and closes that scope.** ViewModels get the owner through their constructor. Screens
never pass flow data to each other through navigation arguments, singletons, or the UI layer.

```
Entry screen ─start()─▶ Step 1 ──▶ Step 2 ──▶ … ──▶ Result screen ─close()
                          │ ▲         │ ▲                 │
                          ▼ │ .data   ▼ │ .data           ▼ .data
      ViewModels ──▶ {Feature}Flow — the one owner, injected by constructor
                          │ opens · replaces · closes
                          ▼
              Koin scope {Feature}FlowScope ──▶ {Feature}FlowData

      Back / ✕ that leaves the flow ─close()
```

## 0. With the official `koin` skill

This standard is built on the official Koin skills (https://github.com/InsertKoinIO/koin-skills) and loads
together with them. One owner per question:

| Question | Owner |
|---|---|
| Where flow data lives, who opens and closes the scope, when a step reads and writes | **this skill** |
| Koin syntax and imports, definition style, modules, versions, Gradle, build verification, runtime checks | **`koin` skill** |

- **Read the project first**, as `koin` does: Koin Compiler Plugin or not, DSL or Annotations. Write the flow
  in *that* style — the templates give every form. Never mix styles, never add Koin KSP, never rewrite
  existing definitions into another form unasked.
- `koin` checks with the user before adding a scope. For a multi-screen flow this standard is the team's
  standing answer: say in one line that the flow gets a flow scope and why (§1), then go on. Ask only when §1
  doesn't clearly point here.
- **Why a custom scope and not an archetype:** the data is shared by several ViewModels, on two UI toolkits.
  The ViewModel scope lives and dies with one ViewModel, and a nav-graph ViewModel exists only on the Compose
  side — no archetype fits, so the scope is opened and closed by code.
- If a `koin` reference shows a newer syntax for a call used here, theirs wins. The lifecycle rules in §3 stay.
- `koin` skill not installed → the templates cover the flow itself; any other Koin question is answered from
  https://insert-koin.io/docs/reference/koin-core/scopes/, not from memory.

## 1. Pick the right tool first

| Situation | Use |
|---|---|
| A screen needs one or two IDs to open, and it can be opened on its own (deep link, list → details) | Navigation argument / `parametersOf(id)` — values only |
| State of one screen, or objects that live exactly as long as one ViewModel | The ViewModel, or the ViewModel scope (`koin` skill) — not this skill |
| Two or more screens in a row build up or share data (wizard, create/edit flow, scan → validate → confirm) | **Flow scope (this skill)** |
| A screen picks something and returns it to the screen before it, inside a flow | Flow scope + re-read on resume (§5.4) |
| App-wide state (session, user, feature flags) | A `single` repository/manager — not this skill |

If in doubt: more than one hop, or more than two values → flow scope.

## 2. The four pieces

Every flow has exactly these, all in the **shared** module, in the feature's packages
(`feature/<feature>/di`, `feature/<feature>/domain/model`, `feature/<feature>/<step>/presentation`).

1. **`{Feature}FlowData`** — `domain/model/{Feature}FlowData.kt`. A `data class` with `var` properties, every one
   with a default. Plain data only: no flows, no coroutines, no Android/iOS types, no callbacks.
2. **`{Feature}FlowScope`** — the scope's type: an empty marker class. The scope is typed, never `named("…")`.
3. **`{Feature}Flow`** — the owner, `di/{Feature}Flow.kt`. The only class that knows the scope ID and calls
   Koin. It takes the root `Scope` in its constructor, is declared as a singleton, and offers `start()` (a new
   flow with empty data), `data` (the open flow's data), `close()` and `isOpen`.
4. **Definitions in the feature's Koin module** — the scoped `{Feature}FlowData`, the owner, and the ViewModels
   as ViewModel definitions. A ViewModel takes `flow: {Feature}Flow` like any other dependency: no `Koin`
   parameter, no `KoinComponent`, no Koin import.

Copy-paste templates for all four, in each definition style: [references/templates.md](references/templates.md).

## 3. Lifecycle rules (the part people get wrong)

| Moment | Who | Code | Why |
|---|---|---|---|
| **Start** | The event that launches the flow (button on the entry screen) | `flow.start()`, fill the initial fields, then `setEffect { Navigate… }` | `start()` closes whatever an abandoned flow left behind and opens a new scope: no half-filled data leaks into the new run. |
| **Each step** | Step ViewModel | `private val flowData: {Feature}FlowData get() = flow.data` | A getter, so the ViewModel always sees the flow that is open now. `data` never throws: after process death it returns empty data. |
| **Read** | `loadInitData()` | copy what the screen shows from `flowData` into `State` | UI renders `State`, never `flowData`. |
| **Write** | the event handler, right before `setEffect { Navigate… }` | `flowData.x = viewState.value.x` | The next step reads it in its own `loadInitData()`. |
| **Finish** | Result / last step | `flow.close()` then `setEffect { … }` | Frees the data; nothing stays open. |
| **Cancel** | Back/✕ on the **first** step, and every "exit flow" action on any step | `flow.close()` then `setEffect { Back }` | Without this the scope lives until the app dies. |
| **Run again** | "Create another" on the result screen | `flow.start()` then navigate to step 1 | Same as Start: empty data, even for a step ViewModel still on the back stack. |
| **Missing data** | any step whose required field is null/empty | `flow.close()` then `setEffect { NavigateToFlowStart }` (or show an error) | Happens after process death or a bad deep link. Flow data is not saved across process death — by design. |

Back from step N to step N-1 does **not** close the scope — the flow is still open.
After `close()` a ViewModel does not read `flowData` again: what the screen still shows is already in `State`.

## 4. What never to do

- ❌ `koin: Koin`, `getKoin()`, `KoinComponent` or any scope call in a ViewModel — a service locator in a normal
  layer: hidden dependencies, nothing checked at build time. Only `{Feature}Flow` talks to Koin.
- ❌ `createScope` / `getOrCreateScope` / `scope.close()` anywhere but the owner. One owner per scope.
- ❌ `private val flowData = flow.data` (a field), or a `Scope` kept in a ViewModel — it outlives the flow, and
  the next read hits stale data or throws `ClosedScopeException`.
- ❌ `single { {Feature}FlowData() }` — data survives across flows and users.
- ❌ `singleOf(::{Feature}Flow)` or `scopedOf(::{Feature}FlowData)` — the constructor DSL tries to inject the
  `Scope` and every defaulted property: both build, both fail at runtime with `NoDefinitionFoundException`.
  Use the forms in the templates (§3a with the Compiler Plugin, §3b without).
- ❌ A ViewModel declared as `factory { }` or `single { }` — ViewModel definitions only.
- ❌ Reading or writing flow data from a Composable, a SwiftUI view, or `KoinHelper`. Only ViewModels.
- ❌ Rendering `flowData` directly or treating it as observable state. Copy into `State`.
- ❌ Two flows sharing one scope type or scope ID. One `{Feature}FlowScope`, one ID, one owner per flow.
- ❌ Putting large/binary data (images, files) in FlowData — store a cache path/URI instead.
- ❌ Passing the same data both as a navigation argument and in FlowData. Pick one (§1).
- ❌ `checkModules()` / `verify()` tests. The owner test (§7) is what checks the scope.

Asked for one of these anyway → say what it costs and offer the form above first; write the requested form
only after an explicit OK, and flag it in the summary.

## 5. Recipes

Full code for each is in [references/templates.md](references/templates.md).

1. **New flow from scratch** — create the four pieces (§2), wire start/finish/cancel (§3), add the ViewModel
   getters to iOS `KoinHelper`, write the owner test (§7).
2. **Step that collects input** — `OnNextClicked`: validate, write to `flowData`, navigate.
3. **Result screen** — read in `loadInitData()`, close in every exit event.
4. **Picker returns a value to the previous step** — the picker writes `flowData.selected = item` and goes back;
   the previous screen sends an `OnScreenResumed` event (Android: `LifecycleResumeEffect`/`ON_RESUME`;
   iOS: `.onAppear`) and its ViewModel re-reads `flowData` into `State`.
5. **Hand-off to another flow** (e.g. edit screen opens the "add" flow) — the source ViewModel takes the
   *target* flow's owner in its constructor and starts it like an entry screen (`start()`, fill, navigate).
   Never write into another flow through `data`.
6. **Edit flow pre-filled from a details screen** — its own `Edit{Feature}FlowData`, scope type and owner; the
   details screen starts it with the loaded model.

## 6. Platforms

- **Android**: `viewModel: {Step}ViewModel = koinViewModel()` (or `koinViewModel { parametersOf(…) }` for real
  navigation arguments). Nothing flow-related in the Composable.
- **iOS**: `KoinHelper` in `iosMain` is the bridge at the platform boundary — the one place `KoinComponent` is
  accepted. It exposes ViewModel getters only (`fun get{Step}ViewModel(): {Step}ViewModel = get()`); never the
  owner, the data or a scope. SwiftUI holds the ViewModel. Same scope, same data on both platforms.

## 7. Verify

1. **Build.** With the Compiler Plugin, a missing owner or ViewModel dependency is a `[KOIN-D001]` build error:
   fix what it reports. Without the plugin there is no build-time check — build, then run the flow.
2. **Owner test — required for every flow.** The owner's lookup of the scoped `{Feature}FlowData` is resolved
   only at runtime (the plugin reports at most a `KOIN-W009` warning), so a missing scoped definition passes
   the build and crashes on the first screen. The test (templates §Tests) runs `start` → `data` → `close` in
   an isolated `koinApplication` and proves: steps share one instance, starting again gives empty data, close
   leaves nothing open, `data` without a start is empty instead of a crash.
3. **Run it on both platforms**: complete once, cancel halfway and start again (must be empty).
4. Whether a scope stays open or is opened twice in a real session is measured, not guessed — that is the
   `koin` skill's runtime handoff; propose nothing extra from here.

## 8. Migrating existing screens (refactor checklist)

A restructuring, so it goes step by step as the `koin` skill requires: **one flow per step, one PR per flow**.
Propose the order (smallest flow first), get the OK on the plan and the first flow, and after each flow
build → run the flow → report → wait for the OK. The app works between steps: flows not yet migrated keep
their current wiring.

**From navigation arguments:**

1. List the screens of the flow and every value passed between them (nav args, `parametersOf`, shared
   singletons, saved-state handles, `rememberSharedViewModel`).
2. Create `{Feature}FlowData` with those values, the scope type, the owner and the definitions (§2).
3. Find the **start** event; add `flow.start()` + fill (§3).
4. In each step ViewModel: add `flow: {Feature}Flow` and the `flowData` getter; move reads to `loadInitData()`
   and writes to the event before navigation. Remove the now-unused constructor params / nav args.
5. Add `flow.close()` on finish **and** on every exit (first-step Back, ✕, "cancel" dialogs).
6. Update the ViewModel definitions and iOS `KoinHelper` getters; remove the old args from Android
   destinations and iOS initialisers.
7. Write the owner test, then verify (§7).

**From version 1 of this standard** (Koin inside the ViewModels):

| Version 1 | Now |
|---|---|
| `private val koin: Koin` in a ViewModel | `private val flow: {Feature}Flow` |
| `koin.getScopeOrNull(ID)?.close()` + `koin.createScope(ID, QUALIFIER).get()` | `flow.start()` |
| `private val flowScope = koin.getOrCreateScope(ID, QUALIFIER)` + `flowScope.get()` | `flow.data` (through the getter) |
| `flowScope.close()` | `flow.close()` |
| `{FEATURE}_SCOPE_QUALIFIER = named("{feature}")`, `scope(QUALIFIER) { … }` | `class {Feature}FlowScope`, `scope<{Feature}FlowScope> { … }` |
| public `{FEATURE}_SCOPE_ID` used by every ViewModel | private to the owner's file |
| `factory { {Step}ViewModel(getKoin(), …) }` | a ViewModel definition without `getKoin()` |
| tests on a hand-built module with the scope calls repeated | the owner test |

Per flow: add the scope type and the owner, switch **all** of that flow's ViewModels in the same change, then
delete the qualifier and the public ID. Never leave one flow half on each form — the two do not share a scope.

## 9. Review checklist (paste into PRs)

- [ ] One `{Feature}FlowScope`, one scope ID and one `{Feature}Flow` owner per flow, in the feature's `di` package
- [ ] FlowData: `data class`, `var`s with defaults, plain data only, declared `scoped` in the flow's scope
- [ ] Owner declared as a singleton; no `singleOf` / `scopedOf` for the owner or FlowData
- [ ] ViewModels: ViewModel definitions, `flow` through the constructor, no Koin import
- [ ] Start event: `flow.start()` + fill; steps: `flowData` getter, read in `loadInitData()`, write before navigating
- [ ] Closed on finish and on every exit path
- [ ] No scope or flow-data access outside ViewModels; UI renders `State` only
- [ ] Required-data check that closes the flow and leads back to the start
- [ ] Android destinations / iOS initialisers no longer pass flow data as arguments
- [ ] Owner test green; build green in the project's own Koin style
