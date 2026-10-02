---
name: engine-internals
description: Architecture of the OpenGOAL C++ runtime, compiler, decompiler, Taskfile commands, and multi-layer build workflow.
---

# Engine Internals & Build System Architecture

OpenGOAL ports the Jak & Daxter trilogy to native x86-64. Understanding the
relationship between the C++ runtime, the offline decompiler, the `goalc`
compiler, and the GOAL game code is essential for fast iteration and
avoiding unnecessary full builds.

---

## 1. The 3-layer mental model

Never wait for a build you don't need. The codebase is organized into three
independent layers:

```
LAYER 1 -- C++ Runtime & Compiler (gk, goalc)
  Source: game/  common/  goalc/  + third-party
  Build command: task build-release-game
  Needed when: editing engine C++, renderers (Merc2/Tfrag), mips2c, goalc

LAYER 2 -- Decompiler & Asset Extraction (decompiler)
  Source: decompiler/  common/
  Build command: task build-release-decomp
  Needed when: editing extraction configs (decompiler/config/jak[x]/*.jsonc)
               or decompiler C++ code. Must re-run 'task extract' after.

LAYER 3 -- GOAL Game Code (*.gc, *.gp, *.gd)
  Source: goal_src/jak[x]/
  Build workflow: interactive REPL -- task repl, then (mi)
  Needed when: editing gameplay logic, states, types, actors, levels
  No C++ build required -- hot-reloads in seconds into the running game.
```

Rule of thumb: over 90% of modding happens in Layer 3. Do not run C++
rebuilds if you only modified `.gc` files.

---

## 2. Core binaries & engine responsibilities

1. **`gk` (Game Kernel Runtime)** — implements the virtual PS2 machine in
   native C++. Allocates one contiguous memory block via `mmap`
   (`EE_MAIN_MEM_SIZE`) to simulate Emotion Engine RAM. Hosts the
   OpenGL/Vulkan PC renderers (`Merc2`, `Tfrag`, `Tie`, `Shrub`, `Direct`).
   Runs `overlord`, the audio engine handling SBK sound banks and streaming
   audio. Listens on a local socket for connections from `goalc`.
2. **`goalc` (OpenGOAL Compiler)** — compiles GOAL source code (`.gc`)
   directly into native x86-64 machine instructions. Acts as an interactive
   REPL capable of live hot-reloading code into the running `gk` instance
   over the network socket. Packs objects into DGO/CGO container files.
3. **`decompiler`** — extracts assets, textures, collision geometry, and
   game objects from retail PS2 ISOs. Disassembles MIPS assembly into
   decompiled GOAL code. Bakes level geometry and extra models into `.fr3`
   files consumed by the PC renderers.

---

## 3. Taskfile command reference

```bash
# Game target selection
task set-game-jak1   # Set active target game to Jak 1
task set-game-jak2   # Set active target game to Jak 2
task set-game-jak3   # Set active target game to Jak 3

# Build & compilation
task gen-cmake-release       # Configure CMake with Ninja + Clang (auto-configures sccache if installed)
task build-release           # Build ALL ~20 binaries (first-time setup or full check)
task build-release-game      # Fast: builds ONLY gk and goalc (C++ runtime iteration)
task build-release-decomp    # Fast: builds ONLY the decompiler (asset pipeline iteration)
task extract                 # Extract assets and run decompiler (offline asset generation)

# Game execution & iteration
task repl                    # Open goalc interactive compiler
# Inside REPL:
(mi)                          # Incrementally compile and hot-reload the active project
task boot-game                # Boot game directly without REPL attached
task run-game                  # Boot game and attach REPL automatically
task format                    # Format all C++ and GOAL files

# Modding tool wrappers (scripts/modding/*.py)
task modding-new-branch -- jak2/features/my-mod   # Create mod branch from master-dev + README template
task modding-sync-branch                          # Safe git merge of master-dev into current branch
task kb-update                                    # Refresh this knowledge base (the .agents/skills submodule)
```

For every other task (decompiling, asset ripping, tools, tests) see
[`docs/modding/guides/task_scripts_reference.md`](../../../docs/modding/guides/task_scripts_reference.md).

---

## 4. Decompiler asset pipeline workflow

When adding custom actors or modifying `decompiler/config/jak[x]/*.jsonc`:

1. Rebuild decompiler: `task build-release-decomp`
2. Re-run extraction: `task extract`
3. A decompiler change is inert until extraction is re-run — the game loads
   baked `.fr3` files and `.go` files produced offline by extraction.

---

## 5. Retail boot vs. debug boot

The OpenGOAL Launcher always boots with `-boot -fakeiso` and no `-debug`
flag. That single difference disables the debug menu, the debug heap, and
the L3+SELECT debug-menu gesture all at once — any player-facing mod code
must work without any of those three. The full mechanism (which C++ flags
are involved, and why it makes L3+SELECT reliably free for the Mods menu to
claim instead) is documented in
[`docs/modding/guides/mods_menu.md`](../../../docs/modding/guides/mods_menu.md) §1.

`kmalloc` on a heap that is NULL in the current boot mode (e.g. `'debug` in
a retail boot) does not fail loudly — it silently falls back to the global
heap. Code that allocates on `'debug` every frame in a path retail can reach
is a permanent leak, not a crash. Prefer `'stack` for scratch data inside a
function body unless the code is genuinely debug-only.

---

## 6. Release packaging & verification

A packaged release needs its binaries at the archive root (`extractor`,
`gk`, `goalc`) plus `data/game/graphics/opengl_renderer/shaders/` and
`data/decompiler/config/`. Missing the shaders directory makes `gk` crash
immediately on launch — always spot-check a packaged build boots before
publishing a release. See
[`docs/modding/guides/mod_distribution_guide.md`](../../../docs/modding/guides/mod_distribution_guide.md)
for the full packaging and catalog pipeline.

When driving `gk` from a script or CI step, its exit codes need
interpreting rather than treated as pass/fail directly: a harness timeout
that kills a still-running game is success (the process was fine, you just
stopped waiting), a human closing the window or a normal shutdown are not
failures either — only a genuine segfault exit code is a real failure.
Don't wire a boot-smoke-test to "exit code 0 or nothing" without accounting
for this.

---

## See also

- [`docs/modding/guides/task_scripts_reference.md`](../../../docs/modding/guides/task_scripts_reference.md) — every task, with concrete examples.
- [`docs/modding/guides/build_and_iteration_workflow.md`](../../../docs/modding/guides/build_and_iteration_workflow.md) — the 3-layer build model in more depth.
- [`repl-workflow.md`](repl-workflow.md) — REPL lifecycle, heap architecture, and the "ghost memory" hot-reload pitfall.
- [`goal-lisp`](../goal-lisp/SKILL.md) — the language this engine compiles and runs.
