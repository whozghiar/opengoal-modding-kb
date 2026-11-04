# Taking control of the game for tests

The reference behind the [`smoke-tests`](SKILL.md) skill: how a script or a scenario drives the
game without anyone at the controller. Verified facts only, each with its evidence.

## The scenario runner of master-dev

`scripts/modding/smoke_test.py` is the GOAL-only route below, packaged:

- **install** copies a scenario file to `goal_src/<game>/pc/features/smoke-test.gc` and lists it
  last in GAME (`dgos/game.gd`; Jak 1 also lists GAME sources in `game.gp`), so the mod's GAME
  code is visible to it. Its top-level `process-spawn-function` starts the scenario when GAME
  links, in a debug or a retail boot.
- **run** starts `gk -v --game <game> -- -boot -fakeiso [-debug]` from `out/smoke/<name>/`: the
  runtime writes its `log/` folder in its working folder, and a hung earlier runtime keeping its
  log file open made a new one in the same folder fail to start. The output is read on a thread,
  so a runtime that hangs silently still reaches the timeout and is killed.
- **Log lines.** `format`'s `~A` prints a string with its quotes, `~S` without
  (`game/kernel/jak3/kprint.cpp`): print the scenario's tags and results with `~S`.
- **Screenshots** land about 0.5 s after `pc-screen-shot`: wait before the scene changes.

Verified: jak3, Haven City: New Dawn's round 2 scenarios (the same steps, its own scripts),
2026-11-04; the runner and the template compiled on jak2, 2026-11-04.

## Driving the game

When the user allows the game to be launched for tests, several instances can run side by side,
each driven by a script, without anyone at the controller:

- **Separate instances:** `gk --config-path <dir> ...` makes `<dir>/OpenGOAL/<game>/` the settings,
  saves and screenshots folder of that instance. Copy the user's `saves/` and `settings/` into it;
  `display-settings.json` (`display_mode` 0 = windowed, `window_xpos`) and the `(window-size w h)`
  line of `pc-settings.gc` place each window. A second debug instance only warns that the REPL
  port is taken.
- **Into the game without input:** `-boot -fakeiso -debug` loads save slot 1.
- **Screenshots from C++:** set `g_screen_shot_settings->name` (and its size), then
  `g_want_screenshot = true` (`game/graphics/screenshot.h`): the renderer saves the next frame as
  `<config>/OpenGOAL/<game>/screenshots/<name>.png`. GOAL's `capture-pc.gc` is debug-only.
- **Input:** `scePadRead` (`game/sce/libpad.cpp`) fills each pad from the input manager; OR extra
  bits into `cpad->button0` (PS2 order, select = bit 0, square = bit 15) or set `leftx`/`lefty`
  there to drive the player. Set the pressure too (`cpad->abutton[i]`, order right, left, up,
  down, triangle, circle, x, square, l1, r1, l2, r2): Jak 3's buggies take their throttle from X's
  pressure alone (`wvehicle.gc`), so a bare button bit does not drive them.
- **Commands:** an environment variable naming a script file read by a kernel function, polled by
  GOAL every frame, keeps everything in-process; `(format 0 ...)` lines land in the instance's
  output for the runner to collect.

The multiplayer mod's `scripts/mp-test/run_mp_test.py` and `multiplayer-test.gc` implement this.

- **Which GOAL function crashed.** An access violation in GOAL code ends `gk` with exit code
  `0xC0000005` and nothing in its output. A vectored exception handler
  (`AddVectoredExceptionHandler`, Windows) sees the faulting `Rip`: inside
  `[g_ee_main_mem, g_ee_main_mem + EE_MAIN_MEM_SIZE)` it is GOAL code, at EE offset
  `rip - g_ee_main_mem`. The function holding it is the one whose code starts last before it:
  scan every symbol value (from `SymbolTable2` to `LastSymbol`) that is a function object
  (`(v & 7) == 4` and the word at `v - 4` is the `function` type,
  `u32_in_fixed_sym(FIX_SYM_FUNCTION_TYPE)`), and every method of every type object, keeping the
  greatest start at or below the offset. GOAL calls run on the native stack: the qwords near
  `Rsp` that point into EE memory are its callers (and stale return addresses). Print, `fflush`,
  return `EXCEPTION_CONTINUE_SEARCH`. A jump to EE offset 0 is a call through an unset function.
  The multiplayer mod's `report_goal_crash` (`game/kernel/jak3/kmultiplayer.cpp`) printed
  `vector<-cspace! +#xc` from `board-zap-track`, the first clue to a crash no log showed.
- **The instruction behind `f +#x...`.** Anonymous code (state handlers, lambdas) has no symbol: it
  is named after the last named function before it (`valid? +#x8b5`). goalc disassembles a file
  with its source lines and IR: `goalc --user-auto --game jak3 --cmd "(begin (mi) (asm-file
  \"goal_src/jak3/<path>.gc\" :color :disassemble \"<out>.asm\"))"` (without `:color` it only
  warns). Each function is listed as `[name]`, its code from `0x10000`: `f +#x955` is the line
  `[0x10955]` under `[f]`. A caller's offset is a return address: its call is the instruction
  before.

Verified: jak3, `game/main.cpp`, `common/util/FileUtil.cpp`, `game/graphics/pipelines/opengl.cpp`,
`game/kernel/jak3/kmultiplayer.cpp`; used for every in-game check of the multiplayer mod,
2026-10-15.

**Without C++: a scenario compiled into GAME.** A mod that adds no kernel function can be checked
the same way from GOAL alone:

- **Pad.** `cpad-get-data` is a kernel function that `service-cpads` (`engine/ps2/pad.gc`) calls
  through its symbol: keep the stock one in a `define-perm`, `set!` the symbol to a function that
  calls it, then, for pad 0, sets `valid` 0 and `status` `#x73`, ORs bits into `button0` and writes
  the sticks.
- **Screenshots in a retail boot.** `capture-pc.gc` is debug only. Register a structure of the same
  layout (`width`, `height`, `msaa` as int32, then a 244-byte `name`) with
  `pc-register-screen-shot-settings`, write the name, call `pc-screen-shot`.
- **The REPL and the boot modes.** `-boot` closes the REPL port (`sceDeci2Disable` when
  `MasterDebug` is 0, `game/kernel/jak3/kmachine.cpp`); `-boot -debug-mem` keeps the retail code
  path (no debug segment) and opens it. For a check where the REPL takes no part, list a scenario
  file in `game.gd` (out of any commit) and boot `-boot -fakeiso`: the scenario starts itself, for
  example from the pad hook on its first call, and writes its results with `(format 0 ...)`.
- **From a retail boot into a save.** The first boot plays the opening scenes: triangle skips one
  (twice), START opens the title menu, X on "Load Game" (already selected), X on slot 1.
- **Running under menus.** A process that must go on under the menus clears `menu`, `pause`,
  `progress` and `freeze` from its mask: the title's progress menu stopped one that only cleared
  `menu` and `pause`.
- **`*pc-dead-pool*` holds 4 processes** of 128 KB (`pc/pckernel.gc`); the subtitles, the Mods menu
  popup and a mod's own process take 3. Take test processes from `*default-dead-pool*`.
- **Symbol names in the log.** In a retail boot the printed name of a symbol (`~A`) can come out
  garbled after level loads while its value is right: the kernel keeps some names above
  `kglobalheap->top_base`, the area DGO loading reuses (`game/kernel/jak3/kscheme.cpp`). Compare
  symbols, never their printed names.
- **"#<#... has invalid type ptr #x...>".** The kernel prints an object through
  `call_method_of_type` (`game/kernel/jak3/kscheme.cpp`), which calls any type above `0x7ffffff`
  invalid and prints that line, then "[ERROR] call_method_of_type failed!". Jak 3 PC level heaps
  can sit above 128 MB: a live unit of a level type (seen: Haven City robots at `#xa7b3324`) prints
  this way. It is a printing limit, not a corrupted object.

Verified: jak3, `engine/ps2/pad.gc`, `pc/debug/capture-pc.gc`, `game/kernel/jak3/kmachine.cpp`,
`kscheme.cpp`, `pc/pckernel.gc`; the battlefront mod's scenario runs (retail boot, every class
tested, screenshots), 2026-10-31.

