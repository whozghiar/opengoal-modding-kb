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
task run-game                  # Start the runtime; it waits for task repl to connect
task format                    # Format all C++ and GOAL files

# Modding tool wrappers (scripts/modding/*.py)
task modding-new-mod                             # Create a mod repository from master-dev (asks for the details)
task modding-sync-branch                          # Safe git merge of master-dev into the current mod
task kb-update                                    # Refresh this knowledge base (the .agents/skills submodule)
```

For every other task (decompiling, asset ripping, tools, tests) see
[`docs/modding/guides/task_scripts_reference.md`](../../../../docs/modding/guides/task_scripts_reference.md).

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
[`docs/modding/guides/mods_menu.md`](../../../../docs/modding/guides/mods_menu.md) §1.

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
[`docs/modding/guides/mod_distribution_guide.md`](../../../../docs/modding/guides/mod_distribution_guide.md)
for the full packaging and catalog pipeline.

When driving `gk` from a script or CI step, its exit codes need
interpreting rather than treated as pass/fail directly: a harness timeout
that kills a still-running game is success (the process was fine, you just
stopped waiting), a human closing the window or a normal shutdown are not
failures either — only a genuine segfault exit code is a real failure.
Don't wire a boot-smoke-test to "exit code 0 or nothing" without accounting
for this.

**The Linux job compiles with an older Clang.** `release.yml` builds Linux on
Ubuntu 22.04 with its Clang 14; Windows (`clang-cl`) and a recent local Clang
accept C++20 constructs in C++17 code as extensions, Clang 14 refuses some. A
lambda that captures a structured binding (`for (auto& [k, v] : m) { f([&] { ... v ... }); }`)
built on Windows and stopped the Linux job ("reference to local binding 'v'
declared in enclosing function"), so no release was published. Bind named
variables instead (`auto& v = entry.value();`), and before a release check
the C++ files a mod changed with the local compile command
(`out/build/Release/compile_commands.json`) plus
`-fsyntax-only -Werror=c++20-extensions`, which flags every such use. Code
under `#ifndef _WIN32` is only compiled by the Linux job: read it.

Verified: jak3 multiplayer mod, release run 41545278942 of
`whozghiar/jak3-multiplayer` (Linux job, `kmultiplayer.cpp`), the check above
on its 13 changed files with Clang 19, 2026-11-09.

---

## 7. Exposing C++ functions to GOAL (kernel functions)

A mod that needs the operating system (sockets, files, clipboard) adds C++ to the runtime and calls
it from GOAL like any function.

- **Register** it in the game's `InitMachine_PCPort` (`game/kernel/jak3/kmachine.cpp` for Jak 3)
  with `make_function_symbol_from_c("pc-my-thing", (void*)my_thing)`. This runs before GAME is
  linked, so GOAL top-level forms of GAME files may already call it. Declare it on the GOAL side
  with `define-extern` and its function type.
- **Arguments** arrive as integers: GOAL pointers are `u32` offsets into EE memory
  (`Ptr<T>(offset).c()`), a string is `Ptr<String>(offset).c()->data()` (its first `u32` is the
  allocated size: bound copies by it), a float is its bit pattern. Return a boolean as
  `s7.offset + true_symbol_offset(g_game_version)` or `s7.offset`.
- **At most four register arguments on Windows:** the win32 trampoline only remaps four registers
  (`kscheme.cpp`). Pass a structure pointer for more.
- **Shared structures:** mirror the GOAL `deftype` field for field with a `static_assert` on the
  size, and give the `deftype` a `:size-assert`. Inline vectors are 16-byte aligned.
- **Threading:** kernel functions run on the game thread. Anything that can block (a socket, a
  file on a network drive) stalls the frame: use non-blocking calls pumped once per frame.
- **Winsock:** `gk` never calls `WSAStartup` globally, and `close_socket` in
  `common/cross_sockets/XSocket.cpp` calls `WSACleanup` on every close. Own your sockets, call
  `WSAStartup` once, and include `winsock2.h` before anything that includes `windows.h`, with
  `NOMINMAX` defined (or `std::min`/`std::max` stop compiling).
- **Tests:** `goalc-test` (`test/CMakeLists.txt`) links `runtime`, so a gtest can include the new
  `game/` header and exercise the C++ without the game.

Verified: jak3, `game/kernel/jak3/kmachine.cpp`, `kscheme.cpp`, `common/cross_sockets/XSocket.cpp`;
the multiplayer mod's `game/kernel/jak3/kmultiplayer.cpp` built and unit-tested, 2026-10-14.

---

## 7b. Merc2: per-model data from GOAL, and a pass after the 3D scene

- **Per-model flags.** `pc-merc-draw-request` (`goal_src/jak3/engine/gfx/foreground/foreground.gc`)
  writes two quadwords of flags per merc model: `enable-mask`, `ignore-alpha-mask`,
  `effect-count`, `bit-flags` (`pc-merc-bits`, bits 0-4 taken), then 14 free bytes.
  `Merc2::handle_pc_model` reads them as `PcMercFlags` and skips the 32 bytes: a new bit and a
  value at byte 20 reach the renderer with no other change. `draw-control-global-effect` has no
  free bit in Jak 3; a global table the request checks (`(= dc (-> table dc i))`) selects models.
- **Drawing after the scene.** Merc draws are flushed per bucket, interleaved with the levels'
  background, so a later bucket's walls cover them. Keep the model (its `MercModel` draws,
  `ShaderMercMat` bones, `m_low_memory` camera) and draw it from
  `OpenGLRenderer::dispatch_buckets_jak3` once a late bucket is done (after `GENERIC_WARP`, before
  `DEBUG_NO_ZBUF1`). Re-upload its bones to `m_bones_buffer` (each model's at a multiple of
  `GL_UNIFORM_BUFFER_OFFSET_ALIGNMENT`) and bind the range at binding 1 (`ub_bones` is bound to 1
  for every shader by `Shader.cpp`).
- **Depth is reversed:** merc tests `GEQUAL` (greater is closer). "Hidden by something" is
  `GL_LESS`. Drawn again with the same transform, a model comes out a hair behind its first
  drawing: a `glPolygonOffset(2, 512)` toward the camera keeps its own visible parts from
  passing a "hidden" test. The framebuffer has a stencil (`DEPTH24_STENCIL8`).
- **Data after the flags.** A request can append quadwords after its 32 bytes of flags (64
  bytes of tints for a flagged model) when `handle_pc_model` skips them for that bit: nothing
  else reads that packet.
- **Per-model textures.** A texture animated for every model at once (Dark Jak's skin, the
  `darkjak` group of `jakc-*` textures, `set-darkjak-texture-morph!`) is one slot shared by every
  model of Jak. `TextureAnimator` can blend the variants itself at fixed levels (17 per texture,
  as the group's blenders do) and Merc2 bind the variant a flagged model asks for in place of the
  shared slot.
- **Not every model goes through Merc2.** A model drawn in the HUD (the START menu's models)
  goes through the generic renderer: Merc2's flags do not reach it. The PC
  `draw-bg-box-alpha-1` of the menu writes depth: a 3D model behind it is hidden, so narrow the
  box where a model must show through.
- **A new shader** goes in `ShaderId` (before `MAX_SHADERS`) and `ShaderLibrary`; its files in
  `game/graphics/opengl_renderer/shaders/` ship with the release (section 6). Uniforms are set
  after `activate()`.

Verified: jak3, `goal_src/jak3/engine/gfx/foreground/foreground.gc`,
`game/graphics/opengl_renderer/foreground/Merc2.cpp`, `OpenGLRenderer.cpp`, `Shader.cpp`; the
multiplayer mod's outline of other players through walls (`outline` scenario: 90 pixels in the
open, 920 behind a vehicle, screenshots), 2026-10-19; `TextureAnimator.cpp`, `merc2.frag`: another
player's Dark Jak skin and outfit tints (`powers`, `skin` scenarios), 2026-10-19.

## 7c. Taking the keyboard for typed text

The renderer's thread (`GLDisplay::render`, `game/graphics/pipelines/opengl.cpp`) polls the
keyboard into pad buttons every frame (`InputManager::poll_keyboard_data`, which reads
`SDL_GetKeyboardState`), or releases them with `clear_keyboard_actions` while ImGui wants the
keyboard; then `process_sdl_events` hands each SDL event to the input manager. For a line the
player types in game:

- Release the game's keys while the line is typed: call `clear_keyboard_actions` there instead of
  polling.
- Hand each SDL event to the line before the input manager, and skip the manager for the ones it
  takes (every `SDL_EVENT_KEY_DOWN` while typing).
- `SDL_StartTextInput(window)` (SDL3, the renderer's thread) turns on `SDL_EVENT_TEXT_INPUT`, its
  `text` in UTF-8; `SDL_StopTextInput` when the line closes.
- A key still held when the line closes reaches the game at the next poll (Enter is START by
  default): keep releasing the keys until it is up.
- GOAL reads the line through kernel functions (section 7) under a mutex: the game runs on
  another thread. A test pushes the same events with `SDL_PushEvent` (a `TEXT_INPUT` event's
  `text` must outlive it).
- Windows: SDL's message pump runs `TranslateMessage`, so a `WM_KEYDOWN` alone gives its
  character (`WM_CHAR`); posting a `WM_CHAR` as well types it twice. SDL sends a key event to any
  window, but text only to the window with the keyboard focus, which it takes from
  `GetForegroundWindow` (`WIN_UpdateFocus`): a posted `WM_SETFOCUS` does not give it. A test can
  post `WM_KEYDOWN` and `WM_KEYUP` to its own window (`SDL_PROP_WINDOW_WIN32_HWND_POINTER`) and
  type, while that window is the foreground one.

Verified: jak3, `game/graphics/pipelines/opengl.cpp`, `game/system/hid/input_manager.cpp`,
`devices/keyboard.cpp`; the multiplayer mod's chat (`kmultiplayer.cpp` "Chat", `chat` scenario:
T, text and Enter through SDL's queue, then as Windows key messages on the focused window;
`third-party/SDL/src/video/windows/SDL_windowsevents.c`, `src/events/SDL_keyboard.c`),
2026-10-29.

## 8. Automated in-game checks

Moved to the [`smoke-tests`](../../smoke-tests/SKILL.md) skill: running scenarios in the game,
and its [game control reference](../../smoke-tests/game-control.md) (separate instances, pad
input, screenshots, crash reports, retail boots).

---

## See also

- [`docs/modding/guides/task_scripts_reference.md`](../../../../docs/modding/guides/task_scripts_reference.md) — every task, with concrete examples.
- [`docs/modding/guides/task_scripts_reference.md`, section 3](../../../../docs/modding/guides/task_scripts_reference.md#3-c-build--compilation-tasks) — the 3-layer build model and sccache.
- [`repl-workflow.md`](repl-workflow.md) — REPL lifecycle, heap architecture, and the "ghost memory" hot-reload pitfall.
- [`concepts.md`](concepts.md) — the language this engine compiles and runs.
