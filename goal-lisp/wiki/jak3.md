# Lisp wiki — Jak 3 specifics

What differs in Jak 3. Read [common.md](common.md) first: only real differences live here. Index: [index.md](index.md).

## 4.1 Dark Jak stages (`darkjak-stage` bitfield)

Dark Jak capabilities are driven by the `darkjak-stage` bitfield enum in
`target-h.gc`, stored in `(-> self darkjak stage)` and
`(-> self darkjak want-stage)`:

| Flag | Effect |
|---|---|
| `active` | Base Dark Jak form. |
| `bomb0` / `bomb1` | Dark Bomb / Dark Blast. |
| `invinc` | Invulnerability. |
| `invis` | Invisibility (suppresses offensive stages). |
| `tracking` | Target tracking. |
| `smack` | Dark Strike. |
| `giant` | Scaling stage flag. |

Entry validation: `want-to-darkjak?` / `want-to-powerjak?` (in
`target-darkjak.gc` / `target-lightjak.gc`) check the
`(game-feature darkjak)` flag in `*setting-control*`, focus tests, and
timing. Transformed movement uses `*darkjak-trans-mods*` surface
parameters. The Jak 3 engine still carries the Jak 2 "Dark Giant"
animation `jakb-darkjak-get-on-fast-ja` and the scale-interp var
`(-> self darkjak-giant-interp)` as a legacy leftover.

## 4.2 Secrets menu (`game-secrets`)

Secrets and cheats are tracked by the `game-secrets` bitfield enum in
`settings-h.gc`, persisted in `(-> *game-info* secrets)`.

```lisp
(logtest? (game-secrets <flag>) (-> *game-info* secrets))
```

Menu entries are `secret-item-option` instances in static arrays like
`*menu-secrets-array*` (`secrets-menu.gc`). Custom/unlocalized labels are
mapped dynamically during option rendering in `progress-draw-pc.gc`.

## 4.3 Powers/weapons interplay

Modifying `*target*` states can disrupt weapon transitions (`gun-states`)
and power transitions (`light-jak` / `dark-jak`). Test both after any
`target` change.

## 4.4 Weapon system (`gun`)

```lisp
;; weapon state, ammo and morph are read through the player process
(when *target*
  (let ((gun (-> *target* gun)))
    ;; access firing modes, ammo counts, morph attachments
    ))
```

## 4.5 Custom art-groups: no dynamic linking hook

Unlike Jak 1 and Jak 2 ([1.2.8](common.md#128-custom-art-groups--dynamic-animation-linking-link-art)),
Jak 3's engine code has no `register-custom-art-group` / `link-art!` /
`needs-link?` hook. Build a self-contained art-group instead
(`build-actor` without `:master-art-group`), or attach the entity through
`target-set-lod`-style skeleton swapping if it must ride on Jak/Daxter's
own skeleton (see the `custom-actors-levels` skill for the general asset
pipeline).

## 4.6 Memory

Kernel entry: `goal_src/jak3/kernel/gcommon.gc`,
`goal_src/jak3/engine/level/level.gc`. Jak 3 ships the PC memory
extension:

```lisp
;; goal_src/jak3/kernel/gcommon.gc
(defconstant END_OF_MEMORY #x20000000)     ;; 512 MB

;; goal_src/jak3/engine/level/level.gc
(defconstant DEBUG_LEVEL_HEAP_MULT 15.0)   ;; master-dev; ends the level heap above 256 MB
```

15.0 puts the top banks of the level heap above 256 MB, where adgif shader links cannot reach
(`shader-ptr`, see [Memory](common.md)): a level loaded there breaks the texture lists and a
later level unload crashes in `unlink-shaders-in-heap`. 12.0 keeps every bank below
`#xf400000`. `BORROW_MULT` follows it (`(defglobalconstant BORROW_MULT DEBUG_LEVEL_HEAP_MULT)`).

Verified: jak3, `engine/gfx/texture/texture-h.gc`, `engine/level/level.gc`; shader lists walked
in game (44 broken links at 15.0, 0 at 12.0), 2026-10-14.


## 4.7 Start menu: adding a page

The ring of the START menu (`*main-options*`) has exactly ten slots: `ring-want-angle` is
`36 degrees * option-index` (`engine/ui/progress/progress.gc:1246`) and the ring model has ten
segments. An eleventh entry does not fit. Hang a page off a list page instead, such as
START > Options (`*options-options-pc*`, `pc/progress/progress-static-pc.gc`), whose
`menu-sub-menu-option` entries are placed by `offset-y`, a fraction of the page height
(stock: 0.1 to 0.8, step 0.175). Swap its `options` array for one with your entry and spread the
offsets over the same span; restore both to remove it.

The entry's `next-state` must resolve to a page: wrap `progress` `set-menu-options` (the stock
method is in `pc/progress/progress-pc.gc`) and map your symbol to a generic scrolling page, like
the stock `'camera-options` maps to `*camera-options-pc*`. Generic links inside that page come back
to it through the generic history stack, which calls `set-menu-options` again with your symbol.

```lisp
;; keep the stock method across REPL reloads, then extend it
(define-perm *my-stock-set-menu-options* (function progress symbol int)
  (the-as (function progress symbol int) (method-of-type progress set-menu-options)))

(defmethod set-menu-options ((this progress) (arg0 symbol))
  (cond
    ((= arg0 'my-page)
     (set! (-> this current-options) *my-page-options*) ;; a progress-new-generic-scrolling-page
     0)
    (else
      (*my-stock-set-menu-options* this arg0))))
```

Labels of generic options are `text-id`s: add them at the end of the `text-id` enum
(`engine/ui/text-h.gc`) and their strings to `game/assets/jak3/text/game_custom_text_<lang>.json`
(English is loaded as the fallback for every language). A custom option type subclasses
`menu-generic-option` and defines `respond-progress` and `draw-option`: while it is selected the
scrolling page only consumes confirm and triangle, so the D-pad reaches it. The confirm that selects
it arrives with `selected?` still `#f` (load the value there); the confirm that ends the selection
arrives with `selected?` `#t` (commit there).

Two traps of the stock generic options: a `menu-generic-button-option` fires again on the confirm
that deselects it (guard actions that must not run twice), and `menu-generic-confirm-option` runs
its action on confirm whatever the yes / no choice; subclass it and test `confirmed?` in
`call-on-confirm` when "no" must do nothing.

Verified: jak3, `engine/ui/progress/progress.gc`, `pc/progress/progress-pc.gc`,
`progress-generic-pc.gc`, `progress-generic-draw-pc.gc`; compiled in the multiplayer mod, not yet
played, 2026-10-14.

## 4.8 Making the player follow another process: `target-grab-ride`

`target-grab-ride` (`engine/target/target2.gc:631`) disables the player's collision, sets no
animation channel, and every frame moves the player to `(get-trans proc 0)` of a
`process-focusable`. It is entered from the standard event handler with the sender as the process
to follow, and left with the grab exit:

```lisp
;; `seat` is a process-focusable whose root trans is where the player must be
(send-event *target* 'change-mode 'grab-ride #t :from seat)
;; ... later
(send-event *target* 'end-mode 'grab) ;; back to target-stance
```

`get-trans` mode 0 only reads `(-> root trans)`, so the followed process may have a plain `trsqv`
root and no skeleton. The player's model stays visible in its last pose: hide it with
`(send-event *target* 'draw #f)`.

Verified: jak3, `engine/target/target2.gc`, `engine/target/target-handler.gc` (`'grab-ride`
branch), `engine/process-drawable/process-focusable.gc` (`get-trans`), 2026-10-14.

## 4.9 Playable Daxter (`indax` mode)

Jak 3 has a playable Daxter: `target-indax-*` states (`engine/target/indax/`), entered with
`(send-event *target* 'change-mode 'indax #f 'stance)` from the standard handler (a truthy third
parameter skips the "Daxter jumps off Jak" intro, which leaves a Jak manipy behind) and left with
`(send-event *target* 'end-mode 'indax)`. `focus-status indax` is set meanwhile, and the target's
`draw art-group` is Daxter's (`daxter-ag`), with the indax clips linked into it by
`jak-indax+0-ag`.

Only the FACC, VOCA and HALFPIPE DGOs carry `target-indax.o`, `target-indax-hang.o` and
`jak-indax+0-ag.go` (VIN carries the art only); GAME only has `indax-h.o`. To use the mode
anywhere, list the three objects in `game.gd` (the art after `daxter-ag.go`) and remove them from
those level DGOs: a copy left in a level DGO would relink the states into that level's heap, and
they would dangle once it unloads.

Verified: jak3, `goal_src/jak3/dgos/*.gd`, `engine/target/indax/target-indax.gc`,
`engine/target/target-handler.gc:1363`; moved into GAME and compiled in the multiplayer mod, not
yet played, 2026-10-14.

## 4.10 Wrapping a stock method from a GAME file

To add behavior to a stock method without copying it, keep the stock method in a `define-perm`
(so a REPL reload does not capture the wrapper itself) and `defmethod` a wrapper that calls it;
the example is in [4.7](#47-start-menu-adding-a-page). Child types that inherited the stock
method take the wrapper only while method-set propagation is on: `method_set`
(`game/kernel/jak3/kscheme.cpp`) updates the children when `*enable-method-set*` is set, which the
C kernel does while it loads GAME (`load_and_link_dgo_from_c("game", ...)` in
`game/kernel/jak3/kmachine.cpp`). So the wrapper must be in a GAME file; types loaded later
inherit it at creation. The multiplayer mod wraps `enemy` `mark-as-dead`
(`engine/common-obs/enemy-states.gc`, called by `on-dying` for every enemy) this way.

Verified: jak3, `game/kernel/jak3/kscheme.cpp` (`method_set`), `game/kernel/jak3/kmachine.cpp`;
compiled, not yet played, 2026-10-14.

## 4.11 Making a process hittable by the player's attacks

The player's punches, kicks and shots never look in `*collide-player-list*` with a `jak` collide-as:
the target's attack collide-with (`target-collide-set!`, `target-danger-set!`) lacks the `jak` bit,
and gun shots query the actor hash, built from `*collide-hit-by-others-list*`. To be hit, a shape
must be in that list, collide-as a bit the attacks look for (`bot`, `civilian`, `enemy`,
`obstacle`), and `solid` (shots only collide with solid prims). The list enum is offset by one:
`(collide-list-enum usually-hit-by-player)` adds to `*collide-hit-by-others-list*`.

```lisp
(let ((cshape (new 'process 'collide-shape self (collide-list-enum usually-hit-by-player))))
  (let ((prim (new 'process 'collide-shape-prim-sphere cshape (the-as uint 0))))
    (set! (-> prim prim-core collide-as) (collide-spec bot civilian))
    (set! (-> prim prim-core collide-with) (collide-spec))
    (set! (-> prim prim-core action) (collide-action solid))
    (set-vector! (-> prim local-sphere) 0.0 (meters 1) 0.0 (meters 0.9))
    (set! (-> cshape total-prims) (the-as uint 1))
    (set! (-> cshape root-prim) prim))
  (set! (-> self root) cshape))
;; its :event gets 'attack, from the target (melee) or from the projectile (shots); return #t
```

Keep a solid sphere's underside out of the player's way: a sphere centered above the ground is a
ceiling for a player standing next to it, who slides down and stays in `target-duck-stance` while
under it. Center big spheres (a vehicle's) at ground level or below.

A process that must also be focused by enemies (they scan the player list for `jak`) needs a
second process for this shape, as one collide-shape sits in one list. A blow lands several frames:
dedupe by `(-> attack-info id)`. Hitting the player back: send `'attack` with a fresh
`(new-attack-id)` and a `vector` (the push direction): without one, `target-hit` pushes away from the
sender's `root`, which a plain process does not have.

Verified: jak3, `engine/target/logic-target.gc`, `engine/collide/collide-shape-h.gc`,
`engine/target/target.gc` (`target-slide-down`); seen in game in the multiplayer mod (player hits,
health 8 to 5, 3 m knockback; a player crouched next to a 2.5 m sphere centered 1 m up),
2026-10-14.

## 4.12 Drawing another process's model elsewhere

A process keeps its draw-control's `art-group` and `jgeo`, not the skeleton-group it was built
from. Every `defskelgroup` registers its skeleton-group in the loading level's art-group array next
to the art groups, so search the active levels' arrays for a skeleton-group whose `art-group-name`
is the process's art group name and whose `jgeo` index points at the same joint geometry. Then
build the copy with `initialize-skeleton`, after setting the copy's `level` to the level that holds
the **art group** (`art-group-get-by-name` on `art-group-name`), which is not always the level that
registered the skeleton-group (the dogat's skeleton is in WASALL code, its art elsewhere).

Play the source's animation by index in the art group (channel 0 frame-group and frame-num); the
copy's effect-control then plays the effects carried by the animation (footsteps, sounds).
`setup-masks draw -1 seg-mask` copies a body variant (citizens), except a 0 mask.

Daxter on Jak's shoulder (`sidekick`) and Jak's gun keep their root still: their root joint is
computed from one of Jak's joints each frame. Their world transform is
`(-> proc node-list data 0 bone transform)`.

To show the source's exact pose (blends, streamed animations, joint mods, ragdolls), copy its
bones instead. `(-> proc node-list data i bone transform)` is each bone's world matrix, and the
renderer skins with it (`bones-mtx-calc-execute` multiplies it by the joint's bind pose). Write
the copy's bones from its joint-control `postbind-function`
(`(function draw-control cspace-array joint-control none)`, `pp` is the process): `do-joint-math`
calls it after computing the bones from the animation and before deriving the draw origin from
them. Do not write them from the process's `post`: `ja-post` usually only queues the math on
`*matrix-engine*`, run after every process. The source's bones are complete only when it is
drawn and its current level of detail animates every joint:
`(>= (+ (-> draw mgeo num-joints) 3) (-> node-list length))`, or `force-math` in the
joint-control status; a lower level of detail leaves the other bones stale.

Within a frame (`engine/game/main.gc`), the processes run first, then `execute-math-engine`
computes the queued bones, then `*mods-menu-hook*` runs: a process reads the bones of the previous
frame, the mods hook those of this frame.

A bone's world matrix is homogeneous: a few joints are sometimes computed with the `w` of their
translation row a little off 1 (0.9992 on two joints of Jak driving a buggy, for one or two
frames). At world coordinates of millions of units (Spargus is about 9.2 million units from the
origin), reading the translation row alone then moves those joints by more than a meter. A pose
expressed relative to the object's frame (`bone x inverse(frame)`, the whole row) is right; to
compare a bone with its parent or bone 0, take `trans - w * base`, not `trans - base`.

Sent to another machine, a pose drawn at the sender's time needs a clock finer than the
millisecond: a running citizen's bones move about 3.5 mm per ms, so whole-millisecond stamps on
the poses and in the clock estimate leave up to about 7 mm between a copy and its source, tenths
of a millisecond 1.6 mm at most (the 1 mm steps of the wire format). The frames of the game are
not even (a 43 ms frame, then two short ones): compare a copy with its source's bones logged on
every frame, not on a script's fixed times.

Verified: jak3, `engine/process-drawable/process-drawable.gc` (`draw-control-method-14`,
`do-joint-math`, `ja-post`), `engine/gfx/foreground/bones.gc`, `engine/target/sidekick.gc`; seen in
game in the multiplayer mod (replicas of citizens, Daxter and the gun on another machine; a
remote Jak in a buggy matching the driver's bones within 1 mm; a running citizen's copy within
1.6 mm with 0.1 ms stamps, 6.7 mm with 1 ms stamps; joints 48 and 64 of the driver read 1.5 m
off with `trans - base`, 1.8 mm at most with `trans - w * base`), 2026-10-14.

## 4.13 Hooks for every effect, projectile, sound and story step

- Every one-shot particle effect (`part-tracker-spawn`, `do-effect "group-..."`) goes through
  `part-tracker-init-base` (`engine/common-obs/generic-obs.gc`), which returns normally; the
  group is its first argument and `(-> self root trans)` (or the target's root) its position.
  Replaying one elsewhere: set `(-> *launch-matrix* trans)`, then `part-tracker-spawn` with
  `part-tracker-subsampler` when the group has `(sp-group-flag sp13)`, else `part-tracker`. A
  group's id is its index in `*part-group-id-table*`; a level's groups are zero while it is not
  loaded.
- Every projectile goes through `projectile-init-by-other` (`engine/common-obs/projectile.gc`),
  which never returns (it goes to the first state): act before calling the stock one. Fire one
  with `spawn-projectile` (type, params, a process-drawable parent, pool); the type is level code,
  so check `(type? (-> (string->symbol name) value) type)`. To make a copy harmless, set its
  `(-> root root-prim prim-core collide-with)` to `(collide-spec backgnd)`.

- Every running particle group goes through the `sparticle-launch-control` `spawn` method
  (`spawn-from-mat` and `spawn-from-cspace` call it); `(-> this proc)` is the owner and
  `(-> this origin)` the launch matrix once its `trans` is set to the argument. Particles launched
  directly (`launch-particles`, no launch control) go through `sp-launch-particles-var`, a
  `def-mips2c` function reached through its symbol like the others; `pp` is the launching process.
  `*part-id-table*` keeps the launchers of unloaded levels (`unlink-part-group-by-heap` only clears
  `*part-group-id-table*`): an entry is safe only when its address lies in the heap of a loaded
  level that defines it, or outside every level heap (global parts).
- Every sound goes through `sound-play-by-name` or `sound-play-by-spec` (behaviors, `self` is the
  playing process; `trans` is a vector, `#t` for the process root, `#f` for none) and stops
  through `sound-stop`. Playing again with an id that is still playing updates that sound
  (`game/overlord/jak3/srpc.cpp`): a lasting sound such as an engine is one id played every frame.

- Every story step goes through the `game-task-node-info` `open!` method
  (`engine/game/task/task-control.gc`), which, despite its name, **closes** the node (and the
  parent nodes it needs, through `open!` again), runs its `on-close` script and gives the fuel
  cell of a task-closing node. The node's index is `(get-idx-in-task-list node)` in
  `(-> *game-info* sub-task-list)`; `(open! node 'event)` closes it anywhere, as when the task is
  done.

All are functions or methods called through their symbol or method table, so a `define-perm` +
`defbehavior`, `defun` or `defmethod` wrapper in a GAME file catches every caller (see
[4.10](#410-wrapping-a-stock-method-from-a-game-file)).

Verified: jak3, the files above, `engine/gfx/sprite/particles/sparticle-launcher.gc`,
`engine/sound/gsound.gc`; seen in game in the multiplayer mod (shots, impacts, a buggy's dust and
engine sound replayed on another machine; task nodes closed on one machine closed on the other),
2026-10-14.

## 4.14 Keeping world objects alike on several machines

- **Cycles.** An object on a cycle (moving platform, wheel, light) gets its phase from a
  `sync-info` (`engine/util/sync-info.gc`) as `(mod (current-time) period) + offset`, where
  `current-time` is the calling process's clock (`(-> pp clock)`, one of `(-> *display* clock i)`).
  Every method reads the `offset` field, so adding a constant to it for the duration of a call
  (and putting it back) shifts the phase without moving any clock or timer. `get-norm!` is
  overridden by `sync-linear`, `sync-eased` and `sync-paused`; `sync-now!` writes `offset`.
- **Doors.** A `com-airlock` (`engine/common-obs/airlock.gc`) opens when `want-cross-airlock?` says
  so, computed from `(target-pos 0)`: distance to `active-distance`, `check-crossing-distance`
  under `open-distance`, `y-height`, `latch-closed-time`, `(script-eval open-test)`, the
  `airlock` setting and `airlock-command-lookup`. Every door type of the game inherits it.
- **Elevators.** An `elevator` (`engine/common-obs/elevator.gc`) in its `waiting` or `arrived` state
  takes `(send-event elev 'move-to 'top)` or `'bottom` and runs there; `(-> elev move-pos 1)` is
  the stop it runs to, `bottom-top` its two ends.
- **Physics objects.** A `rigid-body-object` (`engine/physics/rigid-body.gc`) is put somewhere
  with `(rigid-body-control-method-28 (-> obj rbody) trans quat)` (root position and orientation),
  then `reset-momentum!` and `init-velocities!` to leave it at rest; its next `rbody-post` moves
  the root and the collision there and still carries its riders. Objects spawned by one entity's
  process share that entity (the volcano's 5 lava platforms and their spawner): name them by the
  aid plus their rank among the spawner's children, the same on every machine. A floating
  platform (`rigid-body-platform`, volcano level code) takes a player's weight from
  `(send-event plat 'ridden rider)`, `rider` a `collide-rider` whose `rider-handle` is a process
  with `(process-mask target)` set.
- **Stale bones.** `(do-joint-math (-> proc draw) (-> proc node-list) (-> proc skel))` can be called
  from another process (it sets `pp` itself): it computes the bones of a process whose own joint
  math did not run lately.
- **Cutscenes.** Every cutscene starts with `scene-player-init` (`engine/scene/scene.gc`, a
  `scene-player` behavior; scenes as a name, a scene, a list or an array, then the wait flag and a
  continue point name). `(process-spawn scene-player :init scene-player-init name #t #f
  :name "scene-player")` plays one; `scene-lookup` finds it by name.

Verified: jak3, the files above, `levels/volcano/volcano-obs2.gc`; seen in game in the
multiplayer mod (a test cycle within one frame of the host's, a door opened for a remote player
200 m away, an elevator trip and a Spargus cutscene started on two machines, lava platforms
within 7 cm of the authority's and carrying the player standing on them), 2026-10-14.

## 4.15 Levels: memory banks, and what outlives a level

- A level needs a free bank of the level-group heap for its memory mode. Asking for one more
  level (`want-levels`, `want-display-level`) without one prints `ERROR: level <name> could not
  find free small-edge bank in the level-group heap`, then the loader stops the game on purpose
  (`break!`, an integer division by zero in the PC port). Load only what the player's position
  calls for.
- A level that leaves (`level` `deactivate`, called by `unload!`, also for a level only `'loaded`,
  not active) kills its entities and their particles, and unlinks its groups from
  `*part-group-id-table*` (but not its launchers from `*part-id-table*`, see 4.13); then its
  memory is reused. A process that is not one of its
  entities but uses its art (a model built from its skeleton-group, a part-tracker playing one
  of its groups) keeps pointing there: deactivate those before calling the stock `deactivate`,
  or the game crashes when they are drawn or spawn particles.

- The loader keeps the levels named in `*load-state*` `target` and discards the others, one per
  frame (`update!` of `load-state` in `level.gc`); `target` is rebuilt from `want-exp` only when
  the wanted levels change (`add-borrow-levels`). A level whose `master-level` is not loaded
  stops the game when asked for (`level-get-for-use`: "is loading before master-level").

Verified: jak3, `engine/level/level.gc`, `engine/load/load-state.gc`; seen in game in the
multiplayer mod (the bank error and crash when loading wascityb next to wascitya; crashes at a
level unload until copies and part-trackers of the level, active or only loaded, were dropped
first), 2026-10-14.

## 4.16 Loading the level of another place, outside the banks

The multiplayer mod loads the level a distant player stands in, next to the levels of the local
place, without taking a bank (`goal_src/jak3/pc/features/multiplayer/multiplayer-world.gc`,
"Distant players' levels"). Each step meets a trap of the engine:

- **Memory.** The banks of the PC level heap are 12 times the PS2 page and mostly empty. The level
  gets a heap carved from the free top of an active level (its host), below 256 MB (4.6), and the
  host's `top-base` and `top` are lowered to it, as `level-update-after-load` does for borrow
  heaps: `get-level-by-heap-ptr-and-status` gives the first level whose `[base, top-base)` holds
  an address, and art login stops the game ("illegal login of art ... to level ...") when that is
  not the loading level. `load-begin` gets a hook before its bank search (`memory-mask` 0).
- **Size.** Count the borrow heaps the level carves from its own top at login:
  `BORROW_MULT` times each `borrow-size` (KB). For `ctywide` that is 34 MB on top of its pages.
- **Unloading.** Registered in a borrow slot of the host that none of the host's own borrow levels
  can claim (`level-find-borrow-slot` gives slots 0 to 2 to `borrow-city-small` levels and slot i
  to the levels its borrow info names at i, so 3 or 4 when unused), the level is unloaded by the
  host's `unload!` first. Give the host its `top` and `top-base` back when it leaves.
- **Not displayed.** Kept in `target` with `display?` `#f`, it stays `'loaded`: no entity birth,
  no level callback (they run at birth and display), no traffic. `art-group-get-by-name` only
  searches `'active` and `'reserved` levels, and `assign-draw-indices` gives a draw slot to
  `'active` levels only: a model built from its art must take `level-index` 10 (the default
  level's buckets and lighting). The PC renderer finds merc models by name in any loaded level
  (`Loader::get_merc_model`).
- **Code.** A level's code is global: its `defun`, `define` and `deftype` of names the local levels
  use point those names at its copy, and a `deftype` of an existing type resets that type's
  methods to the parent's (`new_type` in `game/kernel/jak3/kscheme.cpp`). Two halves of a place
  the game never loads together share their code (Spargus `wascitya` and `wascityb`): the second
  one unloading leaves the first with dangling code. The mod notes every symbol value, the methods
  of every type and the `*part-id-table*`, `*part-group-id-table*`, `*lightning-spec-id-table*`
  and texture page dir entries before the load, and puts back each one that held a value and now
  points into the new level's memory, every frame while it links (`pc-mp-sandbox-*` in
  `game/kernel/jak3/kmultiplayer.cpp`; 2226 entries for `ctywide` loaded in Spargus). It also
  unloads it before the local place loads any level, so no local level links next to it.
- **Textures.** `link-texture-by-id` allocates the shader list of a texture page in the loading
  level's heap, and `unload-page` and `unlink-shaders-in-heap` only clean the pages that are
  loaded: a page the level used without loading it keeps a list in freed memory, and the next
  level loaded there corrupts it (a shader list looping forever in testing). Clear those links
  and splice the level's shaders out of every list when it leaves.
- **Art alone.** Levels are capped (10 slots, and the PC renderer keeps `LEVEL_TOTAL` = 11 levels
  of `fr3` data). An art group can be loaded without its level: `(loado "barge-ag" heap)` reads
  `out/jak3/obj/barge-ag.go` (the PC port maps `DATA_FILE_TYPE` there, `game/kernel/jak3/fileio.cpp`;
  the skeleton-group names it `"barge"`). Loaded while no level loads, it logs itself in
  (`art-group` relocate in `engine/anim/joint.gc`, which stops the game unless `*print-login*` is
  `#t`); append it to a level's `art-group` load-dir (100 entries) so `initialize-skeleton` finds
  it through the process's level. On PC its model is drawn only once the renderer has the `fr3`
  of the level the art comes from: `pc_set_levels` (`game/kernel/jak3/kmachine_extras.cpp`) is
  the list the renderer keeps, and at its cap it drops levels not active for 180 frames
  (`Loader::get_most_unloadable_level`).
- **Skeleton-groups are data.** A model drawn without its object's code needs only the
  `skeleton-group` (what `defskelgroup` fills: `art-group-name`, `jgeo`, `janim`, `bounds`, `mgeo`
  and `lod-dist`, `max-lod`, `version`, shadow and joint indices, `clothing` `#f` is fine) and
  the art group. `(new 'global 'skeleton-group)` with those fields set builds one that
  `initialize-skeleton` accepts; the multiplayer mod rebuilds them from a description its sender
  sends. Never keep a pointer to a skeleton-group (or any object) of a level that may unload:
  look it up again by name, its memory is reused (a crash in testing).
- **Borrow levels.** Its borrow levels (Haven's cars `ctycara`, guards `ctypesa`) load in its
  borrow heaps through the stock path once it is `'loaded`, and must leave `target` as soon as it
  is not: a borrow level asked for without its master stops the game.

Verified: jak3, `engine/level/level.gc`, `engine/gfx/texture/texture.gc`,
`engine/anim/joint.gc`, `game/kernel/jak3/kscheme.cpp`; in game with the multiplayer mod
(`far`: `wascityb` loaded in `wascitya`'s heap and back; `far-city`: `ctywide`, `ctypesa`,
`ctycara`, `ctycarb` loaded in Spargus, Haven's guards and car drawn there, then `barge-ag` and
`citizen-chick-ag` loaded alone with `ctyport` and `ctypepa` kept by the renderer, the boats and
citizens drawn; all unloaded cleanly), 2026-10-14.
