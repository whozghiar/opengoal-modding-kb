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
(defconstant DEBUG_LEVEL_HEAP_MULT 15.0)   ;; already the highest tested of the three games
```


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

A process that must also be focused by enemies (they scan the player list for `jak`) needs a
second process for this shape, as one collide-shape sits in one list. A blow lands several frames:
dedupe by `(-> attack-info id)`. Hitting the player back: send `'attack` with a fresh
`(new-attack-id)` and a `vector` (the push direction): without one, `target-hit` pushes away from the
sender's `root`, which a plain process does not have.

Verified: jak3, `engine/target/logic-target.gc`, `engine/collide/collide-shape-h.gc`; seen in game
in the multiplayer mod (player hits, health 8 to 5, 3 m knockback), 2026-10-14.

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

Verified: jak3, `engine/process-drawable/process-drawable.gc` (`draw-control-method-14`,
`do-joint-math`, `ja-post`), `engine/gfx/foreground/bones.gc`, `engine/target/sidekick.gc`; seen in
game in the multiplayer mod (replicas of citizens, Daxter and the gun on another machine; a
remote Jak in a buggy matching the driver's bones within 1 mm), 2026-10-14.

## 4.13 Hooks for every effect, projectile and sound

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

All are functions or methods called through their symbol or method table, so a `define-perm` +
`defbehavior`, `defun` or `defmethod` wrapper in a GAME file catches every caller (see
[4.10](#410-wrapping-a-stock-method-from-a-game-file)).

Verified: jak3, the files above, `engine/gfx/sprite/particles/sparticle-launcher.gc`,
`engine/sound/gsound.gc`; seen in game in the multiplayer mod (shots, impacts, a buggy's dust and
engine sound replayed on another machine), 2026-10-14.
