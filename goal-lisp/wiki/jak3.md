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
