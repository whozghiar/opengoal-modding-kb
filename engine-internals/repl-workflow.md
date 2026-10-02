# REPL Workflow, Heap Management & Ghost Memory Pitfalls

This document details the interactive development cycle of OpenGOAL, the
heap memory architecture, and the critical pitfalls of REPL hot-reloading
("ghost memory").

---

## 1. REPL lifecycle & interactive iteration

The `goalc` REPL allows live hot-reloading of GOAL code into a running game
without restarting the application or rebuilding C++ binaries.

### Standard daily workflow
1. Launch the game: `task boot-game` (or run in debug mode).
2. In a separate terminal, launch the REPL: `task repl`.
3. Connect to the running game with the REPL's "listen to target" command —
   this attaches the REPL to the running `gk` process.
4. Modify your `.gc` code in your editor.
5. In the REPL, run the "make incremental" command to compile only the
   modified `.gc` files and upload them to `gk`. Changes take effect within
   seconds.

Both commands are single top-level forms typed at the REPL prompt: `(lt)`
("listen to target") attaches to the running game, `(mi)` ("make
incremental") compiles and uploads the changed files. They are the same two
commands in every hot-reload cycle across the project. AI agents never run
`(lt)`: attaching to the game is the user's step.

---

## 2. Memory & heap architecture

The C++ runtime (`gk`) reserves a single contiguous block of virtual memory
using `mmap`, accurately mimicking the PS2 Emotion Engine RAM. All GOAL
allocations occur inside this block:

```
SIMULATED PS2 EE RAM
  Global Heap ('global)  |  Level Heap ('level)  |  Debug Heap ('debug)  |  Process Heap ('process)
```

| Heap | Role & contents | Lifetime | Target usage |
|---|---|---|---|
| Global heap | Symbol table, type descriptors, kernel, permanent processes (e.g. the player). | Entire game session | Permanent systems and singleton managers. Do not leak transient data here. |
| Level heap | Level geometry, resident DGO art-groups, level textures, spawned actors. | Level lifespan (cleared on level transition) | Actor spawning, level geometry, level art data. |
| Debug heap | Debug menu entries, profiler hooks, visualizer state. | Entire session | Debug-only code. Disappears cleanly in release builds. |
| Process heap | Process execution stack and private fields. | Process lifespan | Actor private state. |

---

## 3. The "ghost memory" pitfall

### The problem
When you hot-reload code, GOAL updates function code pointers and symbols
in-place within the live memory block. However:

1. **Struct layout changes.** If you alter a type by adding fields,
   reordering fields, or changing sizes, existing instances already
   allocated in RAM retain the old memory layout. Functions compiled with
   the new offsets reading old memory layouts cause silent data corruption
   or crashes.
2. **Order of declaration / forward references.** If file B depends on
   file A, but you previously loaded file B while A was already in memory
   from an earlier session, the hot reload succeeds. On a fresh boot,
   though, file A might not be loaded yet when file B compiles, causing a
   compilation failure that the hot session never showed you.
3. **Lingering symbols & functions.** Renaming or deleting a function or
   symbol does not remove it from the active REPL session's memory. Code
   referencing the deleted name keeps "working" in that session, masking a
   broken reference a clean boot would catch immediately.

### Mandatory cold boot verification rule

> [!CAUTION]
> Always validate code with a cold boot before committing. Never conclude
> a task based solely on hot-reloading. Always close the game and test a
> clean start:
> ```bash
> task boot-game
> ```
> This ensures that the declaration and compilation order in `.gp` is
> strictly correct, struct layouts are clean and initialized without
> legacy offsets, and every symbol and dependency resolves without relying
> on residual session memory.

---

## 4. Project file registration (`.gp`)

Hot-reloading a `.gc` file in the REPL does not mean the game knows how to
build it during a clean boot. Whenever you create a new `.gc` file:

1. Open the project file for your target game:
   - Jak 1: `goal_src/jak1/game.gp`
   - Jak 2: `goal_src/jak2/game.gp`
   - Jak 3: `goal_src/jak3/game.gp`
2. Add your file under the appropriate CGO/DGO group — the exact lines
   for each game are in the Lisp wiki,
   [Registering a new source file](../goal-lisp/wiki/common.md#registering-a-new-source-file).
3. Ensure dependent type definitions are declared in files listed above
   your file in `.gp`.

---

## 5. Save Slot 1 auto-load & PC settings/cheats persistence

### Default auto-load (Save Slot 1)
- On startup (cold boot), OpenGOAL reads the virtual memory card in
  `%APPDATA%/OpenGOAL/jak[x]/saves/BASCUS-.../`.
- If an existing save file is present in Slot 1, the engine restores it by
  default.
- **Testing tip:** to test a clean, unmodified initial state without prior
  quest progression, start a new game via the title menu or temporarily
  rename/remove the Slot 1 save files.

### OpenGOAL cheats & PC settings persistence
- OpenGOAL settings and toggled cheats are saved on disk in
  `%APPDATA%/OpenGOAL/jak[x]/settings/pc-settings.gc`.
- When a cheat (`city-peace`, `turbo-board`, `music-player`, etc.) is
  toggled via the in-game Debug menu or the Pause Secrets menu, the active
  cheat set is written to disk as a bitmask — one bit per cheat.
- **Result:** cheats remain permanently active on every subsequent launch
  until toggled off in-game or the settings file is cleared.
- Toggling a cheat off programmatically from the REPL is the same pattern
  as any other bitfield clear-and-save: clear the specific bit on the
  settings struct's cheat field, then call the settings-save function to
  persist it.
