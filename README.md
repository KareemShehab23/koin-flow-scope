# koin-flow-scope (skill)

One way, for the whole team, to pass data between screens in a Kotlin Multiplatform app
(shared ViewModels + Koin, Jetpack Compose on Android, SwiftUI on iOS).

Each multi-screen flow gets **one Koin scope** holding a `{Feature}FlowData` object. The flow's first step
creates the scope with fresh data, every step reads it in `loadInitData()` and writes it before navigating,
and the last step — or any exit — closes it. Screens never pass flow data through navigation arguments.

```
skills/koin-flow-scope/
├── SKILL.md                  # rules, lifecycle, recipes, migration + review checklists
└── references/templates.md   # copy-paste code: FlowData, Koin module, start/step/finish/cancel,
                              # picker result, hand-off to another flow, Android/iOS, commonTest tests
```

## Install

**Claude Code** — the repo is its own marketplace:
```
/plugin marketplace add KareemShehab23/koin-flow-scope
/plugin install koin-flow-scope@koin-flow-scope
```
Claude then follows the skill whenever you build a flow, pass data to the next screen, or refactor
navigation arguments. Ask it directly too: "migrate the X flow to a koin flow scope".

**Without Claude** — read `skills/koin-flow-scope/SKILL.md`; section 8 is the PR review checklist.

## The rules in one table

| Moment | Code |
|---|---|
| Start (entry event) | `koin.getScopeOrNull(ID)?.close()` + `koin.createScope(ID, QUALIFIER)` |
| Each step | `koin.getOrCreateScope(ID, QUALIFIER)`; read in `loadInitData()`, write before `Navigate` |
| Finish | `flowScope.close()` |
| Cancel (first-step Back, ✕, exit) | `flowScope.close()` |
| Missing data (process death) | navigate back to the flow start |

The templates' tests were checked against Koin 4.0.0.
