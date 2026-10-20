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

The wave concussor (red 2), mass inverter (dark 2) and arc wielder (blue 2) are processes, not
projectiles: `gun-red-2-shockwave` (its `next-state` is `explode` once it goes off),
`gun-gravity`, and `gun-blue-2-lightning-tracker`, whose `setup-draw!` method receives each
frame's arc points. A tracker spawned to draw an arc elsewhere needs the stock heap size,
`:stack-size #x29400`: its six lightning bolts fill most of it, and with less it ends in
`process-drawable-art-error` "memory". Switch its bolts on as the stock `active` state's enter
does, `(lightning-bolt-method-13 bolt 0)`, or they draw nothing. Leave its skeleton out: the art
of `skel-gun-blue-2-tracker` is loaded only with the local player's own arc wielder.

Verified: jak3, `engine/target/gun/gun-blue-shot.gc` (`create-lightning-tracker-if-necessary`,
`active`), `engine/target/gun/gun-red-shot.gc`, `engine/target/gun/gun-dark-shot.gc`; arcs drawn
on another machine in the multiplayer mod, 2026-10-15.

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
(defconstant DEBUG_LEVEL_HEAP_MULT 12.0)   ;; master-dev since 3e47edcc6; 15.0 ended it above 256 MB
```

15.0 (master-dev before 3e47edcc6) put the top banks of the level heap above 256 MB, where adgif shader links cannot reach
(`shader-ptr`, see [Memory](common.md)): a level loaded there breaks the texture lists and a
later level unload crashes in `unlink-shaders-in-heap`. 12.0 keeps every bank below
`#xf400000`. `BORROW_MULT` follows it (`(defglobalconstant BORROW_MULT DEBUG_LEVEL_HEAP_MULT)`).

Verified: jak3, `engine/gfx/texture/texture-h.gc`, `engine/level/level.gc`; shader lists walked
in game (44 broken links at 15.0, 0 at 12.0), 2026-10-14.

Process heaps move. `*default-dead-pool*` is `*nk-dead-pool*`, a `dead-pool-heap`, compacted
every frame (`compact`, `shrink-heap`); a spawn with a large `:stack-size` (the arc wielder's
`#x29400`) forces a large compaction. Compaction calls `(relocate proc offset)` on the old memory,
then copies it: a relocate method fixes the pointers into its own heap that the parent's relocate
does not know of. `joint-control`'s relocate leaves `override` out (the stock players never move),
so a process that allocates `(new 'process 'boxed-array float 54)` for `(-> self skel override)`
relocates it itself:

```lisp
(defmethod relocate ((this my-model) (offset int))
  (if (and (nonzero? (-> this skel)) (-> this skel override) (nonzero? (-> this skel override)))
      (&+! (-> this skel override) offset))
  (call-parent-method this offset))
```

Left behind, every write through the stale pointer lands in whatever process was later put
there: zeroed fields in random processes, crashes in their code (a projectile's `die` reading a
`notify-handle` of 0, `change-parent`, `compact` meeting a heap object of type 0).

Verified: jak3, `kernel/gkernel.gc` (`compact`, `shrink-heap`), `engine/entity/relocate.gc`;
the multiplayer mod crashed once an arc wielder fired, and no longer did with this method,
2026-10-15.


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
(English is loaded as the fallback for every language). That English fallback goes into
`*fallback-text-heap*` (`text-h.gc`, PC port), only 80 KB against 128 KB for the language's own
text: once `out/jak3/iso/0COMMON.TXT` reached 78.6 KB, its link overran that heap and every boot
stopped in `c_symlink2` (`val is 0x1010101`, `klink.cpp:289`) right after loading `0common.TXT`,
with no "Game text heap overrun!" (that check compares with the whole heap). Watch that file's size
when adding many strings; raising the heap to `(* 128 1024)` fixed it (multiplayer mod, every
scenario booting again, 2026-10-19). A custom option type subclasses
`menu-generic-option` and defines `respond-progress` and `draw-option`: while it is selected the
scrolling page only consumes confirm and triangle, so the D-pad reaches it. The confirm that selects
it arrives with `selected?` still `#f` (load the value there); the confirm that ends the selection
arrives with `selected?` `#t` (commit there).

Two traps of the stock generic options: a `menu-generic-button-option` fires again on the confirm
that deselects it (guard actions that must not run twice), and `menu-generic-confirm-option` runs
its action on confirm whatever the yes / no choice; subclass it and test `confirmed?` in
`call-on-confirm` when "no" must do nothing.

- **Title screen.** Its list is `*title-pc*` (New Game, Load Game, Options, Secrets, Quit); it
  opens on Load Game when a save exists, and only after the title screen's first press
  (`title-progress`, `levels/title/title-obs.gc`). Until Secrets is unlocked the stock hides that
  entry by its name but still counts it: moving skips index 3 (`respond-to-cpad`,
  `progress-generic-pc.gc`), sub-menu entries are drawn 0.08 lower and Quit 0.07 higher
  (`progress-draw-pc.gc`). A list with more entries must keep Secrets at index 3 while it is
  locked (pick the order from `(memcard-unlocked-secrets? #f #f)`).
- **The intro of a retail boot.** Without `-debug`, the title screen (`title-obs.gc`, `demo-state`
  0) waits for a press on its demo screen, then starts the game at `pre-intro-start`: the opening
  cutscenes (`levels/intro/intro-scenes.gc`) play, about 4.5 minutes, before the title menu. They
  carry `scf1`: triangle skips each one, and their `end-point` `title-restart` brings the title
  menu back. A test that waits for the title menu presses triangle while a skippable scene plays.
- **Going to a page from your own option:** `(push-state progress)` then
  `(set-next-state progress 'my-page 0)`, as a `menu-sub-menu-option` does with its `next-state`.
  Triangle goes back a page (`pop-state`) after the option's `respond-progress` returned.
- **A page drawn by one option** (a `menu-option` subclass alone in a `menu-option-list`): its
  `respond-progress` sees every press first; clear the ones it uses (`cpad-clear!`). Its
  `draw-option` is handed the unadjusted x (70) and width (375), while the page's frame is drawn
  squeezed to the screen's ratio: compute the frame with `adjust-game-x` (unless `use-vis?`).
  `get-aspect-ratio` can say `aspect4x3` in a 16:9 window (the game's own setting).
- **Everything is paused under a menu,** a mod's processes too, at the title screen as in game:
  work that must go on (network) belongs in the per-frame hook (`*mods-menu-hook*`). The menu
  stops them through a setting: `progress-init-by-other` adds `'process-mask 'set (process-mask
  progress)`, and every process masked `progress` stops. To keep the world running under it, put
  the opposite setting in `*setting-control*` `engine-hi`, which `apply-settings` applies after
  every other connection (`set-setting-by-param` replaces the one of that key there):

  ```lisp
  ;; the world runs under the START menu
  (set-setting-by-param *setting-control* 'process-mask 'clear 0 (process-mask progress))
  (apply-settings *setting-control*)
  ;; back to the stock
  (remove-by-param0 (-> *setting-control* engine-hi) 'process-mask)
  (apply-settings *setting-control*)
  ```

  The same pair overrides any setting a mission holds (`'pilot-exit #t` over a race's `#f`). The
  menu also pauses every sound (`set-master-mode 'progress`): `(sound-group-continue (the-as
  sound-group #xffffffff))` once it is open lets them go on. It shows a still copy of the screen
  (`blit-displays-work` `set-menu-mode`, `engine/gfx/blit-displays-h.gc`): wrap that method and pass
  `#f` to show the live game. The world still reads pad 0 meanwhile; hide it with a process that
  runs before everything (4.20): copy `(-> *cpad-list* cpads 0)` aside, `cpad-invalid!` it, and
  copy it back from the per-frame hook, which runs before the menu reads it.
- **A model drawn over the menu** (a preview on a page) must carry `draw-control-status hud`
  (bit 14): while the still copy shows (menu mode, as at the title screen), the foreground only
  draws the draw-controls with that status. Turning the menu mode off instead brought back
  buckets with no renderer (`tex-lcom-pris2`, bucket 456) and stopped the game on an assertion.
  `(if (get-menu-mode *blit-displays-work*) (logior! (-> proc draw status) (draw-control-status
  hud)) (logclear! (-> proc draw status) (draw-control-status hud)))` each frame.
- **Text from the per-frame hook:** `print-game-text` adds its own DMA bucket. Called inside a
  `with-dma-buffer-add-bucket` block it nests two buckets and the game stops on the `!m_ended`
  assertion: call it after the block closes.
- **SELECT pauses** like START (`determine-pause-mode`, `engine/draw/drawable.gc`). To give it
  another use, wrap that function and clear SELECT from `(-> *cpad-list* cpads 0 button0-rel 0)`
  before calling the stock one.
- **Opening the START menu on a page of yours** from the game (a shop at an NPC):
  `(activate-progress *dproc* 'my-page)`; the `set-menu-options` wrapper routes `'my-page` to
  your options (the multiplayer mod's Kleiver's shop, scenario `economy`, 2026-10-19).

Verified: jak3, `engine/ui/progress/progress.gc`, `pc/progress/progress-pc.gc`,
`progress-generic-pc.gc`, `progress-generic-draw-pc.gc`, `progress-draw-pc.gc`,
`levels/title/title-obs.gc`, `engine/draw/drawable.gc`; the multiplayer mod's title screen entries,
lobby page and SELECT player list driven by the `lobby` and `player-list` scenarios
(screenshots, pages logged), 2026-10-16; `engine/ui/progress/progress.gc`
(`progress-init-by-other`), `engine/game/settings.gc` (`set-setting-by-param`, `apply-settings`);
the multiplayer mod's START menu over a running world (`pause` scenario: clocks running, no still
copy) and the `print-game-text` crash, 2026-10-19.

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
`(send-event *target* 'draw #f)`. Only the position follows: the player keeps the facing it had
when it got in (`control quat`).

Verified: jak3, `engine/target/target2.gc`, `engine/target/target-handler.gc` (`'grab-ride`
branch), `engine/process-drawable/process-focusable.gc` (`get-trans`), 2026-10-14; the facing in
the multiplayer mod's `ride-seat` scenario (a passenger drawn with the player's facing sat at
random angles), 2026-10-19.

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

- **Daxter at the wheel:** `'change-mode 'pilot-daxter` (the parameters of `'pilot`) drives with
  the target's art swapped for Daxter's (`pilot` `as-daxter?`). Its clips are in
  `jak-pidax+0-ag.go`, which only LCTYPATK carries: elsewhere the target dies in
  `process-drawable-art-error`. List it in `game.gd` and remove it from `lctypatk.gd`.
- **From the indax mode,** the handler ignores `'change-mode 'pilot`: end the mode first
  (`'end-mode 'indax`), then ask from `target-stance`.
- **Held as Daxter:** in the indax mode, `'change-mode 'grab` enters `target-indax-grab`, the
  counterpart of `target-grab` (the player held, moved by whoever holds him); `'end-mode 'grab`
  goes back to `target-indax-stance` (`target-indax.gc:1769`).
- **Guns as Daxter:** his surfaces (`*indax-walk-mods*`...) carry `gun-off`. Never give the gun a
  nonzero `gun-type` in this mode: the gun's `use` state calls `target-gun-check` every frame,
  which pushes Jak's `jakb-gun-*` clips onto the player's skeleton. What works:
  - The D-pad marking menu (`target-gun-marking-menu`, a `*pad-engine*` connection) still runs
    and sets `(-> *target* gun using-gun-type)`.
  - Fire through the per-color functions `target-gun-fire-yellow` / `-red` / `-blue` / `-dark`
    (pickup-type 26-37), run as the target (`pp` set to `*target*`), after setting the gun-info's
    `fire-point`, `fire-dir-out` (and `laser-dir`: the vulcan fury reads it). They need no Jak
    animation; `target-gun-fire` adds those and takes the ammo of 26 28 29 30 32 35 36 37
    (`ammo-required`).
  - The ranges (`fire-range`, `track-find-range`...) are set by `target-gun-type-set!` only:
    set them yourself, or the scatter gun reaches nothing.
  - `gun-red-2-shockwave` (`charging`) goes off after 0.1 s and `gun-dark-shot` (`startup`)
    fizzles when `*target*` is in `indax` mode: spawn a subtype whose `(defstate charging
    (subtype) :virtual #t ...)` drops that check (unspecified handlers are inherited,
    `(call-parent-state-handler enter)` runs the parent's). `process-spawn` of a subtype looks
    for `<subtype>-init-by-other`: pass `:init gun-red-2-shockwave-init-by-other`.
  - The arc wielder's tracker stays on while `using-gun-type` is 33 and `gun active?` is `#t`.
  - Where the gun is drawn: `target-gun-compute-pos`, called by the gun's `gun-post` with `self`
    set to the target (`target-gun.gc`); a `define-perm` wrapper can place it. In his hand: his
    gun joint, `(-> *target* sidekick 0 node-list data 28 bone transform)`, with `gun gun-daxter`
    1.0 (the stock place when he fires from a vehicle; his mode skips `target-gun-joint-points`,
    which sets it, so set it back to 0.0 after).
  - `gun-post` hides the gun (`draw-control-status no-draw`) while the player has
    `no-draw-bounds2`, which his mode sets: clear it from a process that runs after the gun's post
    (one in `*pc-pool*`, 4.20).
  - The gun's `idle` state holds its folded clip (`gun-idle-ja`); its `use` state the mod's open
    shape, `gun-idle-red-ja` ... (gun-ag 13-24: red 1-3 13-15, dark 1-3 16-18, yellow 1-3
    19-21, blue 1-3 22-24). Setting that clip on the gun's channel 0 from
    `target-gun-compute-pos` (the gun's post, after its code) opens it.
  - Daxter's clips holding the gun are the ones he fires from a vehicle with:
    `daxter-pilot-gun-<color>-idle` (308 red, 309 dark, 310 yellow, 311 blue) and `-fire` (312
    red, 313 yellow, 314 dark, 296 blue loop), with the gun joint in his hands. They are in
    `jak-pilot-gun+0-ag.go` (88 KB), which only COMBA, CWI, HGA and RAILA carry: list it in
    `game.gd` and remove it from those four. On his upper body only, over his mode's clips: 4.19.
- **Respawn in a loaded game:** a death restores the features of the game loaded (for a game
  loaded from memory, that save's): re-apply what you gave since.
- **The sidekick:** Jak's Daxter is `(-> *target* sidekick)`, which can be `#f` (during the arena
  training's cutscene): test it before `(-> ... 0)`. Every frame it copies the player's clips, then
  calls `joint-control-remap!`, where a `define-perm` wrapper can put another clip on it. Its
  feature bit is `#x100000`: `(set-setting! 'features 'clear 0 #x100000)` takes him off Jak (as
  `target-death.gc` does), without touching the saved features.
- **Daxter climbing onto Jak's shoulder:** the stock climb back after Dark Jak is
  `daxter-darkjak-get-off-ja` then `daxter-darkjak-get-off-end-ja` (Daxter's art 420 and 427,
  `engine/data/art-elts.gc`). Played on the sidekick's channel 0 from that wrapper, each clip
  advancing by its own `speed` times `time-adjust-ratio` per frame, the pair lasts 159 frames (at
  1.0 a frame, 21 frames: far too fast).
- **Seated clips:** `target-pilot-stance` (`pilot-states.gc`) plays the vehicle's
  `<x>-turn-back` clip on channel 0 and `<x>-turn-front` (the index before it) on channel 1,
  blended by how the vehicle pitches, their frames by the steering (middle frame: straight), by
  `rider-stance` (vehicle info: 0 hover bikes `jakb-pilot-bike-*` 240-241, 1 hover cars
  `jakb-pilot-car-*` 233-234, 2 wasteland cars `jakb-pilot-wcar-*` 268-269, 3 glider). These
  come with the levels that have vehicles (`jak-pilot+0-ag`, `-hcar`, `-wcar`). Daxter driving
  (`as-daxter?`) uses `daxter-pidax-missile-turn-front` / `-back` (541-542) in every vehicle,
  placed from the seat point (the `'indax` matrix: control trans plus `cspace-offset`). His
  `daxter-pilot-*` clips are not: the sidekick's default matrix puts his root at Jak's node 34
  (the shoulder) with the sidekick's facing, and those clips seat him next to Jak from there.

- **Eyes:** every model of an art group shares one eye slot (`*eye-control-array*`, found by
  the art group's name when the model logs in), which the last model to animate fills
  (`merc-eye-anim`, `engine/gfx/foreground/eye.gc`). The player in Daxter's mode keeps Jak's
  model, hidden, playing Daxter's clips: it writes Daxter's eyes into Jak's slot, and every other
  Jak on screen shows them. Skip `merc-eye-anim` for that model; a copy of Jak needs
  `(logior! (-> self skel status) (joint-control-status eye-anim))` to feed the slot.
- **Daxter on a lizard:** the stock rides him only in the desert chase (`'wild` mode, the lizard
  running on its own; `as-daxter?` is set by that mode alone, and its states play his clips
  through aliases such as `jakb-mech-pull-ja`). His mode ignores the lizard's
  `'change-mode 'flut` (its idle state, `flut.gc`); answer it in his stance's and walk's handler
  with `(go target-flut-start <handle> 'normal -1)` (common.md 1.4.19), and the lizard goes to
  its pickup as for Jak. For his clips in `'normal` mode, give the player a copy of Daxter's art
  group whose slots of Jak's lizard clips (`jakb-flut-*`, 406 to 432) hold his
  (`daxter-fldax-*`, 384 to 401), and do what `target-fldax-enter` does (`no-draw-bounds2`,
  sidekick `'matrix 'indax`). A basic starts 4 bytes before its address (its type): copy from
  there.

  ```lisp
  (let* ((dax (-> *target* sidekick 0 draw art-group))
         (size (+ 36 (* 4 (-> dax length))))
         (mem (malloc 'global size)))
    (mem-copy! mem (&+ (the-as pointer dax) -4) size)
    (let ((ag (the-as art-group (&+ mem 4))))
      (set! (-> ag data 406) (-> dax data 391)) ;; flut-idle: fldax-idle
      ;; ... one per clip
      (set! (-> *target* draw art-group) ag)))
  ```

  The lizard under him clones the player's clips by name (`joint-control-remap!`:
  `daxter-fldax-idle` gives `flut-saddle-fldax-idle`).
- **An art part linking to two masters:** `jak-fldax+0-ag` holds Daxter's clips
  (`daxter-fldax-*`, master `daxter-ag`, resident) and the lizard's under him
  (`flut-saddle-fldax-*`, master `flut-saddle-ag`, a level's). `link-art-to-master`
  (`engine/load/loader.gc`) links each clip to a master loaded when the part loads: put in GAME,
  the lizard's clips found none (*"could not find a master slot to link for"*) and the lizard
  under Daxter stayed hidden. Such a part goes in the level DGOs that carry the other master
  (WWD and VOCA list it after `jak-flut+0-ag.go`; DESLIZ already has it); a part whose clips all
  have a resident master can go in GAME (`jak-pidax+0-ag`, 4.9 above).

Verified: jak3, `goal_src/jak3/dgos/*.gd`, `engine/target/indax/target-indax.gc`,
`engine/target/target-handler.gc:1363`, `engine/target/target-pilot.gc`,
`engine/target/pilot-states.gc`, `engine/target/sidekick.gc`, `engine/target/target-gun.gc`,
`engine/target/gun/gun-util.gc`, `gun-*-shot.gc`, `engine/data/art-elts.gc`; in the
multiplayer mod's scenarios (`daxter-drive`, `daxter-drive-own`, `daxter-ride`, `coop`: Daxter
drives, rides in a passenger seat and sits on a shoulder held in `target-indax-grab`, 2026-10-16;
`daxter-fire`: Daxter fires each of the 12 mods, 10 kinds of blows landing on another player;
`ride-seat`, `ride-seat-daxter`: passengers in the drivers' clips, a Daxter passenger's upper
body as high as a Daxter driver's within 7 cm; `daxter-fire-anim`: the gun drawn in his hand,
opened, his clips holding it, screenshots, 2026-10-19; `eyes`, `daxter-flut`: the puppets' eye
slots, Daxter riding a lizard 22 m, firing from it, the lizard drawn under him on both machines,
`engine/gfx/foreground/eye.gc`, `engine/target/flut/target-flut.gc`, `engine/load/loader.gc`,
2026-10-20).

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

The same works for a `defbehavior` (a function with `self`): keep the stock one as
`(define-perm *stock* (function symbol symbol none :behavior target) target-death-reset)` and
redefine it with `defbehavior`; call `(*stock* arg0 arg1)` from it. `target-death-reset`
(`engine/target/target-death.gc`) ends every death: it sends `'target 'die` to the task managers
(a mission that fails on death fails there), then resets the game at the continue point
(`initialize!`). Not calling it, and moving the player (`move-to-point!`, health set, `(go
target-stance)`), brings the player back elsewhere with the mission still running.

Verified: jak3, `engine/target/target-death.gc`; the multiplayer mod's co-op return next to a
teammate (scenario `coop-respawn`: each player killed, back next to the other in `target-stance`,
health full), 2026-10-19.

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

The area weapons (wave concussor, mass inverter, needle lazer, blasts) do not collide: they fill
a list from the actor hash and keep the `process-focusable`s with a process mask among `crate
enemy guard vehicle civilian`, not `disable dead inactive`, and a `solid` or `semi-solid` root
prim (`pc-check-focus-fix`). A plain `process-drawable` is skipped. To be hit by them, make the
process a focusable of kind crate (no faction of the AI picks crates), offered to no search, with
an aim point that needs no draw-control:

```lisp
(deftype my-hurtbox (process-focusable) ())
(defmethod get-search-info-flag ((this my-hurtbox))
  (search-info-flag)) ;; neither the guns' aim nor an enemy picks it
(defmethod get-trans ((this my-hurtbox) (mode int))
  ;; the stock method reads (-> this draw status) for modes 2, 3 and 10
  (if (or (= mode 2) (= mode 3) (= mode 10))
      (the-as vector (-> this root root-prim prim-core))
      (-> this root trans)))
;; in its init: (logior! (-> self mask) (process-mask crate))
```

The mass inverter's blows come from a `gravity-spinner`, a child of the process it lifts, with
`'attack #f` and the shooter in the attack-info's `attacker`: damage 0 at the lift (`eco-dark`),
0.2 at the end (`gravity-end`).

Verified: jak3, `engine/target/gun/gun-red-shot.gc` (`find-targets-and-attack!`),
`engine/target/gun/gun-dark-shot.gc`, `engine/target/gun/gun-blue-shot.gc` (`gun-fire-blue-3`),
`engine/process-drawable/process-focusable.gc`; in the multiplayer mod, the four weapons hurt
another player through such a hurtbox, 2026-10-15.

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

A copy's own death effect: `(do-effect (-> skel effect) "death-default" 0.0 -1)` plays the
dissolve of characters, but the art of a vehicle (cars, bikes, barges: `vehicle` and its
children) has no `death-default`, and the call jumps to address 0: a crash when the copy of a
destroyed vehicle plays it. Skip it for vehicles; their explosion is their own effects.

Verified: jak3, seen in game in the multiplayer mod (`multiplayer-v1.0.0` crashed on the guest
when a barge its gyro burster or plasmite RPG destroyed died; fixed by the vehicle test),
2026-10-15.

Cloth (Jak's scarf, `:clothing` in `target-util.gc`): its strips are drawn apart from the model
and rebuilt by `cloth-on-skeleton` only while the owner is on screen. A copy moved far in one
step (a teleport, a player gone to another region) keeps a strip stretched from the old place to
the new one, across the world. Reset it with `(process-drawable-reset-all-cloth proc)` after a
jump, and empty its strips (`num-verts` 0) while the owner is off screen or `no-draw`.

Verified: jak3, `engine/physics/cloth.gc`, `engine/target/target-util.gc`; a stretched scarf seen
and gone in the multiplayer mod's screenshots, 2026-10-15.

A `manipy` of Jak's model (a preview in a menu) needs the room a player's model has: spawned with
`:stack-size #x10000` it fails to set up, with `#x20000` it draws. Give it the body segments of
the target (`setup-masks`), and no cloth (its scarf strips stretch away from a model placed in
front of the camera). To be seen under the START menu it must run there (a pool not masked
`progress`, 4.7) and be drawn in the 3D scene: a model drawn in the HUD goes through the generic
renderer, not Merc2 (no per-model flags, engine-internals 7b).

Verified: jak3, the multiplayer mod's Appearance page preview (`skin` scenario), 2026-10-19.

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
  so check `(type? (-> (string->symbol name) value) type)`. A copy of a player's shot fired on
  another machine runs its code there, and the gun shots' code reads the local player: the peace
  maker (`gun-dark-shot`) charges with that machine's R1 and vanishes unless its player holds a
  gun, homes on its player's target and takes `(-> *target* gun ...)`; the gyro burster's saucer
  (`gun-yellow-3-saucer`) drains that machine's ammo and fires real sub-shots. Send such shots as
  objects to draw (place, model, emitters) instead of firing them again. A vehicle's gun shells
  and shots are spawned under `*rigid-body-queue-manager*` with the vehicle as `owner-handle`,
  not under the player, and a shell's explosion is that shell's child: to find a player's shots,
  walk that manager's children too and follow owners up to the player.

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

- A particle's callback (`(:func 'name)` in a `defpart`, the `spt-func` init spec, read with
  `(get-field-spec-by-id launcher (sp-field-id spt-func))`) runs with the launching process in
  `(-> cpuinfo key proc)`. Replaying a group on a `part-tracker` changes that process: callbacks
  that read its parent's joints (`board-zap-track`: under `*entity-pool*` it read garbage, a crash),
  a collide-shape root (`sparticle-track-root-prim`, `eco-track-root-prim-fadeout`), joints of the
  process itself (`sparticle-track-joint-xz`, `sparticle-track-joint-glider`) or a given type
  (`sparticle-track-hadouken`) break, and those that read `*target*` (`sparticle-track-gun-joint`,
  `-player-y`, `part-tracker-track-target-joint`, `board-charge-track`, `sparticle-red-2-converge`)
  follow the local player. Spawn a replayed group under a process of the same skeleton as the
  original (`:to copy :target copy :mat-joint joint`), and skip groups with the others. The gun's
  laser (`gun-info-method-9`, `engine/target/gun/gun-util.gc`) is particles 207-212 launched with
  `draw-beam`, which orients them from globals (`*particle-quat*`): draw it again from the laser
  joint (14) and barrel (joint 13) of the gun's model rather than replaying the launches.

- Every precursor orb taken goes through `game-info` `give` (`engine/game/game-info.gc`) with
  `'skill`, the amount and the pickup's handle; that process's `entity` names the orb (its type:
  `(-> (the-as entity-actor ent) etype symbol)`, `'skill`; `entity-by-aid` finds it again, with
  its `trans`, on any machine with its level loaded).
- Every enemy death goes through `enemy` `mark-as-dead` (4.10); its killer is
  `(handle->process (-> this incoming attacker-handle))`. A Metal Head is an enemy whose
  `enemy-info` `gem-joint` is above 0 (it can carry a skull gem).
- A cutscene's actors are `(-> scene-player scene actor)`: `scene-actor`s with a `name` (Jak:
  `jakc-highres`, `jak-highres`, `darkjak-highres`, `jakc-scarf`; Daxter: `sidekick-highres`) and
  a `process` handle, 0 until the actor spawns (common.md 1.4.17 before `handle->process`).

All are functions or methods called through their symbol or method table, so a `define-perm` +
`defbehavior`, `defun` or `defmethod` wrapper in a GAME file catches every caller (see
[4.10](#410-wrapping-a-stock-method-from-a-game-file)). Orbs, Metal Heads and cutscene actors:
verified in the multiplayer mod's scenarios `economy`, `economy-trophy` and `scene-skins`,
2026-10-19.

Verified: jak3, the files above, `engine/gfx/sprite/particles/sparticle-launcher.gc`,
`engine/sound/gsound.gc`, `engine/target/gun/gun-dark-shot.gc`, `gun-yellow-shot.gc`,
`engine/target/board/board-part.gc`; seen in game in the multiplayer mod (shots, impacts, a
buggy's dust and engine sound replayed on another machine; task nodes closed on one machine closed
on the other; the hoverboard zap replayed under `*entity-pool*` crashed the other machine in
`board-zap-track`, no crash once parented to the player's copy), 2026-10-15.

## 4.14 Keeping world objects alike on several machines

- **Cycles.** An object on a cycle (moving platform, wheel, light) gets its phase from a
  `sync-info` (`engine/util/sync-info.gc`) as `(mod (current-time) period) + offset`, where
  `current-time` is the calling process's clock (`(-> pp clock)`, one of `(-> *display* clock i)`).
  Every method reads the `offset` field, so adding a constant to it for the duration of a call
  (and putting it back) shifts the phase without moving any clock or timer. `get-norm!` is
  overridden by `sync-linear`, `sync-eased` and `sync-paused`; `sync-now!` writes `offset`.
  To show another machine's phase, take that machine's clocks with the time it read them, and
  shift by its clocks at the time its objects are drawn here (its clock as estimated for drawing,
  not the time its clocks arrived): with the arrival time and a fixed draw delay, the cycles were
  30 to 34 ms off under 50 ms of delay and 30 of jitter, 2 to 3 ms with the drawing clock
  (multiplayer mod, `mp-apply-clocks`, 2026-10-15).
- **Doors.** A `com-airlock` (`engine/common-obs/airlock.gc`) opens when `want-cross-airlock?` says
  so, computed from `(target-pos 0)`: distance to `active-distance`, `check-crossing-distance`
  under `open-distance`, `y-height`, `latch-closed-time`, `(script-eval open-test)`, the
  `airlock` setting and `airlock-command-lookup`. Every door type of the game inherits it.
- **Elevators.** An `elevator` (`engine/common-obs/elevator.gc`) in its `waiting` or `arrived` state
  takes `(send-event elev 'move-to 'top)`, `'bottom` or a path position and runs there;
  `(-> elev move-pos 1)` is the stop it runs to, `bottom-top` its two ends. In `waiting` it has
  `(process-mask actor-pause)`: out of view it does not run, and a `'move-to` received then only
  sets its next state (its `running` enter, which clears the mask, never runs): clear the mask
  first. Its entity scripts (`on-activate`, `on-deactivate`, `on-up`, `on-down`, `on-running`,
  `on-notice`, `on-wait`, pairs in the elevator) load levels and change the continue point, the
  camera and the player's settings, and some do not check `player-ridden?`: Spargus' palace
  elevator arriving runs `(want-vis 'waspala) (want-continue "waspala-start")`. Set them aside
  (`#f`) for a trip the local player is not on. To show another machine's trip, place it: clear
  `(elevator-status waiting-to-ascend)` (its `running` post then stops seeking), set `path-pos`,
  `move-pos` and `basetrans` with `get-point-in-path!` at
  `(+ from (* (ease-value-in-out path-pos 0.08) (- to from)))`; its post moves the platform and
  its riders; `path-pos` 1.0 makes it arrive.
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

Verified: jak3, the files above, `levels/volcano/volcano-obs2.gc`, the palace elevator's res
(`decompiler_out/jak3/entities/wascitya-actors.json`); seen in game in the multiplayer mod (a test
cycle within one frame of the host's, a door opened for a remote player 200 m away, an elevator
trip and a Spargus cutscene started on two machines, lava platforms within 7 cm of the
authority's and carrying the player standing on them; the palace elevator still at the bottom on
a guest 25 m away until `actor-pause` was cleared, then rising with the host's, the guest's
continue point unchanged), 2026-10-15.

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

## 4.16 Drawing the objects of another place, outside the banks

How to draw objects of another place next to the local one without taking a bank, and why the
place's own levels must not be loaded (the multiplayer mod tried, then kept the art alone:
`goal_src/jak3/pc/features/multiplayer/multiplayer-world.gc`, "Art of distant players'
objects"). Each step meets a trap of the engine:

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
- **Code: do not.** A level's code is global: its `defun`, `define` and `deftype` of names the
  local levels use point those names at its copy, a `deftype` of an existing type resets that
  type's methods to the parent's (`new_type` in `game/kernel/jak3/kscheme.cpp`), and its top-level
  forms run: Haven's `traffic-manager.gc` (in `cwi.gd`) runs `(define *traffic-engine* #f)`, so
  loading `ctywide` next to Spargus leaves Spargus' traffic without its engine and the next
  citizen it spawns crashes the game (seen in testing: "traffic-entity-hack wlander-male", then
  an access violation). Putting back the values that point into the new level's memory misses
  such a `#f`, and putting back every changed value would undo the local game's own changes made
  in the frames the level links. Load the art alone instead (below): a model needs no code.
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
`engine/anim/joint.gc`, `game/kernel/jak3/kscheme.cpp`, `levels/city/traffic/traffic-manager.gc`;
in game with the multiplayer mod (`far`: `wascityb` loaded in `wascitya`'s heap and back;
`far-city`: `ctywide`, `ctypesa`, `ctycara`, `ctycarb` loaded in Spargus, Haven's guards and car
drawn there, then `barge-ag` and `citizen-chick-ag` loaded alone with `ctyport` and `ctypepa` kept
by the renderer; then the guest crash from `ctywide`'s code in Spargus, and the art loaded alone
into a heap carved from an active level instead, no level loaded: no crash), 2026-10-15.

## 4.17 A game loaded from memory, and saves turned off

- **A save in memory.** `(new 'global 'game-save #x1e000)` is a save as large as a memory card's
  (80 bytes of header, then `#x1e000` of tags). `(save-game *game-info* save "name")` (a
  `game-info` method, `engine/game/game-save.gc`; call it from a process) fills it with the whole
  game: progress (task nodes), features, items, the entity perms of the active levels, the
  checkpoint (`current-continue`), and the player's settings. It is `(+ (-> save type size)
  (-> save length))` bytes from `(&-> save type)`, about 43 KB early in the game; send or copy
  those, and on the receiving side set `(-> save type)` to `game-save` again (the stock restore
  does the same) and check `(-> save version)` is 4.
- **Loading it.** As the stock Load Game does after reading the card: `(set! (-> *game-info*
  mode) 'play)`, `(initialize! *game-info* 'game save (the-as string #f) (the-as resetter-spec
  #f))`, `(set-master-mode 'game)`. It works in game and from the title screen: the target is
  killed, actors are reset, a loading screen (`target-continue`) shows while the checkpoint's
  levels load, and `load-game` applies the save one frame later (keep the buffer). Processes in
  `*pc-pool*` survive (`reset-actors` only kills entity and pusher pool processes). It also works
  from the per-frame hook with a menu open (the multiplayer lobby at the title screen), where every
  process is paused.
- **The player's own settings.** `load-game` also sets the language, subtitles, screen position,
  camera, hints, brightness and contrast from the save's tags. To keep the local ones, rewrite
  those tags' `elt-type` to `(game-save-elt sfx-volume)`, which `load-game` skips (the PC port
  keeps volumes in `pc-settings`). Tags follow each other at
  `(logand -16 (+ (* elt-size elt-count) 31))` bytes.
- **No save.** Every card write goes through `auto-save-command` (`'save`, `'save-last`,
  `'auto-save`, `'create-file`, `'format-card`, `'unformat-card`; `'restore` reads): the pause
  menu's saves, checkpoints, missions' `(auto-save 'auto-save)`, a new game's first save. A
  `define-perm` wrapper that returns `#f` for those turns saving off; tell the menu that asked
  with `(send-event menu 'notify 'error (mc-status-code no-process))`, or its "Saving" page waits.
- **Title screen or not.** `(= (level-status? *level* 'title #f) 'active)` while the title screen
  is up (`*game-info*` `mode` is `'play` there too).
- **A new game somewhere, or at a mission.** `(initialize! *game-info* 'game (the-as game-save #f)
  "<continue name>" spec)` then `(set-master-mode 'game)` (the menu closes by itself): with `spec`
  a `resetter-spec` whose `node` is a task's `play-node` and `continue` its `play-continue`
  (`*game-info* play-list`), the game starts at that mission with the nodes it needs closed, as
  the stock mission list's `play-task` does. The stock New Game uses `"intro-start"`.
- **The whole story done:** `(task-node-close! (game-task-node city-win-resolution) 'event)` runs
  that node's `open!`, which closes every node it needs first (254 nodes, with their rewards).
  The rest (gun upgrades, vehicles: `(-> *game-info* vehicles)`, secrets) is set by hand.
- **Daxter off Jak's shoulder without touching the saved features:** the sidekick exists while
  `game features` and `(-> *setting-control* user-current features)` both have bit 20; a process
  holding `(set-setting! 'features 'clear 0 #x100000)` takes him off (`target-death.gc` does it),
  `(remove-setting! 'features)` gives him back.

Verified: jak3, `engine/game/game-save.gc`, `engine/game/game-info.gc`,
`engine/ui/progress/progress.gc`, `engine/game/task/task-control.gc`, `engine/target/sidekick.gc`,
`engine/target/logic-target.gc`; in the multiplayer mod's `coop-campaign` and `lobby` scenarios
(a mission started with 4 nodes closed, the story closed with 254, the sidekick off and back),
2026-10-19; in game with the multiplayer mod (`join-world`: a guest in
Haven loaded the host's 42928-byte save sent over the network, came out at the host's Spargus
checkpoint with the host's progress, its language kept, an auto-save refused; `join-title`: the
same from the title screen), 2026-10-15.

## 4.18 What the game removes out of its player's view

The city's NPCs leave when the local player does not see them, which another viewer of the same
world (a second player, a spectator camera) can see happen:

- **Traffic vehicles** (`vehicle-flag traffic-managed`, not `persistent`), every frame
  (`levels/city/traffic/vehicle/vehicle.gc`), with `d` the nearer of the player and the camera:
  farther than 200 m, back to the traffic pool (`vehicle-method-109`); between 20 and 200 m and
  off screen (`draw-control-status on-screen` clear), back to the pool after 10 s off screen or
  at once beyond 100 m. On screen within 200 m, a vehicle stays.
- **Citizens** (`citizen` `enemy-common-post`, `levels/city/traffic/citizen/citizen.gc`): not in
  a mission, off screen or farther than 110 m (xz) from the camera, and their branch's territory
  not wanted by `*cty-faction-manager*` (methods 23 and 9): `go-inactive`. The inactive state
  sets `focus-status inactive`, `no-draw` and turns collision off; the process lives on.
- **Metal head flitters** go inactive on entering water deeper than 2.8 m below its surface
  (`in-ditch`, `check-water`), a fall from a quay included.
- **Crimson guards** off screen, not in formation nor in a mission, can be retired by their
  squad (`crimson-guard-method-270`).

A citizen that goes inactive is still a process: test `focus-status inactive` or `no-draw`, not
whether its handle resolves, to tell it vanished.

Beyond that, every process with the `actor-pause` mask stops running (trans, code, post) farther
than `(-> *ACTOR-bank* pause-dist)` (50 m) plus its root's `pause-adjust-distance` from the camera:
`run-logic?` of `process-drawable` (`engine/entity/entity.gc`), and of `enemy`, which also pauses
an idle enemy off screen (`engine/ai/enemy.gc`). The kernel asks it each frame, so wrapping both
methods (4.10) keeps the actors near another viewpoint running.

Verified: jak3, the files above; seen in the multiplayer mod's tests (flitters going inactive in
the port's water, copies hidden when the host walked away), 2026-10-15.

## 4.19 A clip on part of a skeleton

A `joint-control` computes a frame from its channels with its `generate-frame-function`
(`create-interpolated-joint-animation-frame` by default). With
`create-interpolated2-joint-animation-frame` (`engine/anim/joint.gc`), it computes the frame twice,
the blend channels weighted by `frame-interp 0` then by `frame-interp 1`, and each joint takes the
second when its bit is set in `interp-select` (bit = joint index in the skeleton: 0 align, 1
prejoint, 2 main...; joints 64 and up in `interp-select 1`). With both masks 0 it runs the plain
computation, so leaving the function set costs nothing. `top-anim` (the gun's clips on Jak's arms)
works this way.

To play a clip on chosen joints only, append a blend channel after the clips the skeleton holds:
`parent` the joint-control, `command` `(joint-control-command blend)`, the clip in `frame-group`,
`num-func-identity`, `frame-num`, `frame-interp 0` 0.0 and `frame-interp 1` 1.0; increment
`active-channels` (below `allocated-length`), set `interp-select`, and the function above. A puppet
can use `(ja-channel-set! 2)` and `(ja :chan 1 :group! clip :num! (identity f) :frame-interp0 0.0
:frame-interp1 1.0)`.

The sidekick (Daxter's model) copies the player's channels every frame, and turns the player's
`top-anim` channels into full-body blends (`frame-interp 0` set to `frame-interp 1`): a clip for
part of Daxter goes after that copy. `joint-control-remap!` is called right after it in the
sidekick's post, with the sidekick's joint-control: a `define-perm` wrapper can append the channel
there. Daxter's upper body (`skel-sidekick`): upper_body 3 to the gun joint 27 (head, ears, arms,
hands; the gun joint's parent is upper_body) and the thumbs 44-47, `#xf0000ffffff8`; hips 28 and
the legs and tail under it.

Verified: jak3, `engine/anim/joint.gc`, `engine/process-drawable/process-drawable.gc` (`top-anim`),
`engine/target/sidekick.gc`; the multiplayer mod's `daxter-fire-anim` scenario (Daxter's upper body
in his gun clips over his mode's legs, screenshots; without the function, the channel had no
effect), 2026-10-19.

## 4.20 The order processes run in

`change-parent` (`kernel/gkernel.gc`) puts a process first among its parent's children, and a
tree runs a parent, then its children from first to last: the latest child runs first. The pools
are made in that order under `*active-pool*`: `*pc-pool*`, `*camera-pool*`, `*target-pool*`,
`*entity-pool*`, `*pusher-pool*`, `*bg-pool*`, `*default-pool*`, so they run the other way round,
`*pc-pool*` last. A process in `*pc-pool*` runs after the player and its children (gun,
sidekick) every frame, before anything is drawn: it can undo what their posts set.

To run before every pool, make a pool of your own with `change-parent` under `*active-pool*`
after boot: it becomes the first child. Make it with `define-perm` so a reload keeps one:

```lisp
(define-perm *my-first-pool* process-tree
  (let ((pool (new 'global 'process-tree "my-first-pool")))
    (change-parent pool *active-pool*)
    (set! (-> pool mask) (process-mask process-tree))
    pool))
```

The display process runs before the pools: it runs the per-frame hook (`display-loop-main`,
`engine/game/main.gc`), ends the frame, decides the pause (`determine-pause-mode`), reads the
pads (`service-cpads`, `display-frame-start`, `engine/draw/drawable.gc`), then its children, the
START menu (`*progress-process*`) among them. What the first pool does to pad 0 is seen by every
process after it, and the hook of the next frame runs before the menu reads the pads again.

Verified: jak3, `kernel/gkernel.gc` (`change-parent`, `execute-process-tree`, the pools); the
multiplayer mod's manager in `*pc-pool*` showing the gun the gun's post hid (`daxter-fire-anim`;
from the sidekick's post it did not work: the sidekick runs before the gun), 2026-10-19; the
mod's input pool hiding pad 0 from the world under the START menu (`pause` scenario: the stick
pushed with the menu open does not move the player, the menu still reads it), 2026-10-19.

## 4.21 How missions check the player: waypoints and races

- **Waypoints.** The yellow beam of a mission is a `task-arrow` (`engine/game/task/task-arrow.gc`,
  spawned with `task-arrow-spawn`, hidden while `taf4` is set). It only draws: the process that
  spawned it (its `parent`, a task manager or an object of the mission) checks the arrival itself,
  almost always as `(< (vector-vector-xz-distance (target-pos 0) <spot>) <radius>)`, radii of 3 to
  40 m. A wrapper of `target-pos` that answers that process alone (`with-pp`, compare `pp` with
  the arrow's parent) holds the mission without touching it; keep the stock function with
  `define-perm`. Missions checking the vehicle or a cutscene region do not call it.
- **Race continue points** (`continue-flags race`, `vehicle-type` set): `arrive-at-continue`
  (`engine/target/target-death.gc`) stores `mode-cache 'pilot`, and the end of `target-continue`
  sends `'change-mode 'pilot`, which spawns the vehicle where the target stands then: moving
  `(-> *target* control trans)` while `mode-cache` is `'pilot` moves the vehicle. The start of
  `desertb-race-start` is on a slope: the turtle rolls about 30 m in 20 s with no input.
- **Race starts.** `task-manager-race` (`levels/desert/race/course-race.gc`: Kleiver's race, the
  beast hunters' time trial and rally) and the artifact race's manager
  (`levels/desert/artifact-race/artifact-race.gc`: Damas's races) set `pilot-exit #f` and start
  the countdown from their `:code`; a `(while <test> (suspend))` before it (after the
  `ai-set-target-speed 0` send, after `wasall-kill-duplicate-vehicle`) holds the start. Before it,
  `task-manager-race` hands the player's vehicle to its AI (`'set-control-hook-ai`, target speed
  0): the vehicle stays still at the line, the player cannot drive it until `'force-start`. Every
  machine runs its own race manager (one per task, spawned where the player is).
- **The arena's jump training** (`task-manager-arena-training`, `wasstadb-obs.gc`): the first beam
  (`wait-touch`) is passed by touching the judge (`judge` `wait` state, a `'touch` event), not by
  `target-pos`; the beam of its last step (`done`) is checked in `:trans` with
  `(target-pos 0)` within 3 m.
- **Taking a task manager to a later step** (a test): `(method-state (-> proc type) 'done)` finds a
  virtual state by name, `(go-process proc (the-as (state process) st))` sends it there; without
  the cast, the compiler stops on "Attempting to go to a state with incomplete type".
- **When a mission starts.** A mission runs while its task manager lives: its node's `on-open`
  script spawns it (`engine/util/script.gc`, `run-now-in-process` with the symbol
  `task-manager-init-by-other`) when the node opens and its level is active, or
  `update-task-masks` (on a level load) once the level is there. Wrapping
  `task-manager-init-by-other` (`define-perm` the stock, call it last: it never returns) sees
  every mission start. START > Restart is `restart-mission` (`(function int)`,
  `task-control.gc`).
- **Places and result of a race** (`levels/common/race/race-manager.gc`, in WASLEAPR, DESRALLY,
  DESTRACK): the manager's `active` post sets `(-> *game-info* race-position)` from the player's
  `racer-state` `rank` (0 = 1st; the HUD has sprites 1st to 8th, `hud-race-position`), and once
  the player finished (`racer-state-flags rsf1`) goes `win` or `lose` on `player-win?`:
  `(zero? (-> racer finish-count))` (1st), or for a time trial (`racer-info-flag rif6`) a new
  score. `update-rankings` sorts the racers by `lap-count` plus `lap-distance` (a part of the lap),
  finished ones by finish order; `finish-time` and `race-start-time` count `current-time` units.
  Damas's races (`artifact-race.gc`, DESRACE1-2) have no ranking.

Verified: jak3, `engine/game/task/task-arrow.gc`, `engine/target/target-death.gc`,
`levels/desert/race/course-race.gc`, `levels/desert/artifact-race/artifact-race.gc`; the
multiplayer mod's co-op waypoints and race starts (`beam`, `race-board`, `race-continue`
scenarios), 2026-10-19; Kleiver's race and the arena training played in co-op, the arena's manager
sent to `done` (`race-real`, `race-real-daxter`, `beam-real`), the intro skipped in a retail boot
(`retail-general`: the title menu after 6 s instead of 274 s), 2026-10-19;
`levels/common/race/race-manager.gc`, `race-hud.gc`, `engine/game/task/task-control.gc`,
`engine/util/script.gc`; the mod's `mission-start`, `race-result`, `race-board` scenarios,
2026-10-20.
