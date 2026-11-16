# koin-flow-scope (skill)

One way, for the whole team, to pass data between screens in a Kotlin Multiplatform app
(shared ViewModels + Koin, Jetpack Compose on Android, SwiftUI on iOS).

Each multi-screen flow gets **one Koin scope** holding a `{Feature}FlowData` object, and **one owner class**,
`{Feature}Flow`, that opens and closes it. The entry screen calls `flow.start()`, every step reads `flow.data`
in `loadInitData()` and writes it before navigating, and the last step — or any exit — calls `flow.close()`.
ViewModels get the owner through their constructor; screens never pass flow data through navigation arguments.

Built on the official Koin skills, [InsertKoinIO/koin-skills](https://github.com/InsertKoinIO/koin-skills):
this skill decides where flow data lives and when the scope opens and closes, the official `koin` skill owns
the Koin syntax, the project's definition style, versions and build verification.

```
skills/koin-flow-scope/
├── SKILL.md                  # rules, lifecycle, recipes, verification, migration + review checklists
└── references/templates.md   # copy-paste code: FlowData, scope owner, definitions in each Koin style,
                              # start/step/finish/cancel, picker result, hand-off, Android/iOS, tests
```

## Install

**Claude Code** — the official Koin skills first, then this one (the repo is its own marketplace):
```
/plugin marketplace add InsertKoinIO/koin-skills
/plugin install koin@koin-skills
/plugin marketplace add KareemShehab23/koin-flow-scope
/plugin install koin-flow-scope@koin-flow-scope
```
Claude then follows both whenever you build a flow, pass data to the next screen, or refactor navigation
arguments. Ask it directly too: "migrate the X flow to a koin flow scope".

Already installed: `/plugin marketplace update koin-flow-scope`.

**Other agents** that read agent skills: `npx skills add KareemShehab23/koin-flow-scope`.

**Without an agent** — read `skills/koin-flow-scope/SKILL.md`; section 9 is the PR review checklist.

## The rules in one table

| Moment | Code |
|---|---|
| Start (entry event) | `flow.start()`, fill the initial fields, navigate |
| Each step | `private val flowData get() = flow.data`; read in `loadInitData()`, write before `Navigate` |
| Finish | `flow.close()` |
| Cancel (first-step Back, ✕, exit) | `flow.close()` |
| Run again | `flow.start()` |
| Missing data (process death) | `flow.close()`, navigate back to the flow start |

Only `{Feature}Flow` talks to Koin. ViewModels have no `Koin` parameter and no Koin import.

## What changed in 2.0.0

- **The skill loads again.** The 1.0.0 `SKILL.md` frontmatter was not valid YAML (an unquoted `Trigger on:`
  inside the description) and the description was over the 1024-character limit, so loaders that parse it
  strictly skipped the skill.
- **Koin moved out of the ViewModels.** 1.0.0 passed `koin: Koin` into every ViewModel, which the official Koin
  skills reject (a service locator in a normal layer, nothing checked at build time). One owner class per flow
  now holds every scope call, and a missing owner or ViewModel dependency is a build error with the Koin
  Compiler Plugin.
- **A crash is gone.** A 1.0.0 ViewModel kept its `Scope` in a field; if it outlived the flow ("Create another"
  with step 1 still on the back stack, or an event after Back) the next read threw `ClosedScopeException`.
- **Typed scope** (`scope<{Feature}FlowScope>`) instead of `named("…")`; **ViewModel definitions** instead of
  `factory { }`.
- **Every Koin style**: Compiler Plugin DSL, classic DSL, Annotations — the skill follows what the project uses.
- **An owner test per flow**, because the scoped lookup is the one thing the build cannot check.

Moving a 1.0.0 flow over: `SKILL.md` §8 has the line-by-line table.

## Checked against

The templates were compiled and their tests run on the JVM with Kotlin 2.3.20, Koin 4.2.2 and Koin Compiler
Plugin 1.2.3 — the minimum versions the official Koin skills target — in all three definition styles.
