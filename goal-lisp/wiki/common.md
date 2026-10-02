# Lisp wiki — common to all games

GOAL patterns, the engine model and the pitfalls shared by Jak 1, Jak 2 and Jak 3. Index: [index.md](index.md).

## 1.1 Vocabulary

| Term | Meaning |
|---|---|
| **GOAL** | Naughty Dog's Lisp dialect, compiled to native x86-64 by OpenGOAL. All game logic (`goal_src/**/*.gc`). |
| **`goalc`** | The OpenGOAL compiler. REPL (`task repl`) or batch (`-c "(...)"`). |
| **`gk`** | The C++ runtime ("game kernel") that runs the compiled code. |
| **`(mi)`** | REPL command: incremental compile + hot-reload the running game. |
| **process** | Lightweight cooperative task with its own stack/heap. |
| **`process-drawable`** | A process that also has a 3D model + transform. Jak 1's `target` (the player) derives directly from it; Jak 2/3/Jak X derive `target` from `process-focusable` instead (see [2.1](jak1.md#21-vocabulary-delta)). |
| **`process-focusable`** | Common base for actors the camera/AI can target — the default actor base in Jak 2 and Jak 3. |
| **`*target*`** | The global symbol pointing at the player process (Jak). |
| **`self` / `pp`** | Inside a `behavior`: `self` = the process; `(with-pp ...)` binds `pp` = the process pointer. |
| **state** | A named node of a process's state machine: `:event`, `:code`, `:post`. |
| **`ja`** | Family of macros that drive skeletal animation ("joint animation"). |
| **art-group** | A named bundle of a model's skeleton + geometry + animations. |
| **DGO** | On-disk package of compiled objects loaded as one unit. |

## 1.2 Core Lisp patterns

### 1.2.1 Define a custom actor type

`deftype` declares a new type. For something that moves and is drawn,
derive from `process-drawable` (Jak 1) or `process-focusable` (Jak 2/3 —
also valid on Jak 1, just not what `target` itself uses there). Extra
fields go in the first list; `:state-methods` pre-declares the states this
type can be in.

```lisp
;; a minimal custom actor with two fields and two states
(deftype my-actor (process-drawable)
  ((hp        int32)
   (wake-time time-frame))
  (:state-methods
    idle
    active))
```

Trap: the parent type must already be known — its file must be earlier in
the `.gp` include list. See [1.4.1](#141-symbol-and-load-order).

### 1.2.2 Define a state

`defstate` fills one state of a type. `:event` handles messages sent to
the process; `:code` is the behavior loop; `:post` runs every frame after
`:code` (usually `ja-post` to flush animation). Add `:virtual #t` when the
state overrides a parent's state and must dispatch through the vtable.

```lisp
(defstate idle (my-actor)
  :virtual #t
  :event (behavior ((proc process) (argc int) (message symbol) (block event-message-block))
    (case message
      (('touch 'attack)
       (go-virtual active))))
  :code (behavior ()
    (loop
      ;; my-actor-idle-ja = the animation symbol from this actor's art-group
      (ja-no-eval :group! my-actor-idle-ja :num! (seek!) :frame-num 0.0)
      (until (ja-done? 0)
        (suspend)
        (ja :num! (seek!)))))
  :post ja-post)
```

`(go active)` vs `(go-virtual active)`: `go` jumps to the state named in
*this* type; `go-virtual` dispatches through the vtable so a subclass's
override wins. Use `go-virtual` for states declared in `:state-methods`.

Residency trap: if this actor is spawned by an always-resident system,
every `:virtual #t` state must live in a resident file — see
[1.4.7](#147-virtual-methodstate-residency-the-vtable-trap).

### 1.2.3 Drive animation: the `ja` macros

A skeleton is a tree of joints; an animation is a stream of joint poses
per frame. You do not move joints by hand — you tell a **channel**
(usually channel 0) which animation to play and how fast to advance it,
once per frame, then let `ja-post` push the result to the renderer.

| Macro | What it does |
|---|---|
| `(ja-no-eval :group! G :num! (seek! MAX) :frame-num F)` | Start channel 0 on animation `G` at frame `F`; set its target advance. |
| `(ja :num! (seek!))` | Advance the current animation one frame toward the target. |
| `(ja-done? 0)` | `#t` when channel 0 reached the end of its clip. |
| `(ja-aframe N 0)` | Read/seek to a specific frame number on channel 0. |
| `(suspend)` | Yield to the engine for this frame (like `yield`). |
| `ja-post` | State `:post` helper: flush the pose + collision. |

```lisp
;; play an animation once, from start to end, then continue
(ja-no-eval :group! my-anim-ja :num! (seek!) :frame-num 0.0)
(until (ja-done? 0)
  (suspend)                ;; give the frame back to the engine
  (ja :num! (seek!)))      ;; step the animation forward
```

Trap: forgetting `(suspend)` inside an animation loop freezes the game —
the process never yields.

### 1.2.4 Skeletons & joints

The joint subsystem is `cspace` / `joint-control`
(`goal_src/<game>/engine/anim/joint.gc`). A process's live joint
transforms are reachable through its `node-list`. You rarely need direct
joint access — prefer animations ([1.2.3](#123-drive-animation-the-ja-macros))
and `joint-mod` helpers — but it exists.

```lisp
;; number of joints on a live process (useful to compare two skeletons)
(-> self node-list length)

;; a joint's world transform matrix, by index
(-> self node-list data 5 bone transform)
```

Trap: joint **indices** are skeleton-specific. Code that pokes joint `5`
of `skel-jchar` will poke a *different* body part on another skeleton. See
[3.4](jak2.md#34-live-re-skinning-a-process).

### 1.2.5 Bind a model to a process: `initialize-skeleton`

`initialize-skeleton` is what a `process-drawable` calls once at init to
bind its mesh + skeleton + animation set. It takes a resolved
`skeleton-group`. `initialize-skeleton-by-name` takes a plain string and
does the lookup for you. It is not restricted to first-time init: calling
it again on a live process rebinds its look in place (a "live re-skin") —
the process keeps its identity, position and handlers.

```lisp
;; standard init — resolve the art-group from the current level, then bind
(initialize-skeleton
  this
  (the-as skeleton-group (art-group-get-by-name *level* "skel-my-actor" (the-as (pointer uint32) #f)))
  (the-as pair 0))

;; shorthand when you only have the name
(initialize-skeleton-by-name this "skel-my-actor")
```

Multi-DGO trap: the lookup uses `(-> this level)`. If a parent in level A
spawns a child whose art lives in level B, set `(-> this level)` **and**
`(-> pp level)` to level B *before* calling `initialize-skeleton`, or the
child crashes into `:state process-drawable-art-error "art-group"`.

```lisp
;; fix for a child whose art-group is in another level DGO
(when (= (level-status *level* 'lwidea) 'active)
  (set! (-> this level) (level-get *level* 'lwidea))
  (set! (-> pp   level) (level-get *level* 'lwidea)))
(initialize-skeleton this <skeleton-group> (the-as pair 0))
```

### 1.2.6 Play a sound

`sound-play` is the simple form. `sound-play-by-name` gives control over
volume, pitch, position and group.

```lisp
;; fire-and-forget SFX by name
(sound-play "menu-select")

;; full control (volume 1024 = nominal, group sfx, positional flag #t)
(sound-play-by-name
  (static-sound-name "my-sound")
  (new-sound-id)
  1024 0 0
  (sound-group sfx)
  #t)
```

Trap: the sound name must exist in a loaded sound bank. For looped or
frame-updated sounds, pre-allocate the sound ID once in the actor's
`-init` state via `(new-sound-id)`, never inside the frame loop.

### 1.2.7 Collision basics

A `process-drawable`'s collision lives in its `root` field (a
`collide-shape` or `trsqv`). Surface behavior (slippery, deadly, etc.)
comes from `pat-surface` data (`engine/collide/pat-h.gc`). To make a
corpse stop blocking movement, clear its collide specs.

```lisp
;; disable an actor's collision (e.g. on death)
(let ((prim (-> self root root-prim)))
  (set! (-> prim prim-core collide-as)   (collide-spec))
  (set! (-> prim prim-core collide-with) (collide-spec)))
```

### 1.2.8 Custom art-groups & dynamic animation linking (`link-art!`)

Available in **Jak 1 and Jak 2 only** — confirmed absent from Jak 3's
engine code (no `register-custom-art-group` / `link-art!` / `needs-link?`
anywhere in `goal_src/jak3/`; see [4.5](jak3.md#45-custom-art-groups-no-dynamic-linking-hook)
for the Jak 3 alternative).

To add animations imported from a `.glb` into a resident character
art-group (`jakb-ag`, `daxter-ag`) *without* recompiling the hundreds of
native animations: `build-actor` (in `game.gp`) bakes target slot indices
with `:master-art-group` / `:master-ag-map`; `link-art!` (`loader.gc`)
attaches the custom group's entries into those slots.

```lisp
;; hook link-art! in art-group::relocate (engine/anim/joint.gc) — NOT in gameplay
(when (or (not s5-1) (= (-> s5-1 name) 'default))
  (login this)
  (if (or (needs-link? this)
          (string= (-> this name) "jakb-my-import"))
      (link-art! this)))
```

Registering the art-group name at file load makes the check above see it:

```lisp
;; register your art-group name (without "-ag") into *custom-art-groups-to-link*
(register-custom-art-group "jakb-my-import")
```

Trap: never call `link-art!` during gameplay (e.g. from a state `init`):
the level art-group arrays are not in a stable state and you risk a
memory crash. The safe hook is `art-group::relocate`.

### 1.2.9 Send an event

`send-event` delivers a message to another process's current `:event`
handler. It uses a stack message block — no heap allocation — so it is
safe to call at any time, including while the player crosses a
level/district boundary.

```lisp
;; ask a manager process to do something; guard it, it may not exist yet
(when *some-manager*
  (send-event *some-manager* 'some-message some-argument))
```

Trap: guard managers that may not exist. A global manager symbol can be
`#f` outside the context where it's relevant — always check before
sending.

### 1.2.10 Generic enemy death effect (`do-effect`)

Many enemies across the trilogy dissolve into particles tracing the mesh,
with a "fizz" sound. This is a reusable engine system
(`goal_src/<game>/engine/gfx/merc/merc-death.gc` and friends), not a
per-bone emitter. Trigger it from any skeleton-having `process-drawable`'s
death code with one call. The `effect-control` at `(-> self skel effect)`
is created for you inside `initialize-skeleton`.

```lisp
;; canonical death code (pattern from wasp.gc die-now state)
:code (behavior ()
  (dying self)                                            ;; plays enemy sound-die + drops gems
  (let ((prim (-> self root root-prim)))                  ;; stop the corpse blocking things
    (set! (-> prim prim-core collide-as)   (collide-spec))
    (set! (-> prim prim-core collide-with) (collide-spec)))
  (set! (-> self hit-points) 0)
  (do-effect (-> self skel effect) 'death-default 0.0 -1) ;; spawn the purple dissolve + "enemy-fizz"
  (suspend-for (seconds 1))                               ;; MUST wait — particles spawn while alive
  (send-event self 'death-end)
  (cleanup-for-death self))
```

Trap: do **not** call `cleanup-for-death` right after `do-effect` — the
particle spawn is driven per-frame while the process is still drawing;
killing it immediately makes the entity vanish silently. Also: needs a
skeleton. Presets: `death-default` (purple, generic kill), `death-seed`
(orange, life-seed scene), `death-warp-in`/`out` (blue-purple, warp gate —
not a kill).

### 1.2.11 Register an in-game Mods toggle

Available in **Jak 2 and Jak 3 only** — Jak 1 has no port yet, see
[2.4](jak1.md#24-in-game-mods-toggle-not-ported) for the workaround. Every mod
must expose an on/off switch in the in-game **Mods** menu, opened with
**L3 + SELECT**. That menu is not the debug menu: it works in a retail
boot, which is how the launcher starts the game. This is a project rule,
not an optional nicety — it is what lets a player enable or disable a
mod's changes without anyone touching vanilla files, while the mod itself
touches the minimum possible amount of shared code. You never edit
`default-menu*.gc`, and your menu file must never carry
`(declare-file (debug))` — a DEBUG segment is not linked in a retail boot,
so the registration would silently never run. You call
`mods-menu-register` from one of your mod's own compiled `.gc` files.

```lisp
;; NO (declare-file (debug)) here -- it would strip the file in a retail boot

;; one prefixed symbol per option
(define *mod-my-slug-enable* #f)

;; the menu, fully static. Nested :entries and the lambdas MUST be inline.
(define *mod-my-slug-menu*
  (new 'static 'popup-menu-submenu :label "my-slug"
    :entries (new 'static 'boxed-array :type popup-menu-entry
      (new 'static 'popup-menu-flag :label "Enable"
           :is-toggled? (lambda () *mod-my-slug-enable*)
           :on-confirm (lambda ()
                         (set! *mod-my-slug-enable* (not *mod-my-slug-enable*))
                         (none))))))

;; builder — pure, no argument, returns this mod's entry
(defun mod-my-slug-build-menu ()
  (the-as popup-menu-entry *mod-my-slug-menu*))

;; register at file load (top-level)
(mods-menu-register "my-slug" mod-my-slug-build-menu)
```

Prefix every symbol with your mod slug (`*mod-<slug>-*`, `mod-<slug>-*`)
to avoid collisions between mods. Add `"my-slug-menu.o"` to your
`.gd` **after** `"mods-menu.o"`. Full architecture (controls, entry types,
pitfalls, the one real Jak 2/Jak 3 difference) is in
[`guides/mods_menu.md`](../../../../docs/modding/guides/mods_menu.md); copy-paste starting point:
[`templates/mod_menu.template.gc`](../../../../docs/modding/templates/mod_menu.template.gc).

### 1.2.12 Static props, custom levels, and audio banks

Three more building blocks, confirmed present in all three games
(`goal_src/jak[x]/engine/data/art-h.gc` for `def-actor`; the custom-level
macros are referenced from each game's own debug menu warp command).

**A lightweight static prop** (a pickup, a marker, anything with no
skeleton) skips the full character pipeline entirely. `build-actor`'s
tooling auto-synthesizes a trivial `align`/`prejoint`/`main` joint chain
and a single-pose idle animation for any `.glb` with no skin at all — the
same fallback vanilla's own simple static props use. Registering the
resulting art-group by hand needs three `def-art-elt` lines plus a
`defskelgroup` block; `def-actor` is a one-call shortcut that expands to
exactly that boilerplate:

```lisp
;; goal_src/jak2/levels/test-zone/test-zone-obs.gc
(def-actor test-actor
  :bounds (0 0 0 5))    ;; sphere: x y z radius (metres)
```

Defaults to `:art (idle-ja)` and a single LOD; pass `:idle`, `:art`,
`:lods`, `:texture-level` etc. to override. Reach for the manual
`def-art-elt` x 3 + `defskelgroup` recipe only when you need multiple
LODs, several named animations, or non-default naming.

**A custom level** is declared in `game.gp` and warped to from the REPL:

```lisp
;; goal_src/jak[x]/game.gp
(build-custom-level "my-level")
(custom-level-cgo "MYLEVEL.DGO" "my-level/mylevel.gd")
```

```lisp
;; from the REPL, after (mi) and with the game connected via (lt)
(bg-custom 'my-level-vis)
```

See the `custom-actors-levels` skill for the full directory layout
(`custom_assets/jak[x]/levels/<name>/`) and `level-info.gc` registration
steps this depends on. Working example to copy from:
`custom_assets/jak1/levels/test-zone/`.

**Audio banks** — extending or adding a sound bank is two GOAL calls, both
declared in `game.gp`:

```lisp
;; Route A — append into a resident bank (recommended for global SFX; play
;; immediately anywhere with (sound-play "my-sound"))
(append-sbk "COMMON" "custom_assets/jak2/sounds/sfx/MY_SFX" :force-run #t)

;; Route B — a standalone bank, loaded on demand into the 3-slot rotating pool
(build-sbk "MYBANK" "custom_assets/jak2/sounds/sfx/MYBANK" :force-run #t :bank-id #x6d79736e)
;; then, before use:
(sound-bank-load (static-sound-name "MYBANK"))
```

Route A needs `"COMMON"` removed from `copy-sbk-files` in `game.gp` first
(avoids a duplicate-output build error). Route B's rotating pool can
conflict with level sound banks if too many are loaded at once — prefer
Route A unless the bank is genuinely level-scoped. Source audio (16-bit
PCM WAV + a `metadata.txt` manifest) and the full extraction pipeline —
including the standalone `extract_sbk` tool for pulling a handful of
sounds out of another game without a full decompiler pass — are
documented in the `custom-actors-levels` skill.

## 1.3 Engine model

The plain-language model of how the engine works below the Lisp syntax:
memory, heaps, level streaming, process life cycle, boot diagnostics. Read
it once before your first mod; come back to it whenever a crash mentions
memory or an object "not found".

### The simulated PS2 memory block

The original games ran on the PS2's "Emotion Engine" (EE) CPU with a
fixed block of RAM. The PC port keeps that model: at start-up the C++
runtime (`gk`) reserves one big contiguous virtual memory block with
`mmap` and lets the GOAL code allocate inside it at fixed offsets, exactly
as the PS2 kernel did. Nothing in GOAL uses the operating system's
`malloc` directly — everything lives inside this one block. A mod that
allocates too much, or writes past a heap, does not corrupt your PC — it
corrupts this block and the game crashes with a memory error.

### The three heaps

A "heap" is a labelled region inside the big block where GOAL allocates
objects:

| Heap | Holds | Lifetime |
|---|---|---|
| **global heap** | types, the symbol table, kernel code, always-resident processes, `*target*` | whole session |
| **level heap** | everything a level needs: its geometry, textures, actors, art | freed when the level unloads |
| **debug heap** | debug menu, REPL helpers, profiling tools | whole session, but only in `-debug` builds |

The heap an allocation lands on is chosen by the first argument to `new`
(`'global`, `'process`, `'debug`, and so on — see
[1.4.8](#148-memory-heap-selection) for the exact call shape and the
traps). Never allocate gameplay objects on `'global` "to be safe" — it is
small and never frees. Debug-only code must allocate on `'debug` so it
disappears cleanly from release builds.

### Memory constants: where they live

Some memory limits are set in shared C++ (one value for all four games —
Jak 1, 2, 3 and Jak X); changing one silently changes it for every game.
Others are set per game in GOAL, in each game's own section of this
document.

| Constant | File | Scope |
|---|---|---|
| `EE_MAIN_MEM_SIZE` | `common/goal_constants.h` | shared (all games) |
| `GLOBAL_HEAP_END` | `game/kernel/common/memory_layout.h` | shared |
| `DEBUG_HEAP_START` | `game/kernel/common/memory_layout.h` | shared |
| `END_OF_MEMORY` (used by `valid?`) | `goal_src/<game>/kernel/gcommon.gc` | per game |
| `DEBUG_LEVEL_HEAP_MULT` (level-heap size multiplier) | `goal_src/<game>/engine/level/level.gc` | per game |

`valid?` (in `gcommon.gc`) is the engine's pointer sanity check: it
rejects any address `>= END_OF_MEMORY`. The PS2 limit was 128 MB
(`#x8000000`). The PC port can reserve much more, but if `END_OF_MEMORY`
is left at the PS2 value while the block is bigger, every object
allocated above 128 MB fails with `bad address` / `not a valid object`.
Always validate a memory change at runtime — it can compile clean and
still panic at boot.

Where each game stands on `master-dev`:

| Metric | Jak 1 | Jak 2 | Jak 3 |
|---|---|---|---|
| `EE_MAIN_MEM_SIZE` (shared) | 512 MB | 512 MB | 512 MB |
| `GLOBAL_HEAP_END` (shared) | `0x12D00000` (~300 MB) | same | same |
| `END_OF_MEMORY` | `#x20000000` | `#x20000000` | `#x20000000` |
| level-heap tuning | `LEVEL_HEAP_SIZE_DEBUG`, no multiplier scheme | `DEBUG_LEVEL_HEAP_MULT 12.0` | `DEBUG_LEVEL_HEAP_MULT 15.0` |

Jak 3 is already at the ceiling — 15.0 is the highest tested multiplier of
the three. Jak 1 has a different level-heap architecture entirely: no
page/multiplier scheme, just `LEVEL_HEAP_SIZE_DEBUG` allocated from the
global heap. The exact `defconstant` snippets live in each game's
"Memory" section ([2.2](jak1.md#22-memory), [3.8](jak2.md#38-memory),
[4.6](jak3.md#46-memory)). Raising the *shared* `DEBUG_HEAP_START` once broke
Jak 1 outright, because its `InitMachine` computed the debug heap end from
a 128 MB address mask — always boot all three games with `-debug` when
touching shared C++ memory constants.

### DGOs and level streaming

Compiled GOAL code and data are packed into DGO files ("Data Group
Object"). A DGO is loaded as one unit:

- **Resident DGOs** (`KERNEL`, `GAME`, `ENGINE`, city hub DGOs) — loaded
  once at boot and never freed. Code here is always available.
- **Level DGOs** — streamed in when entering a level/area and freed when
  leaving. Code and assets here exist only while that level is loaded.

Which `.o` files go into which DGO is declared in the game's `.gd` files
(`goal_src/<game>/dgos/*.gd`); how they are compiled is declared in the
project file `goal_src/<game>/game.gp`. If your code is spawned by an
always-running system (traffic, a global manager) but *defined* in a level
DGO, it will be missing or dangling the moment that level is not loaded.

#### Registering a new source file

A brand-new `.gc` file (one the decompiler never produced) needs both: its
`.o` in a `.gd` list, and a compile step in `game.gp`. The compile step
differs by game:

- **Jak 1:** `game.gp` lists every source explicitly with `goal-src` and
  `goal-src-sequence`. Add the file there, after the files it depends on.
- **Jak 2 and Jak 3:** `game.gp` compiles most code through `cgo-file`, which
  walks a `.gd` list and looks each `.o` up in the decompiler's file index.
  A new file is not in that index, so pre-mark it in `*file-entry-map*`
  **before** the `cgo-file` line that reads its `.gd` (then `cgo-file` skips
  it), and compile it with an explicit `goal-src` step whose dependencies
  pin the compile order:

```lisp
;; goal_src/jak2/game.gp, before (cgo-file "game.gd" ...)
(hash-table-set! *file-entry-map* "my-mod-h.o" #f)
(hash-table-set! *file-entry-map* "my-mod-menu.o" #f)

;; further down, with the other build steps: path relative to goal_src/jak2/,
;; then the objects (no extension) to compile first
(goal-src "pc/mods/my-mod-h.gc" "settings")
(goal-src "pc/features/my-mod-menu.gc" "my-mod-h" "mods-menu")
```

In the `.gd` list itself, a Mods-menu file goes after `"mods-menu.o"`.

Verified: jak2 `cgo-file` in `goal_src/jak2/lib/project-lib.gp` and the
haven-city-chaos mod's `game.gp` (built and played); jak3 uses the same
`cgo-file` (`goal_src/jak3/lib/project-lib.gp`); jak1 `goal-src` lists in
`goal_src/jak1/game.gp`. 2026-10-02.

### Virtual method / state residency, conceptually

`defmethod` and a state declared `:virtual #t` fill a slot in the type's
virtual table (vtable) when the object file is **linked into memory** —
i.e. when its DGO loads — not at compile time. If a type is instantiated
by an always-resident system, every `:virtual #t` state and method must
live in an always-resident file, or the vtable slot is empty (silent
no-op) or dangling (crash), depending on whether the defining level was
ever loaded. The worked example and code fix pattern are in
[1.4.7](#147-virtual-methodstate-residency-the-vtable-trap); Jak 2's most
common real-world trigger for this is documented in
[3.3](jak2.md#33-virtual-method--state-residency-in-practice).

### Process life cycle

Almost every live thing in the game is a process (a lightweight
cooperative "thread" with its own small stack and heap). A
`process-drawable` is a process that also has a 3D model:

1. **Spawn** — allocates the process on a pool's heap and runs its `init`
   code.
2. **Run** — each frame the engine resumes the process's current state;
   `:code` runs until it yields. `:post` runs every frame after `:code`.
3. **Transition** — switching to a static or virtual state changes what
   the process does next.
4. **Death** — the process is deactivated; its heap is reclaimed.
   Children die with their parent.

A freshly spawned process has no active state until the kernel dispatches
it on the frame *after* the spawn — see
[1.4.14](#1414-process-spawn-does-not-install-state-until-the-next-frame).

### Boot diagnostics & the compile/validate loop

A GOAL-only change needs no C++ build:

1. Pick the game once: `task set-game-jak2` (or `jak1`/`jak3`).
2. Open the compiler REPL: `task repl`, then inside it, incrementally
   compile and hot-reload.
3. Boot and check the log: `task boot-game`, then scan the newest
   `log/<game>.<timestamp>.log` for memory or "not a valid object"
   errors.

### Sound bank architecture

`.SBK` banks are loaded into simulated PS2 SPU RAM by Overlord, the audio
engine. Jak 2 and 3 provide fixed dedicated slots for resident banks
(`common`, `gun`, `board`) plus a 3-slot rotating pool shared dynamically
by active level banks. The exact Lisp calls are in
[1.2.12](#1212-static-props-custom-levels-and-audio-banks); the full
audio pipeline (source WAV placement, `metadata.txt` format, extraction
tooling) is documented in the `custom-actors-levels` skill.

## 1.4 Known pitfalls

### 1.4.1 Symbol and load order

Parent types must be declared before child types — order files correctly
in the `.gp` include list. A forward reference to an undeclared type is a
compile error; a forward reference to an unloaded *virtual* slot is a
silent runtime no-op (see [1.2.2](#122-define-a-state)).

### 1.4.2 REPL ghost memory

After large changes, do a cold restart of the REPL and a fresh boot. Hot
reload (`(mi)`) keeps stale state; a change can appear to work only
because the old code is still resident.

### 1.4.3 A clean compile is not a validation

Always `(mi)` -> `task boot-game` -> check `log/<game>.*.log` for
`bad address` / `not a valid object` / `unable to malloc` before
declaring done.

### 1.4.4 Native non-regression

A mod must not change default game behavior unless its spec explicitly
requires it. Ship changes off by default, gated behind the mod's in-game
Mods toggle ([1.2.11](#1211-register-an-in-game-mods-toggle)).

### 1.4.5 Floating point equality

Never compare floats with `(= f1 f2)`; small precision differences fail
the comparison and lock up a state waiting for an exact match that never
happens. Use an epsilon or a unit-scaled bound instead:

```lisp
(< (abs (- (-> self speed) target-speed)) (meters 0.01))
```

### 1.4.6 `basic` vs `structure` allocation

A `basic` always has a runtime type tag at its base and can be passed and
checked generically (`type-type?`). A `structure` has no runtime type
tag — the compiler must know its exact type at compile time. Passing a
bare `structure` somewhere expecting a `basic`, or vice versa, is a type
error the compiler catches, but the distinction also explains why some
helpers only accept one or the other.

### 1.4.7 Virtual method/state residency (the vtable trap)

`defmethod` and `(defstate ... :virtual #t)` fill a slot in the type's
virtual table (vtable). That slot is filled when the object file is
linked into memory — i.e. when its DGO loads — not at compile time.

- Override compiled in a resident DGO -> slot always filled -> works.
- Override compiled in a level DGO that is not loaded -> slot empty -> the
  call silently falls back or does nothing.
- Override in a level DGO that was loaded then unloaded -> slot dangling
  -> crash or undefined behavior.

Rule: if a type is instantiated by an always-resident system, define
**all** its `:virtual #t` states and virtual methods in an
always-resident file.

```lisp
;; BAD — custom traffic actor's active state defined in a mission-only file.
;; GOOD — define it in the resident vehicle/enemy file (e.g. vehicle.gc, car.gc).
(defstate active (my-traffic-actor)
  :virtual #t
  :code (behavior () (loop (ja :num! (seek!)) (suspend))))
```

### 1.4.8 Memory heap selection

Allocating transient actor data on `'global` leaks memory permanently
across level loads — the global heap never frees. Use the process heap
(`'process`) or level heap (`'level`) for gameplay instances; debug-only
allocations must use `'debug` so they disappear cleanly from release
builds.

```lisp
;; allocate a short-lived actor on the process/level heap (normal case)
(process-spawn my-actor :to *entity-pool*)

;; allocate menu/diagnostic data on the debug heap so release builds drop it
(new 'debug 'debug-menu *debug-menu-context* "My menu")
```

`kmalloc` on a NULL heap (a `'debug` allocation reached from a retail
boot, where the debug heap is NULL) does not fail — it silently falls
back to the global heap. A per-frame `(new 'debug ...)` in a path retail
can reach is therefore a permanent leak, not a crash. Use `'stack` for
scratch data inside a function body instead of `'debug` unless the code
is truly debug-only.

### 1.4.9 Untyped stack arrays

`(new 'stack-no-clear 'array 'sometype N)` produces an **untyped** stack
blob with no `data`/`length` field. Passing it somewhere expecting a real
array type-errors or reads garbage. Use a boxed array instead:

```lisp
(the-as (array sometype) (new 'stack 'boxed-array sometype N))
```

### 1.4.10 `matrix` row access is not guaranteed

Do not assume a `matrix` type exposes named `rvec`/`uvec`/`fvec` row
overlays — some code only exposes `(-> mat vector 0..2)`. Check the
actual type definition before writing code that assumes named row fields
exist.

### 1.4.11 Bit-flags: position, not name, is the stable contract

Bitfield enums (`sp-group-item-flag`, vehicle `:flags`,
`traffic-alert-flag`, etc.) keep stable bit **positions** across engine
revisions and across games, but the decompiler can assign different
*names* to the same bit in different games (generic `sp0..sp15` in one
file, semantic names like `is-3d`/`launch-asap` in another). When moving
code that touches a bitfield between files or between games, verify the
bit position matches — never assume the name carries over unchanged, and
watch for a naive text search that only matches alphanumeric flag names
and silently skips ones containing a hyphen.

### 1.4.12 Symbols and functions linger in a hot REPL session

Renaming or deleting a function or symbol does not remove it from an
active REPL session's memory. Code that still references the old name
keeps "working" in that session, masking a broken reference that a clean
boot would catch immediately — another reason
[1.4.2](#142-repl-ghost-memory) and [1.4.3](#143-a-clean-compile-is-not-a-validation)
both end in a cold boot.

### 1.4.13 `(new 'static ...)` fields must be written inline

Inside a `(new 'static ...)` form (a static menu, a static array of
entries), both array fields and function fields must be written inline.
Referencing a `define`d array or a named `defun` fails to compile with
*"could not be evaluated at compile time"*. Wrap a named helper in a
one-line `lambda`:

```lisp
:on-confirm (lambda () (my-helper) (none))   ;; not :on-confirm my-helper
```

### 1.4.14 `process-spawn` does not install state until the next frame

A newly spawned process has no active state until the kernel dispatches
it on the frame *after* the spawn. `send-event` to a just-spawned process
hits a stateless process and the event is silently dropped — no error.
Spawn on one frame, send on a later one.

### 1.4.15 Porting code between games in the trilogy

When hand-porting `defstate`/`defpart`/`defpartgroup` GOAL code from one
game's decompiled source into another:

- Particle group/launcher IDs are index slots into a table sized by an
  explicit constant. Compute the next free block from the destination
  table's current length instead of reusing the source game's raw numeric
  IDs.
- `(new 'stack-no-clear 'array 'sometype N)` is untyped — see
  [1.4.9](#149-untyped-stack-arrays) — a common trap when porting into an
  older/simpler engine revision.
- A field present on the source game's vanilla type (a cached timestamp,
  an accuracy stat) may not exist on the destination's version of that
  same type. Add it to the mod's own subtype, never widen a shared
  vanilla type's layout.
- A referenced particle launcher/group ID from the source may simply not
  exist in the destination's decompiled tables — drop that reference
  rather than guessing a substitute.
- API gaps are common, not exceptional. A newer game frequently has
  helpers an older one lacks entirely (a generic light-trail renderer, a
  2D zoom-blur setup call, `defstate :parent`). Check for a same-purpose
  primitive already in the destination engine before concluding a
  feature can't be ported.
- Texture substitution is expected, not a bug. A shared texture page
  (e.g. `effects`) rarely has pixel-identical content across games; keep
  an explicit name-to-name substitution table next to the port script
  rather than silently reusing whatever name happens to compile.

