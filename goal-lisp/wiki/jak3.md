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

**Endless dark and light eco refill every frame.** `target-eco-process` (`powerups.gc`) sets
`(-> *game-info* eco-pill-dark)` to its maximum (100) each frame while `endless-dark` is set, and
`eco-pill-light` while `endless-light` is set. A mod that spends or converts that eco sees it
come back at once: clear the bit for the mod's time and set it back after.

```lisp
(set! saved? (logtest? (-> *game-info* secrets) (game-secrets endless-dark)))
(logclear! (-> *game-info* secrets) (game-secrets endless-dark))
;; ... later
(if saved? (logior! (-> *game-info* secrets) (game-secrets endless-dark)))
```

Verified: jak3, `engine/common-obs/powerups.gc` (`target-eco-process`); the battlefront mod's
runs on a 100 % save with the secret on: dark eco back at 100 every frame, jetpack fuel stuck
full until the mod cleared the bit, 2026-10-31.

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

**Which mods a player has.** A mod is offered (the D-pad's choice, the HUD, `target-gun.gc`
picking the next gun type) when its `game-feature` bit is in both `(-> *game-info* features)` and
`(-> *setting-control* user-current features)`. The twelve mods, pickup-type `gun-red-1` (26) to
`gun-dark-3` (37), are game-feature bits 6 to 17: `(ash 1 (+ type -20))`. To keep mods from a
player, clear their bits from `game-info` `features` every frame (a respawn or a loaded save puts
the saved features back), and set `(-> *game-info* gun-type)` back to a mod it still has (the
first of its color is `(+ 26 (* 3 (/ (+ type -26) 3)))`).

Verified: jak3, `engine/game/settings-h.gc` (`game-feature`), `engine/game/game-info-h.gc`
(`pickup-type`), `engine/target/target-gun.gc`; the multiplayer mod's free roam, each color at
its first mod and the others bought (`free-shop` scenario, two machines), 2026-10-29.

**Ammo and its pickups.** `(-> *game-info* gun-ammo i)` holds the ammo, 0 yellow, 1 red, 2 blue,
3 dark, capped by `get-max-ammo-for-gun` (100, 50, 100 and 5 before upgrades). The stock pickups
refill it: from a process, `(birth-pickup-at-point pos (pickup-type ammo-red) 1.0 #t *entity-pool*
(the-as fact-info #f))` spawns one `ammo-collectable`, worth 10 (5 for red, 1 for dark), and
`(pickup-type health)` with 1.0 one health pickup worth 1. `verify-pickup-type` turns red, blue
and dark into yellow when the player lacks that gun. A mod's own weapons can spend these counters
and get the pickups for free. To keep a red or blue pickup whatever guns the save has, wrap the
function in GAME: keep the stock one with `(define-perm *stock-verify* (function pickup-type
pickup-type) verify-pickup-type)`, then `defun verify-pickup-type` again, returning its argument
unchanged for those types while the mod runs and calling the stock one otherwise.

**Pickups that vanish.** A collectable fades (1 s, its `fade` state) once `fadeout-timeout` has
passed since its birth, but only while Jak is over 50 m away, unless it carries
`no-distance-check-fadeout`. Pass `birth-pickup-at-point` a static fact-info as its last argument,
`(new 'static 'fact-info :options (actor-option fade-out no-distance-check-fadeout) :fade-time
(seconds 10))`, its `process` left 0 (a nonzero one has its entity read): eco, health and dark eco
pills take that time. An ammo pickup keeps a fixed 15 s (`initialize-ammo-by-other`), so set
`(-> (the-as collectable (-> pp 0)) fadeout-timeout)` on the `(pointer process)` the call returns
(the last one born).

Verified: jak3, `engine/game/game-info.gc`, `engine/common-obs/collectables.gc`
(`birth-pickup-at-point`, `verify-pickup-type`, `initialize-options`, the `wait` and `fade`
states); the battlefront mod's in-game runs (up to 12 pickups left by dead units, counters spent by
its class weapons, 2026-10-31; a blue ammo pickup born blue under the wrapper on a save without the
blue gun, 2026-11-11; an ammo pickup 15 m from Jak there at 9 s and gone at 12.5 s with
the fact and the timeout set, still there at 11 s with the fact alone, 2026-11-09).

**Free aim through the camera.** The ray behind a fixed crosshair at screen point (x, y) of the
512 x 416 screen: project the spot 1 m ahead of the camera and the spots 1 m to its right and
above it (`transform-point-vector!`, origin (1792, 1840), as in 4.31); the direction is `fvec` plus
`rvec` times `(x + 1792 - s0.x) / (sr.x - s0.x)` plus `uvec` times `(s0.y - (y + 1840)) / (s0.y -
su.y)`. Cast it with a `collide-query` (`start-pos` the camera's `trans`, `move-dist` the
direction at the reach, `collide-with` `backgnd civilian enemy obstacle hit-by-others-list
vehicle-sphere bot`, `ignore-process0` `*target*`) and `fill-and-probe-using-line-sphere`; the
unit met is `(-> (as-type (-> cq best-other-tri collide-ptr) collide-shape-prim) cshape process)`.
The hit point projected back lands on the crosshair within 0.07 screen units.

**Pointing Jak's gun elsewhere.** The gun post (`gun-util.gc`) computes `fire-dir` and
`fire-dir-out` of `(-> *target* gun)`, then calls `target-gun-check`, which fires along them.
Rewriting both (unit vectors) from a function called just before that call turns every shot; a
`define-extern`'d mod function there is a two-line hook.

Verified: jak3, `engine/target/gun/gun-util.gc`, `engine/collide/collide-cache.gc`; the
battlefront mod's runs: rifle shots and grenades on the crosshair's point (a grenade 0.73 m from
it), Jak's plain blaster shot flying straight at it (dot 1.000), 2026-10-31.

## 4.5 Custom art-groups: no dynamic linking hook

Unlike Jak 1 and Jak 2 ([1.2.8](common.md#128-custom-art-groups--dynamic-animation-linking-link-art)),
Jak 3's engine code has no `register-custom-art-group` / `link-art!` /
`needs-link?` hook. Build a self-contained art-group instead
(`build-actor` without `:master-art-group`), or attach the entity through
`target-set-lod`-style skeleton swapping if it must ride on Jak/Daxter's
own skeleton (see [assets.md](assets.md) for the general asset
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

The same goes for any `(new 'process ...)` object a field of the process points to, such as a
`lightning-control` (crimson-guard's relocate moves its `l-control`). The object's own pointers
(the lightning arrays) are relocated by its own method; the field is the process's job. The
battlefront mod's avatar kept the guards' tazer arc in a field without moving it: its first
tazer shot read the old address and crashed the game (access violation in the avatar's code),
and no longer did with the method.

Verified: jak3, `levels/city/traffic/citizen/guard.gc` (crimson-guard `relocate`); a crash
reporter's stack, then the battlefront mod's runs, 2026-10-31.

A `joint-mod` made in a process's init (`(new 'process 'joint-mod (joint-mod-mode joint-set*) self
13)`) needs the same: `gun`'s relocate moves its `mag` fields (`engine/target/gun/gun-util.gc`).
The kernel's heap walk relocates the joint-mod's own pointers (`process`, `joint`) and
`cspace-array`'s relocate moves the cspace's `param1` onto it (`engine/entity/relocate.gc`), so the
joint keeps working; only the process's field stays behind. Writing a scale through it each frame
crashed the game a few seconds after the battlefront mod's guard class spawned (access violation,
no crash report), at the first compaction, not at the spawn.

Verified: jak3, `engine/entity/relocate.gc`, `engine/target/gun/gun-util.gc`; the battlefront
mod's runs, a crash with the field left out and none with it, 2026-11-06.

The LZO cache of animations (`*anim-manager*`, `art-joint-anim-manager`, `engine/anim/joint.gc`:
64 slots in #x30000 bytes) frees its oldest slot when full, even one used this frame. Jak 3
compresses few animations that way (`frames flags` 1): a two-player session with Haven's traffic
decompressed 2 in all, 1 slot used at most. Jak 2 compresses many more (himstan's Jak 2 mod had to
grow it for its player replicas).

Verified: jak3, `engine/anim/joint.gc`, `engine/process-drawable/process-drawable.gc`; a probe on
`decompress` in the multiplayer mod's `tints` scenario, 2026-10-28.

## 4.7 Start menu: adding a page

The ring's Load Game and Save Game are `menu-main-menu-option`s named `progress-load-save` and
`progress-save-game` (`*main-options*`). A `define-perm` wrapper of their `respond-progress` that
takes `confirm` first, while the ring stands still (`(= (-> progress ring-angle) (-> progress
ring-want-angle))`, as the stock does), keeps their memory card pages from opening: clear the
button (`(cpad-clear! 0 confirm)`), act, return 0; call the stock for everything else.
Verified: jak3, `engine/ui/progress/progress.gc`, `progress-static.gc`; the multiplayer mod's
saves online (`coop-save` scenario), 2026-10-29.

The ring of the START menu (`*main-options*`) has exactly ten slots: `ring-want-angle` is
`36 degrees * option-index` (`engine/ui/progress/progress.gc:1246`) and the ring model has ten
segments. An eleventh entry does not fit. Hang a page off a list page instead, such as
START > Options (`*options-options-pc*`, `pc/progress/progress-static-pc.gc`), whose
`menu-sub-menu-option` entries are placed by `offset-y`, a fraction of the page height
(stock: 0.1 to 0.8, step 0.175). Swap its `options` array for one with your entry and spread the
offsets over the same span; restore both to remove it.

START opens the menu through `(activate-progress *dproc* 'main)` (`engine/game/main.gc`, master
mode `'progress`; when no menu process comes out, it sets `'game` back). A `define-perm` of the
stock function and a `defun activate-progress` of the same signature (`(function process symbol
none)`) can open a page of your own instead for `'main` and hand every other state (save icons,
card errors) to the stock one; let a flag through to reach the stock menu from your page. Your
page's own START press can come back through the game's menu once the page closes: ignore a new
request for 0.4 s, timed on `(-> *display* real-clock frame-counter)`, since a process's own clock
stops under the pause (a guard timed with `time-elapsed?` never ran out there).
Verified: jak3, `engine/game/main.gc`, `engine/ui/progress/progress.gc`; the battlefront mod's round 20 runs, 2026-11-09.

The Quit / Restart page (`*quit-restart-options*`) holds a `menu-restart-mission-qr-option` and a
`menu-quit-qr-option`, both handled by their parent's `respond-progress` (`menu-qr-option`,
`engine/ui/progress/progress.gc`). A `defmethod respond-progress` on the child type alone, its
`define-perm` stock the parent's method, refuses Restart Mission (clear `confirm`, beep, return 0)
and leaves Quit as it is. Verified: jak3, `engine/ui/progress/progress-h.gc`, `progress-static.gc`,
`progress.gc`; compiled in the battlefront mod's round 26, 2026-11-12.


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
(English is loaded as the fallback for every language). A mod that keeps those files as they are
can answer ids past the enum's last (`#x136c`) itself: every label goes through `lookup-text!` on
`*common-text*`, so a `define-perm` wrapper of `game-text-info` `lookup-text!` (4.10) returns its own
string for its range, `pc-encode-utf8-string`-encoded into a string of its own per label (two
labels can be drawn from one format), and calls the stock for the rest. In `new 'static` option
pages give the id as a plain integer (`:name #x7f00`, or a `defconstant`):
`(the-as text-id #x7f00)` stops the compiler ("Could not evaluate ... at compile time"). That English fallback goes into
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
`call-on-confirm` when "no" must do nothing. A button that closes the START menu calls
`(set-next-state progress 'go-away 0)`; a mod's own process then waits for `*master-mode*` `'game`
before opening its own page. The generic pages read no input while the menu still moves
(`menu-transition` not 0.0, `respond-to-cpad`, `progress-generic-pc.gc`), and keep the hovered line
of the last visit (`*progress-pc-generic-store*` `current-menu-hover-index`): a test that presses
down counts from there, not from the top.

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
  copy it back from the per-frame hook, which runs before the menu reads it. Empty the PC port's
  high-fps button history too, `(-> *cpad-history-list* cpads 0)` (`button0-abs` and
  `button0-rel`, 15 entries each): `service-cpads` fills it before any process runs, and the
  buffered checks read it (`recently-pressed?`, `engine/ps2/pad.gc`: the jump). With only
  `cpad-invalid!`, R1 and square were blocked but X choosing a menu entry made the player jump.
  Key the hiding on `*progress-process*` (the menu alive), not on `*master-mode*` `'progress`.
  Verified: jak3, the multiplayer mod's `menu-buttons` scenario (X jumped before, not after),
  2026-10-20.
- **A model drawn over the menu** (a preview on a page) must carry `draw-control-status hud`
  (bit 14): while the still copy shows (menu mode, as at the title screen), the foreground only
  draws the draw-controls with that status. Turning the menu mode off instead brought back
  buckets with no renderer (`tex-lcom-pris2`, bucket 456) and stopped the game on an assertion.
  `(if (get-menu-mode *blit-displays-work*) (logior! (-> proc draw status) (draw-control-status
  hud)) (logclear! (-> proc draw status) (draw-control-status hud)))` each frame.
- **Text from the per-frame hook:** `print-game-text` adds its own DMA bucket. Called inside a
  `with-dma-buffer-add-bucket` block it nests two buckets and the game stops on the `!m_ended`
  assertion: call it after the block closes.
- **Button icons are long.** `lookup-text!` returns a `<PAD_...>` icon expanded into its drawing
  codes (`common/util/font/dbs/font_db_jak3.cpp`): about 40 characters for `<PAD_X>`, about 110
  for `<PAD_DPAD_UP>`. A line formatted into a 64-character string stopped right after the D-pad
  icon; give such lines a 256-character string. Jak 3's font has icons for the face buttons, L1,
  L2, R1, R2 and the D-pad, none for L3 or R3. Verified: the multiplayer mod's challenge
  invitation (`challenge` scenario screenshots), 2026-10-27.
- **No left or right D-pad icon, and the down one lights up too.** The font has `<PAD_DPAD_UP>`,
  `<PAD_DPAD_DOWN>` and `<PAD_DPAD_ANY>`, each the four `<PAD_PART_DPAD_*>` parts drawn with `~3L`
  (lit) or `~22L` (dim) colors (`font_db_jak3.cpp`); its `<PAD_DPAD_DOWN>` lights both the up and
  the down part. Write the left arrow, and a down arrow alone, the same way, the part you want lit:
  `~Y~3L<PAD_PART_DPAD_L>~Z~22L~+17H~-13V<PAD_PART_DPAD_U>~Z~22L~+17H~+14V<PAD_PART_DPAD_D>~Z~22L~+32H<PAD_PART_DPAD_R>~Z~+56H`
  (left), the same with `~22L` before the left part and `~3L` before the down part (down). The
  icon stands taller than a 17-unit line at scale 0.36: give lines under D-pad icons 22 units.
  Verified: the battlefront mod's class help, drawn through `print-game-text` (screenshots),
  2026-11-06 and 2026-11-11.
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

**Greying an option of the ring.** Only the selected option's name is drawn, by
`menu-main-menu-option` `draw-option` (`pc/progress/progress-draw-pc.gc`, color `font-color-32`);
the ring has no disabled state. A `define-perm` wrapper of `draw-option` that draws the stock label
itself for that option with `font-color-44` (the PC generic pages' disabled color,
`progress-generic-draw-pc.gc`) and calls the stock for the rest greys it; pair it with the
`respond-progress` wrapper above, playing the stock refusal beep `(sound-play "generic-beep"
:pitch -0.6)`.

Verified: jak3, `pc/progress/progress-draw-pc.gc`, `engine/ui/progress/progress.gc`; the
battlefront mod's Save during a battle ("Save Game" grey, X left the ring as it was), 2026-11-01.

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
`(send-event *target* 'draw #f)` (`no-draw`: his bones stop too, see 4.34 before moving him while
hidden). Only the position follows: the player keeps the facing it had when it got in (`control
quat`).

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

- **Modes his handler refuses:** `target-indax-handler` takes only `'change-mode` `grab`,
  `normal`, `falling`, `hang`, `tube` (and `indax`): a mission's turret (`'turret`), vehicle,
  torpedo (`'pilot-daxter`), bomb robot (`'grab-ride`) or mech then waits for good. Forward those
  to `target-standard-event-handler` (it leaves his mode through `target-indax-exit`), and only
  those: the stock asks for `'gun` in any mode (`target-gun.gc`: the D-pad's mod choice and
  `want-to-gun?` send `'change-mode 'gun` to the player), which his mode must keep ignoring.
  `'turret` takes the player only while `on-surface` (`target-handler.gc`), and an idle turret
  shoves whoever touches it.
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
  `target-death.gc` does), without touching the saved features. Stock code that has Daxter act
  for Jak reads him without a test, and crashes without him: `gun-post` (`gun-util.gc`) reads his
  hand when `gun gun-daxter` is 1, which `target-gun-joint-points` sets by Daxter's firing clip in
  a vehicle (the glider of `desert-glide` at once); `target-pilot-pidax-enter`
  (`target-pilot.gc`) draws the player with his art for `'change-mode 'pilot-daxter` (the port
  attack's torpedo). Keep `gun-daxter` at 0 without him, and bring him back first
  (`target-sidekick-setup #t`, after removing the setting and `apply-settings`, or logic-target's
  own check removes him again within the frame).
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

**The whole death replaced: a state's code.** `target-death-reset` comes last: before it the
stock death freezes enemies, platforms, projectiles and guards (a `'process-mask` setting) and may
play a death movie (`target-death-main`). A state is an object whose `code` slot is called with the
`go` arguments, so keep the stock code, `(define-perm *stock* function (-> target-death code))`, and
`(set! (-> target-death code) my-code)`, a `defbehavior` of `target` taking the death mode, which
calls `((the-as (function symbol none :behavior target) *stock*) mode)` when it does not apply. To
die without the freeze, set what the stock death sets that the game reads, `(focus-status dead)`
and `(target-flags disable-attacks)` (attacks are refused while it is on), put the gun away, wait
(`suspend` in a loop: the world runs), then `move-to-point!`, health, and `(go target-stance)`: the
state's `:exit` clears the flags. Meanwhile `(send-event *camera* 'change-target proc)` gives the
camera to any `process-focusable` (cam-master.gc, as `camera-look-at` does) and `*target*` gives
it back; a unit taller than Jak fills the screen behind the default string, so set
`'string-min-length`, `'string-max-length`, `'string-min-height` and `'string-max-height` (`'abs`,
10, 14, 4 and 6 m) from the target meanwhile and remove them after.
`(send-event *camera* 'teleport-to-vector-start-string pos)` puts it at POS looking at the target.

Verified: jak3, `engine/target/target-death.gc`, `engine/target/target-handler.gc`,
`engine/camera/cam-master.gc`, `kernel/gkernel-h.gc` (`state`); the battlefront mod's deaths (dead
4 s with the camera on a guard of the player's side, then back at the spawn in `target-stance` with
full health; 20 of 21 and 18 of 23 units the same processes before and after), 2026-11-01. A save
with the "invulnerable" secret refuses plain attacks (`target-attacked`): a test that must kill
the player sends `'attack-invinc`.

Only for functions of GAME files. A file whose header lists level DGOs (`;; dgos: HGA, LPATK,
RAILA, ...`, as `engine/target/target-pilot.gc`) is level code: its object is linked again each
time one of those levels loads, which defines the stock function again over the wrapper. Call the
mod's function from the stock file instead: a `define-extern` above `;; DECOMP BEGINS` and one call
in the function, as the races do with `mp-race-hold?` (`artifact-race.gc`, `course-race.gc`).

Verified: jak3, `engine/target/target-pilot.gc` header and `goal_src/jak3/dgos/*.gd`; the
multiplayer mod's `mp-pidax-sidekick!` (the host rode the port attack's torpedo, which had crashed
it), 2026-10-27.

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

**Such a hurtbox crashes the dark blast.** Dark Jak's blast (L1 + square: `target-attack-uppercut`,
then `target-darkjak-bomb1`, `engine/target/target-darkjak.gc`) takes every focusable of kind
crate, enemy, guard or vehicle in the actor hash within 80 m and sends each a lightning bolt at
joint 3 (`target-bomb1-fire-shot`). The bolt follows that joint every frame (`lightning-tracker`
`update`, `engine/common-obs/generic-obs.gc`: `vector<-cspace!` on its `node-list`): a focusable
without a skeleton, or with 3 joints or fewer, stops the game (access violation in
`vector<-cspace!` from the tracker). Joint 256 makes the bolt follow `get-trans` instead; wrap
`lightning-tracker-init` (its 5th argument is the joint, typed `vector`):

```lisp
(define-perm *my-stock-lightning-tracker-init*
             (function lightning-spec time-frame symbol process-drawable vector vector none :behavior lightning-tracker)
  lightning-tracker-init)

(defbehavior lightning-tracker-init lightning-tracker ((arg0 lightning-spec) (arg1 time-frame) (arg2 symbol)
                                                       (arg3 process-drawable) (arg4 vector) (arg5 vector))
  (let ((joint (the-as int arg4)))
    (if (and arg3 (>= joint 0) (< joint 256) (type? arg3 process-focusable)
             (or (zero? (-> arg3 node-list)) (>= joint (-> arg3 node-list length))))
        (set! arg4 (the-as vector 256))))
  (*my-stock-lightning-tracker-init* arg0 arg1 arg2 arg3 arg4 arg5)
  (none))
```

Setting the blast off needs the darkjak stage `bomb1` (from the `darkjak-bomb1` feature when Dark
Jak starts), that feature in `*setting-control*` `user-current features`, the dark animations
(`ext-anim` dark), no `latch-out-time` and `eco-pill-dark` at least `darkjak-bomb-min` (50); the
input is L1 held (`target-duck-stance`), then square with L1 still held.

Verified: jak3, `engine/target/target-darkjak.gc`, `engine/target/target.gc`,
`engine/common-obs/generic-obs.gc`; the multiplayer mod's `dark-blast` scenario stopped the blasting
player's game before the wrapper, passes after it (both players set one off, no crash), 2026-10-20.

**A shock: the blue gun's knock.** An attack whose `penetrate-using` has `jak-blue-shot` gives the
unit `knocked-type blue-shot` (`enemy.gc` `penetrate->knocked-type`): it is thrown and twitches
(a Jak 3 guard plays its `blue-hit` clips, 10 to 12 of `crimson-guard-ag`). An attack that would
kill keeps the unit at 1 point for a second (`apply-incoming-hitpoint-mods`, its `fated-time`):
strike it again after a second to kill it. The knock draws no arc: a `lightning-control` (spec 20 of
`*lightning-spec-id-table*`, the guards' tazer's) between two points drawn again each frame around
the unit's body makes one.

Verified: jak3, `engine/ai/enemy.gc`, `engine/game/penetrate-h.gc`; the battlefront mod's tazer
and spider headbutt (a guard knocked by blue-shot, dead 2 s after the second strike), 2026-11-11.

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
dissolve of characters. Art without effects has no effect-control: `(new 'process
'effect-control proc)` returns `#f` when the art's `jgeo extra` has no `effect-name`
(`engine/game/effect-control-h.gc`), so `skel effect` is `#f`, not 0. Test it as the stock does
(`(if (-> self skel effect) ...)`, `process-taskable.gc`): `nonzero?` lets `#f` through, and
`do-effect` looked up on a symbol jumps to address 0 (crashes on the guest when the copy of a
barge, a KG hover or a Haven City citizen died). Vehicles keep their explosion in their own
effects: skip the call for them anyway.

Verified: jak3, `engine/game/effect-control-h.gc`, `engine/process-drawable/process-taskable.gc`;
the multiplayer mod's crash in `mp-replica-update` located with goalc's disassembly (the call
right after reading `skel effect`), gone in Haven City's KG missions once tested as a symbol,
2026-10-27. Replaces an earlier entry blaming a missing `death-default` in vehicle art.

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
renderer, not Merc2 (no per-model flags, [engine.md 7b](engine.md#7b-merc2-per-model-data-from-goal-and-a-pass-after-the-3d-scene)).

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
  through `sound-stop`. Jak's cry as a hit lands is `oof` (`hit-up` from below: target-death.gc
  `target-hit`, target-handler.gc): a wrapper of `sound-play-by-name` (`define-perm` the stock one)
  can play another sound in its place when `self` is `*target*`. Compare names with `sound-name=`
  (`engine/sound/gsound.gc`: a sound-name is 128 bits), as `(sound-name= name (static-sound-name "oof"))`.
  The guards cry `guard-hit` and `guard-die`, the spider robot `spydroid-hit` and `spydroid-die`
  (their `nav-enemy-info`); the robotguard has no cry, only `robo-explode`. Playing again with an id that is still playing updates that sound
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

**A particle group drawn in an object's local space** (`sp-group-flag sp12`) cannot be started
anywhere else: its launch control's `spawn` (`sparticle-launcher.gc`) copies the matrix of
`local-space-binding`, a `particle-local-space-info` that overlays the control's `fade` and that
only the owning object sets (`set-local-space-info`, method 15). A part-tracker started elsewhere
for such a group reads address 0 + #x20 and the game stops (access violation in the stock `spawn`
+#x143, under a `part-tracker-subsampler`). Test the flag before replaying a group:
`(logtest? (-> group flags) (sp-group-flag sp12))` (Spargus's pre-game crystal,
`group-wascity-pre-game-crystal-glow`, is one).

Verified: jak3, `engine/gfx/sprite/particles/sparticle-launcher.gc`, `sparticle-launcher-h.gc`;
the multiplayer mod's `pre-game` scenario stopped the guest twice out of two at that address (as in
a co-op session the user played), three runs without a crash once those groups are skipped,
2026-10-20.

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
  `airlock` setting and `airlock-command-lookup`. Every door type of the game inherits it, and
  none of the level subtypes overrides it: a wrapper of the method reaches every door. In its
  `open` state the door closes as soon as `want-cross-airlock?` says no (`:trans`); returning #t
  there keeps an open door open, and `(= (-> door next-state name) 'open)` tells it apart from a
  closed one (which the same answer would open). Closing runs the door's `on-exit`, `on-inside`
  and `on-deactivate` scripts only when the player stands on its front side
  (`com-airlock-method-26 'front`: `check-crossing-distance` above 0): they hide or unload the
  level behind it, and some close a story step (the temple's `tpl-door-a`), after which the door
  never opens again. `(check-crossing-distance door pos #f)` only measures the side (front
  positive); with `#t` it also runs the `on-cross` scripts and overwrites `last-distance`, which
  `'front` reads: never call it with `#t` for a position other than the player's.
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
  :name "scene-player")` plays one; `scene-lookup` finds it by name. Its `wait` state grabs the
  player, then loops while `talking`, `spooling`, a hint or another movie runs: one more term in
  that `while` holds a cutscene before it plays, nothing of it run yet (a mission's introduction
  closes its node with the scene's last commands). A mission's introduction is the scene a node's
  `when-open` event names (`(-> node when-open i scene)`, `game-task-event`). To drop a held one,
  release the player (`process-release?`) and `deactivate` the scene player: `'abort` runs its
  remaining commands, which close the introduction. Some givers (Kleiver at the arena) start their
  scene by themselves when the player comes near: a player brought next to one plays it too.
  Verified in game: the multiplayer mod's `mission-trigger` (the kanga-rat hunt's introduction
  held on the host until the guest was brought there, then played on both machines) and
  `door-wait` (a Spargus door kept open, its on-exit scripts not run, while a player stood behind
  it), 2026-10-28.

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
  frame (`update!` of `load-state` in `level.gc`). Without an `update-callback`, `target` is
  rebuilt from `want-exp` only when the wanted levels change (`add-borrow-levels`). In Haven City
  `update!` also calls the callback, the city borrow manager (`cty-borrow-manager-borrow-update`),
  every frame: it rebuilds `target` from `want-exp`, keeping a borrowed level it holds until its
  traffic objects are gone. A level whose `master-level` is not loaded stops the game when asked
  for (`level-get-for-use`: "is loading before master-level").
- **Ten level slots.** `*level*` has 10 (`LEVEL_MAX`, `engine/level/level-h.gc`), and borrowed
  levels take them too. Haven City's stock alias `ctywide-ff` borrows ctypesa, ctycara, ctycarb
  and ctypepa (`engine/game/task/game-task.gc`), and citycast often loads: ctywide and two
  sections already take 7. An eleventh wanted level gets a slot anyway: `level-get-for-use` takes
  the most disposable loaded level (`level-get-most-disposable`) with no message, and when that is
  a borrowed level whose units live, the game crashes natively (no GOAL message). The battlefront
  mod holds ctywide, ctypesa, ctypesc and six sections, nine slots: a hook at the start of
  `add-borrow-levels` rewrites `want` with plain level names (an alias would add its `auto`
  borrows), and the sections come only once every other level has left.
- **A bank of your own.** A hook in `load-begin` after the stock code clears its mask (`sv-56`)
  and before the bank search can set `heap-size`, `level-page-offset` and the mask, then
  `(goto cfg-322)`, the common heap setup. The 146 pages are 18 chunks of 8 pages, chunks 8 and 9
  of 9: chunk k starts at page 8k up to k = 8, at 8k + 2 after. A Haven City section fits in two
  chunks (16 pages, 24 MB with the PC's pages): the load notice `NOTICE: loaded <level>-vis, ...
  at top N` gives N, the bytes used from the heap's base, about 6 MB for ctyport, ctyinda and
  ctyindb. Six sections fit in chunks 0-1, 2-3, 4-5, 12-13, 14-15 and 16-17 beside ctywide's 6
  to 11.
- **The background's tree lists.** Each displayed level's `draw` of its trees adds them to
  `*background-work*` (`engine/gfx/background/background-h.gc`): 11 entries of each kind (tfrag,
  trans, water, shrub, tie), with no bound check, and `init-background` clears only 8. ctywide and
  six sections drew 17 tie trees: with 11 entries the game crashed natively once the fifth section
  was displayed, twice; with 32 it ran (7 tfrag, 6 shrub, 17 tie trees). Changing
  `background-h.gc` rebuilds the whole GAME.
- A respawn at a continue point (`target-continue`, `engine/target/target-death.gc`) writes the
  point's `want` list into `*load-state*`, calls `add-borrow-levels`, then waits until every level
  of that list shown (`display?`) is `active`. A wrapper of `add-borrow-levels` that drops one of
  them leaves the respawn waiting for ever: the battlefront mod's level filter hung its Krimzon
  Guard camp in the Port, `ctyport-warzone`, which lists `ctyinda`.
- `*load-state*` `vis-nick` is the level the game sees the player in (`level-get-target-inside`).
  While `*level*` `border?` is on, the load-state update switches it (`want-vis-level`) to an
  active level with continue points whose boxes hold him (`inside-boxes?`), once the current one's
  no longer do; a continue point sets it from its own `vis-nick`. It is not a map of the districts:
  moved to `ctyinda-bbush-5` (Industrial Zone) with `ctyinda` loaded, Jak stood there 8 s and it
  stayed `ctyport`. The battlefront mod also tests his minimap cell (4.28, `city-map-bits`).

Verified: jak3, `engine/level/level.gc`, `engine/load/load-state.gc`; seen in game in the
multiplayer mod (the bank error and crash when loading wascityb next to wascitya; crashes at a
level unload until copies and part-trackers of the level, active or only loaded, were dropped
first), 2026-10-14; `engine/target/target-death.gc` (`target-continue`); the battlefront mod's runs
(a respawn waiting for ever on a level its filter dropped; vis-nick kept on `ctyport` with Jak 8 s
in the Industrial Zone), 2026-11-01. Slots, banks and tree lists: jak3, `engine/level/level.gc`
(`level-get-for-use`, `load-begin`, `update!`), `engine/level/level-h.gc`,
`levels/city/common/cty-borrow-manager.gc`, `engine/gfx/background/background-h.gc`,
`background.gc`; the battlefront mod's whole-city runs (two native crashes as the sixth section
loaded, then six sections, nine slots and the counts above), 2026-11-09.

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
- **The last save, again.** `(auto-save-command 'restore 0 (-> *game-info* auto-save-which) proc
  #f)` loads the slot the game last loaded or saved, as the START menu's Load Game does
  (`progress.gc`); the auto-save process's `done` keeps that slot in `*game-info*` `auto-save-which`
  and sets `(-> *setting-control* user-default auto-save)`, so that flag says a slot was loaded or
  saved since the boot. The load's `initialize!` 'game resets the game at the save (the battlefront
  mod's battle ends there: back at the save's place within a second, 2026-11-01).
- **No save.** Every card write goes through `auto-save-command` (`'save`, `'save-last`,
  `'auto-save`, `'create-file`, `'format-card`, `'unformat-card`; `'restore` reads): the pause
  menu's saves, checkpoints, missions' `(auto-save 'auto-save)`, a new game's first save. A
  `define-perm` wrapper that returns `#f` for those turns saving off; tell the menu that asked
  with `(send-event menu 'notify 'error (mc-status-code no-process))`, or its "Saving" page waits.
- **The save icon, from any process.** `auto-save-post` draws a `hud-sprite` (`new 'global`
  works): texture `(get-texture checkpoint level-default-minimap)`, `pos` 440 210 (z `#xffffff`),
  scale 2.0, its four `color-ptr` at 128, no flags, shown while
  `(< (mod (-> *display* real-clock frame-counter) 300) 270)`. Draw it as that post does:
  `(draw sprite buf (the-as level #f) #f)` into the frame's `global-buf`, then, if `base` moved,
  a `next` `dma-packet` and `dma-bucket-insert-tag` into `hud-draw-hud-alpha` from the old base.
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
`engine/target/logic-target.gc`; the save icon drawn after the multiplayer mod's co-op saves
(`coop-save` scenario, screenshot), 2026-10-29; in the multiplayer mod's `coop-campaign` and `lobby` scenarios
(a mission started with 4 nodes closed, the story closed with 254, the sidekick off and back),
2026-10-19; in game with the multiplayer mod (`join-world`: a guest in
Haven loaded the host's 42928-byte save sent over the network, came out at the host's Spargus
checkpoint with the host's progress, its language kept, an auto-save refused; `join-title`: the
same from the title screen), 2026-10-15.

**Telling a load from a respawn.** Every reset goes through `game-info` `initialize!`
(`engine/game/game-info.gc`). A save loaded from the START menu arrives as `'game` with a non-#f
`game-save` (`game-save.gc` `restore` state, the only such call); a new game or the title as
`'game` with #f; deaths and `initialize!` calls of a mod as `'life` (`'dead` becomes `'life`); a
mission retry as `'try`. A `define-perm` wrapper of the method sees the call before the reset and
before the save's data (`load-game` runs a frame later in the `'play` branch's thread): a mod that
stops its mode there, synchronously (deactivate its process, so its settings go too), lets the save
set everything afterwards. Processes in `*pc-pool*` outlive the load; only `*entity-pool*` and
`*pusher-pool*` are reset.

Verified: jak3, `engine/game/game-info.gc`, `engine/game/game-save.gc`, `engine/entity/entity.gc`;
the battlefront mod (a load of slot 1 during a battle: mode off, its director and avatar gone, Jak
at the save's place, the stock city units back), 2026-11-01.

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
- **Every awake pedestrian of the traffic** (citizens, guards, flying turrets, roboguards,
  spydroids: tracker 1), every frame: outside the city cells marked `active-pedestrian` it is sent
  `'traffic-off` and goes back to its pool, unless it is `persistent` (`traffic-engine.gc`, the
  `sphere-in-loaded-city-infos?` test with bit 1). Those cells are the ones the camera's cell can
  see: a crowd fighting 100 m away behind buildings vanished. One `or` with a mod's own test in that
  `if` keeps the units of an area awake (the battlefront mod: units standing on its battlefield,
  during a fight); the stock `persistent` flag is no help, the guards' and turrets' own states set
  and clear it.

A citizen that goes inactive is still a process: test `focus-status inactive` or `no-draw`, not
whether its handle resolves, to tell it vanished.

Beyond that, every process with the `actor-pause` mask stops running (trans, code, post) farther
than `(-> *ACTOR-bank* pause-dist)` (50 m) plus its root's `pause-adjust-distance` from the camera:
`run-logic?` of `process-drawable` (`engine/entity/entity.gc`), and of `enemy`, which also pauses
an idle enemy off screen (`engine/ai/enemy.gc`). The kernel asks it each frame, so wrapping both
methods (4.10) keeps the actors near another viewpoint running.

Verified: jak3, the files above; seen in the multiplayer mod's tests (flitters going inactive in
the port's water, copies hidden when the host walked away), 2026-10-15; the traffic's cell test in
`levels/city/traffic/traffic-engine.gc` and `levels/wascity/traffic-util.gc` (`sphere-in-grid?`),
the battlefront mod's armies with the battlefield kept awake (up to 17 units a side awake with
Jak at the battlefield's edge, 12 before), 2026-11-01.

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

**The engine's own way: `top-anim`.** Any `process-drawable` can carry a
`top-anim-joint-control` in `(-> self skel top-anim)` (Jak's gun clips on his arms,
`target-gun.gc`); `evaluate-joint-control` updates it every frame before the joints are computed,
so the locomotion code keeps its own channels (`ja-channel-push!` and `ja-channel-set!` move the
float channels it adds). Make it in the process's init, `(new 'process 'top-anim-joint-control
self)`, with `interp-select 0` the joints' mask and `base-anim` a clip of the model. To play a clip:
`interp` 1.0 first (`push-anim-to-targ` does nothing while it is 0), then `(push-anim-to-targ top
clip start-frame blend-in post-blend speed post-end #f)` (frames counted as `artist-step` times the
frame index); set `frame-post-put-away` (a `basic`: `(the-as basic #t)`) right after the push, as
the push clears it: at the clip's end `interp` goes to 0 and the upper body blends back into the
legs' clip. Without it the clip holds its end, then plays `base-anim`; never leave `base-anim` #f
there, `update` would make a channel of `#f`. `(-> top frame-targ)` turns #f when a pushed clip
ends: the moment to push the next one. The joint-control's `relocate` moves the top-anim with a
heap that compacts. Crimson guard skeleton (`crimson-guard-lod0`): above the hips are swivelMe 3 to
left_wrist 17, the fingers 25 to 27 and the head, ears and thumbs 29 to 40, `#x1ffee03fff8`; the
hips 18 hold the legs (18 to 24, 28).

Verified: jak3, `engine/process-drawable/process-drawable.gc` (`top-anim-joint-control` `update`),
`engine/target/target-gun.gc`, `engine/entity/relocate.gc`; the battlefront mod's guard class
(rifle, grenade and reload clips on the upper body while the legs walk, logged clip by clip and
seen in screenshots), 2026-11-01.

## 4.20 The order processes run in

`change-parent` (`kernel/gkernel.gc`) puts a process first among its parent's children, and a
tree runs a parent, then its children from first to last: the latest child runs first. The pools
are made in that order under `*active-pool*`: `*pc-pool*`, `*camera-pool*`, `*target-pool*`,
`*entity-pool*`, `*pusher-pool*`, `*bg-pool*`, `*default-pool*`, so they run the other way round,
`*pc-pool*` last. A process in `*pc-pool*` runs after the player and its children (gun,
sidekick) every frame, before anything is drawn: it can undo what their posts set.

A process in `*entity-pool*` runs before the player: a model that copies `(-> *target* root trans)`
there draws him where he was the frame before, while the camera follows him where he is. The gap
is his speed times a frame, so it changes with every change of speed and the model shakes (the
battlefront mod's jetpack). Place it again from a `*cam-post-hook-engine*` connection, then
`do-joint-math` (4.31): its drawn root then matched the player's height of that frame exactly
(gap 0.0000 m with the player climbing 0.10 to 0.19 m a frame).

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
- **Any process can race.** A `racer-state` follows its `racer` handle's process: its
  `root trans` counts the laps (`update-lap-distance`), and a process gone, `dead` or `inactive`
  puts it out of the race (`rsf2`). `(init-racer! (-> rs racer-array i) proc)` binds any
  `process-focusable` (another player's puppet) to slot `i` after the race spawned its racers
  (`task-manager-race-method-33`, whose course and rally overrides cast slots 0-3 to `vehicle`),
  once the AI there is removed with `'go-die` (as `'kill-npc-racers` does; non-vehicles ignore it).
  `'force-start` sets `rf4`, which skips the racers' `'test-ready` (a process that does not answer
  it would hold `rs0` forever). `i-player` is the slot the HUD, the place and the result read; the
  race files are level code (WASLEAPR, DESRALLY, DESTRACK): code reading `race-state` goes above
  `;; DECOMP BEGINS` of `race-manager.gc` or `course-race.gc`. The racers' AI
  (`levels/desert/wvehicle/wvehicle-race.gc`, `v-marauder` method 177) reads every other racer's
  `rbody matrix fvec` to steer clear of it: a racer that is not a `rigid-body-object` crashes
  every AI racer (an access violation at address -1) unless that read is guarded.
  Verified in game: the multiplayer mod's `race-finish` and `race-finish-win` scenarios (another
  player's puppet in slot 3 of Kleiver's race, its laps, place and finish counted on the host),
  2026-10-28.
- **Running a race to the finish without a player** (a test): once the race runs (`*game-info*`
  `race-timer` above 0), `(send-event <player's vehicle> 'set-control-hook-race-ai)` hands the
  player's turtle to the racers' AI (`wvehicle-method-177`, `levels/desert/wvehicle/wvehicle.gc`),
  which drives Kleiver's three laps (about 75 s a lap) and crosses the line: the race finishes as
  for a player. The race drives its racers every frame: `'set-control-hook-ai` with
  `'ai-set-target-speed 0` does not stop them; deactivating a racer leaves its place frozen.

Verified: jak3, `engine/game/task/task-arrow.gc`, `engine/target/target-death.gc`,
`levels/desert/race/course-race.gc`, `levels/desert/artifact-race/artifact-race.gc`; the
multiplayer mod's co-op waypoints and race starts (`beam`, `race-board`, `race-continue`
scenarios), 2026-10-19; Kleiver's race and the arena training played in co-op, the arena's manager
sent to `done` (`race-real`, `race-real-daxter`, `beam-real`), the intro skipped in a retail boot
(`retail-general`: the title menu after 6 s instead of 274 s), 2026-10-19;
`levels/common/race/race-manager.gc`, `race-hud.gc`, `engine/game/task/task-control.gc`,
`engine/util/script.gc`, `levels/desert/wvehicle/wvehicle.gc`; the mod's `mission-start`,
`race-result`, `race-board`, `race-finish`, `race-finish-win` scenarios (Kleiver's race driven to
the finish by the racers' AI on two machines), 2026-10-20.

**Closing every mission and activity.** `(set-setting! 'exclusive-task #f 0.0 (game-task none))`
makes every task node count as not open: `game-task-node-info-method-12` (`task-node-open?`,
`engine/game/task/task-control.gc`) lets a node through only if its task is one the setting names,
and no node has the task none. NPC prompts (`process-taskable`), burning bushes, task managers and
their minimap icons all go through that gate. A node already in progress (flag `exclusive`) stays
open. The setting goes with the process that set it. `task-mask` is not a mission switch: it
decides which entities are born.

Verified: jak3, `engine/game/task/task-control.gc`, `engine/game/settings.gc`; the battlefront mod's
battles (open task nodes: 15 before, 0 during, 15 after), 2026-11-01.

**No vehicle either.** `(set-setting! 'pilot #f 0.0 0)` keeps the player out of every vehicle:
vehicle `check-player-get-on` (`levels/city/traffic/vehicle/vehicle-util.gc`) wants
`user-current pilot`, as do the flut, mech and turret mounts. `vehicle-hijacking` only decides
whether a vehicle without the `waiting-for-player` flag can be taken; one waiting for the player
is taken without it.

Verified: jak3, `levels/city/traffic/vehicle/vehicle-util.gc`, `engine/game/settings.gc`; the
battlefront mod's battles (`user-current pilot` #f during a fight), 2026-11-01.

## 4.22 Leaper lizards (`flut`): where their code lives, spawning one

- **Type in GAME, code in the levels.** `deftype flut` is in `flut-h.gc` (GAME): a GAME file can
  test `(type? p flut)` and read its fields. Its states and `flut-init` are in `flut.gc`, loaded
  only with WWD, DESLIZ and VOCA. Find the init by name, and only call it while a lizard exists:

  ```lisp
  (let ((init (-> 'flut-init value)))
    (when (and init (type? init function))
      (process-spawn flut
                     :init (the-as (function entity-actor transformq handle flut-flag symbol object) init)
                     ent tq rider (the-as flut-flag (+ 1 (shl color 32))) 'normal
                     :name "flut" :to *entity-pool*)))
  ```

- **A spawned lizard.** With an entity and a transform, it stands at the transform (the entity
  gives its level and task mask). A live `rider` handle makes it go idle at once; `#f` sends it to
  `wait-for-start`, which waits while its spot is in view within 100 m, while another lizard
  stands within 4 m (`flut-method-33`), and while 10 lizards are out (`*game-info* flut-count`,
  counted by `idle` once per lizard through `count-lock`). The `rider` field is not read after
  the init.
- **Picked up.** A lizard that is not its entity's own process (`(-> entity extra process)`)
  deactivates once ridden; the entity's own one waits hidden (`wait-for-return`) until its next
  birth. Getting off spawns a new lizard at that spot (`target-flut.gc`).

Verified: jak3, `engine/target/flut/flut.gc`, `flut-h.gc`, `target-flut.gc`, `dgos/game.gd`; the
multiplayer mod's `mounts-coop` scenario (a second lizard of the same entity, idle at once, kept
out of the count, hidden and shown for another player), 2026-10-20.

## 4.23 A borrowed level on demand, and an engine shadow crash

- **Loading a borrowed level from your code.** Missions list their borrowed levels in their task
  nodes (`game-task.gc`, `:borrow '((waswide 0 lkleever special))`: level `lkleever` into
  `waswide`'s borrow slot 0). Ask the same from a process that lives as long as you need it:

  ```lisp
  (set-setting! 'borrow '((waswide 0 lkleever special)) 0.0 0) ;; owned by the calling process
  (apply-settings *setting-control*)                            ;; a borrow change runs update-task-masks
  ;; later: (remove-setting! 'borrow) (apply-settings *setting-control*)
  ```

  `apply-settings` compares the borrow lists and calls `(update-task-masks 'event)` when they
  change (`engine/game/settings.gc`), which applies them and loads the level
  (`add-borrow-levels`). Its art is then found by name (`art-group-get-by-name *level*
  "skel-kleever-highres" #f`). Spawn a model of it to `*entity-pool*`: when the borrowed level
  unloads, `check-for-rougue-process` (`engine/entity/entity.gc`) kills every process-drawable of
  `*pusher-pool*`, `*entity-pool*` and `*default-pool*` whose art is in that level's heap.
  Kleiver's model (`lkleever`, 4.5 KB of level plus his art) loads at Spargus this way.
- **Engine: the shadow overflow skip.** `Shadow2::render`
  (`game/graphics/opengl_renderer/foreground/Shadow2.cpp`) skips a shadow too big for its VU
  program ("shadow overflow detected, skipping all shadows for this frame!", upstream #3934, for
  NaN bones). Its loop tests `while (dma.current_tag_offset() == render_state->next_bucket)`: it
  skips nothing, and `dispatch_buckets_jak3` then stops on `dma.current_tag_offset() ==
  m_render_state.next_bucket` (OpenGLRenderer.cpp). The fix is `!=`. Seen on a guest's game in the
  palace ruins (`rubblea2`), right after the overflow message.
- **Engine: the shadow vertex buffers.** `Shadow2` gathers every shadow volume of a frame in fixed
  buffers (`kMaxVerts` = 8192 * 3 * 2 vertices, `Shadow2.h`). A crowd of shadow casters (a few
  dozen guards and robots on screen) fills them and `alloc_verts` asserts
  (`m_vertex_buffer_used <= m_vertex_buffer.size()`), which closes the game. Before each upload,
  skip it when less than its worst case is left (255 triangles of 4 vertices and 5 indices, top and
  bottom), and make the buffers larger: the battlefront mod does both (2026-10-31), and the
  assertion did not come back.

Verified: jak3, `engine/game/settings.gc`, `engine/game/task/task-control.gc`,
`engine/entity/entity.gc`; the multiplayer mod's `merchant` scenario (Kleiver's model at his
spot, `lkleever` active) and the user's crash log, 2026-10-20.

## 4.24 Mission failures: the resetter

- **One entry point.** Every failure, retry and restart screen is a `resetter` process started by
  `(spawn-resetter! *resetter-control* params node-info)` (`engine/game/task/task-control.gc`):
  the task manager's `fail` state with its `fail-info`, the game's fallback (`mission-fail`, back
  to the mission's `play-continue`), a restart. Wrap that method to see or copy them all.
- **Params** (`resetter-params`, 48 bytes): `message` (`mission-fail`: no choice, X goes on;
  `mission-fail-or-retry`: X retry, triangle quit; `mission-retry`: a restart), `flags`
  (`auto-reset` with `reset-delay`: it goes on by itself), `fail` and `retry` specs (continue name
  or continue point, node, `reset-mode` `'life` or `'try`, an `execute` script), `task`,
  `text-message`. Story missions have no `fail-info` and use the fallback; the minigames offer
  retry or quit.
- **Its choice** is read in its `idle` state with `cpad-pressed? 0 confirm` / `triangle` once it
  grabbed the player and slowed the world down (1.5 s on the real clock; then
  `set-master-mode 'freeze`). Hide those buttons from it (a process that runs before it) and give
  them back on one frame to drive it. Time it with `(-> *display* real-clock frame-counter)`
  against its `grabbed-time`: a process's `current-time` stays frozen meanwhile.
- **`mission-fail-or-retry` never goes on by itself:** its `idle` loop reads only X and triangle;
  `auto-reset` is honored by `mission-fail` and `mission-retry` alone. The burning bush challenges
  fail with it (their `fail-info`): the game stays frozen until the player chooses (a test left on
  it waited three minutes).

Verified: jak3, `engine/game/task/task-control.gc`; the multiplayer mod's `mission-fail` and
`mission-fail-retry` scenarios (a failure shown on two machines, decided together), 2026-10-21;
`challenge-one` (a challenge's failure, 2026-10-27).

## 4.25 Burning bush challenges (`bbush`)

- **Two task nodes,** both with `game-task-node-flag bbush` (`engine/game/task/game-task.gc`): the
  introduction, which the bush's `talking` state closes (`task-node-close! current-node 'event`,
  `levels/desert/des-burning-bush.gc`), and the resolution, whose `task-manager-info` runs the
  challenge: `type-to-spawn`, `level` (the manager spawns once that level is active), `user-count`
  (its reward in precursor orbs) and `reset restart-info retry continue` (the challenge's start).
  The task's play-list entry names the introduction (`pre-play-node`). The skull gems are paid
  once per bush, in its entity perm (`user-object 0` set to 1).
- **When a bush offers it:** `task-node-open?` (`game-task-node-info-method-12`) wants the parents
  closed and no exclusive task running (`task-node-exclusive`, `exclusive-task-count`): during an
  exclusive story mission no challenge's introduction is open. `task-node-close!` closes it anyway
  (and the parents it needs).
- **The reward** is `(give *game-info* 'skill n handle)`: from the manager (`process->handle
  this`: rings, chases, the jet-board, vehicle scores, the races' `race-manager`, a child of the
  task manager), or from the precursor orb a get-to challenge leaves at its end, spawned as the
  manager's child (`birth-pickup-at-point ... self`). `adjust-money-per-level` (`game-info.gc`)
  reads the parent of the handle's process with no check: never pass `#f` or a dead handle to
  `give 'skill`.
- **The jet-board's judge** (`judge`, `engine/common-obs/voicebox.gc`) is spawned where the
  player stands when the challenge starts, asleep in `wait`, and starts its 90 s only on a
  `'touch` from the player (the stock collision). Daxter (`indax` mode) never touches it, and Jak
  switched in inside its sphere gave no touch either: send it the touch yourself, an
  `event-message-block` whose `from` is `(process->ppointer *target*)`. It copies
  `(-> *target* fact trick-point)` into `*game-info*` `score` every frame; the manager gives
  `(cups gained) x user-count` orbs, or fails without a new cup.
- **Scores** live in level types: `task-manager-bbush-board` `score` at offset 240,
  `task-manager-vehicle-bbush` `score` at 304 (its report state sets it from the air time, the
  distance or the roll count), `node-info`, task-manager's first field, at 128
  (`decompiler/config/jak3/all-types.gc`). Code linked before those levels reads them at
  `(&+ (the-as pointer (&-> mgr node-info)) (- offset 128))`.

- **Following a manager to its end:** keep its state and score every frame. Right after the
  jet-board's manager gave its reward, its handle stopped answering on the next frame, while its
  resolution still played its line (`bb17win`).

Verified: jak3, `levels/desert/des-burning-bush.gc`, `des-bbush-tasks.gc`, `des-bush.gc`,
`levels/city/ctywide-obs.gc`, `engine/game/game-info.gc`, `engine/common-obs/voicebox.gc`; the
multiplayer mod's `challenge`, `challenge-one` and `challenge-board` scenarios (the city's get-to challenge 26 started and reached on two machines, its
3 orbs held and given), 2026-10-27.

- **Playing a challenge again.** Once its resolution closed, `(task-node-open! <introduction> 'event)`
  (as the bush's own menu does, `des-burning-bush.gc`) reopens the introduction and, through
  `game-task-node-info-method-11`, its children: the bush offers it again (it reads its current
  event every frame). A bush offering several challenges (`burning-bush-desc-4`) offers the first
  open one: reopen them once all are done. A challenge paid by cups pays only the improvement over
  `get-best-rank` (`*game-info*` `game-score`, 8 floats per score): zero those scores for a play to
  pay its cups again.
  Verified in game: the multiplayer mod's free roam (`challenge-free`: Haven's get-to challenge 26
  offered again on both machines 5 s after it was decided), 2026-10-28.

## 4.26 The jet-board, and Daxter on it

- **Taking it out:** `want-to-board?` (R2) in Jak's stance, or `'change-mode 'board` to the target,
  goes to `target-board-start` (`engine/target/board/board-states.gc`), which leaves the current
  mode (`target-exit`) and spawns the `board` process. Daxter's mode (`indax`) ignores both: a
  replaced handler of his stance and walk can send him to `target-board-start` too; the game feature
  `board` must be in `(-> *game-info* features)`.
- **The clips** are external: `jak-board+0-ag` (target-anim `board`, `*target-anim-control*`)
  holds Jak's `jakb-board-*` (jakb-ag 164-228) and 56 of Daxter's 67 `daxter-board-*` (daxter-ag
  147-213, the same order, 17 below); they are loaded once the board is out and freed after it.
- **The sidekick on the board** plays Daxter's board clips matched to Jak's, its root on Jak's joint
  37 (`sidekick.gc`, `'matrix 'board`, sent by the get-on): Daxter clinging to Jak's back. A
  `cspace` root function set in `(-> sidekick node-list data 0 param0)` places him elsewhere (on the
  board: `board board-trans`, `board-quat`); the get-on sets it back, so set it every frame.
- **The board** copies `board-scale` (the target's `control scale`, every frame) into its root
  and hides whenever the target has `no-draw-bounds2` (`board-post`, `board-util.gc`): wrap its
  `use` state's post (`(method-of-type board use)`) to scale or show it.

Verified: jak3, `engine/target/board/*.gc`, `engine/target/sidekick.gc`,
`engine/target/indax/target-indax.gc`; the multiplayer mod's `daxter-board` scenario (Daxter on a
board at 60 %, stance, jump, put away; screenshots), 2026-10-27.

## 4.27 Places at the same coordinates

Several places share Haven's ground plan, reached through a loading transition (entity positions,
`decompiler_out/jak3/entities/<level>-actors.json`, in meters): the volcano (`volcanoa`, x -250..129,
z -321..255), the mine (`minea`..`minee`) and the halfpipe lie on Haven's ruins (`rubble*`, x
-500..420, z -384..882); the war factory (`factoryb`..`d`, y 274..566) and the precursor ship
(`precura`..`d`, `lprecurc`, y 200..574) stand above them and overlap each other; the catacombs
(`comb*`) and their rail (`rail*`, down to y -3780) run below; the nest (`nsta`, `nstb`) lies under
Haven's industrial quarter. Within Haven, the ruins (`rubble*`, `stadiumb`, `lpattack`,
`lppatrol`: loaded with `stadiuma`, never with `ctywide`, reached through `stadium`) and the Metal
Head city (`mhcity*` with `tower*`: loaded with `ctywide-mh`, reached from `ctyport`) both lie over
the farms (`ctyfarm*`): a ruins gate panel at y 16 stands 4 m from a farm emitter at y 38. The
test for two places: never in the same continue point's load list, and entities filling the same
volume at heights one ground cannot hold (stacked levels such as the sewers under the city share
one consistent space). Haven (x about -880..1180) and the wasteland with Spargus (x above 1170, or
z above 2000) never overlap. Anything placed by position alone (a remote player, a copy,
an effect, a distance check) must also compare the levels: the continue `volcano-flut-1` stands
23.6 m from `ctygenb-stadium`.

Verified: jak3, the entity dumps above; the multiplayer mod's `places` and `places-before`
scenarios, 2026-10-27.

## 4.28 The minimap: icon classes, the turned marker, the big map

An icon is a connection: `(add-icon! *minimap* proc class-index #f (the-as vector #t) 0)` follows
`proc`'s root; its `class` is a pointer into `*minimap-class-list*` (177 classes,
`engine/ui/minimap.gc`) that can be pointed at a copy (`mem-copy!` a node, change `color`, `flags`,
`scale`, `icon-xy`). Removed with `kill-callback` on `(-> *minimap* engine)`.

- **Icons** (`sub-draw-1-2`): a cell of the `mini-map-icons` texture (`icon-xy`, `scale`), in the
  class color; class 117 `red-clamped-dot` is a dot kept on the minimap's edge when far (`clamp`)
  and shown on the big map (`bigmap`).
- **The turned marker** (`racer` flag, class 29 `racer`): `draw-racer-2` draws
  `map-target-marker`, the player's own marker, turned by the process's root quaternion, into the
  map's texture (`sub-draw-1-1`) within 50 texels of the player, in the class color (128 = the
  texture's color; the player's own marker is drawn at 255). A `racer` class is never drawn as an
  icon, and the icons, drawn after, cover it. In Spargus a texel is `meters-per-texel` ×
  `icon-inv-scale` = 22118.4 × 0.846 units (4.6 m): a player 30 m away is 6 texels off, under its
  own dot.
- **The big map** (`bigmap.gc`) draws every class with `bigmap` (or `bigmap-only`) as an icon, by
  its `icon-xy` and `scale`.
- **A unit's mark goes by its type, not its side.** `crimson-guard`'s init adds class 25
  `guard-frustum` (blue, its view cone drawn by `draw-frustum-1`), `kg-squad-member`'s (flying
  turret, roboguard, spydroid) class 139 `robot-frustum` (red). Each type keeps its own `minimap`
  field (`levels/wascity/guard-h.gc`, `levels/city/common/kg-squad-member-h.gc`). To color by side,
  point `(-> unit minimap class)` at `(-> *minimap-class-list* 25)` or `139` every frame from the
  unit's first awake frame: done twice a second, one newly woken unit in 80 still showed the stock
  one.
- **A mark at a fixed place.** Pass a vector instead of `#t` to `add-icon!`: the connection keeps
  its address (`position`) and draws the mark there; class 10 `guard` is a plain dot (`icon-xy` 0
  1), black with `color` 0 0 0 and larger with `scale`.
- **Hiding the HUD minimap.** `(set-setting! 'minimap 'clear 0.0 (minimap-flag minimap))` hides it,
  as the stock bombbot and blow-tower missions do; `(remove-setting! 'minimap)` from the same process
  brings it back: call it when your own full-screen map closes and when the mode ends.
Verified: jak3, `engine/ui/minimap.gc`, `levels/city/traffic/citizen/guard.gc`, `levels/city/common/kg-squad-member.gc`; the battlefront mod's round 20 runs, 2026-11-09; the minimap setting in its round 21 run (R3's map up, no minimap), 2026-11-09.

Verified: jak3, `engine/ui/minimap.gc`, `engine/ui/bigmap.gc`; the multiplayer mod's players shown
by a marker in their color (scenarios `minimap-cone`, `coop-roles`, screenshots), 2026-10-28.

**The save menu's pictures outside the menu.** A save slot's picture is
`(-> *progress-icon-arrays* task-level)` (`draw-icon-array!`, four tiles making 96 x 96; every
Haven City section is task-level 1, `city`), in the progress-minimap texture page. `*bigmap*`
loads that page only while its drawing is on and the blit count-down reaches 1 (`bigmap.gc`
`update`): `(enable-drawing *bigmap*)` then `(set! (-> *blit-displays-work* count-down) (the-as uint
1))`, wait for `(loaded? *bigmap*)`, `(disable-drawing *bigmap*)` after. It links the page as the
default level's pris2 page, whose upload goes to `tex-lcom-pris2`, a bucket the PC renderer wants
empty: the game stops on "Bucket renderer 456 was supposed to be empty". While the page is held,
upload only what the stock menu mode does, `(texture-enable-mask shrub water hud)` in both
`*texture-pool*` `texture-enable-user-menu` and `texture-enable-user` (`blit-displays.gc`), and put
the saved value back after `disable-drawing` (which waits until the page is dropped).

**A part of the world map, framed by you.** The same load brings the START menu's map image,
`(-> *bigmap* bigmap-image art-group)`: a `(pointer uint32)` header (palette offset, image offset,
width, height) then 8-bit pixels in 256-wide columns, each column the image's full height. Haven
City's is 512 x 832 pixels of 4 m: pixel x is `(world x - offset x) * offset w`, pixel y `(world z -
offset y) * offset w`, `offset` being `(-> *bigmap* offset)` (`*bigmap-info-array*`'s city entry).
Its art is the in-game minimap's. `draw!` always shows 512 pixels from `scroll`, the whole width of
that image, and `bigmap-method-20` reads 512 pixels from the scroll whatever the box: neither can
frame a district. Draw the columns yourself, in a bucket of your own (`tex-hud-hud-alpha`): set
`draw!`'s scissor (your box), test, alpha and clamp registers; per column, `pc-upload-map-texture`
256 x 416 rows from the row you need (`image offset + column * height * 256 + row * 256 + 16`, the
palette at `palette offset + 16`, tbp 8), the `tex0-1`/`tex1-1`/`texflush` set of
`bigmap-method-20`, and a copy of `sprite-dma` that takes the texture rows as arguments (its own
always start at row 0); then `(set-dirty-mask! (-> *level* level-default) 4 #x1a400 0)` as `draw!`
does. Scale the sprite's width and height to zoom.

Verified: jak3, `engine/ui/bigmap.gc`, `engine/gfx/blit-displays.gc`, `pc/progress/progress-draw-pc.gc`,
`engine/gfx/texture/texture-h.gc`; the battlefront mod's zone pages (the assertion without the
mask, the picture drawn with it; the world map framed on each war zone, its spawns on the streets),
2026-11-01.

**The minimap's own textures differ from that map.** The START menu's map lays the streets over a
textured ground; the in-game minimap's textures have a flat dark gray ground (same streets, water
and scale). Haven City's minimap is a grid of 5 x 7 cells of 384 m: column `(x + 768 m) / 384 m`,
row `(z + 768 m) / 384 m`; each cell names a texture covering 1024 m
(`*minimap-texture-name-array*`, `map-ctyportc`...), 256 x 256 texels of 4 m, or 256 x 128 texels
8 m tall on the city's south row (`map-ctyportd`, `-e`, `-f`), whose top left corner is
`*minimap-corner-array*` (-1088 m + 384 m per column or row), so neighbouring cells overlap. A level
lists the cells it covers in its `city-map-bits` (`level-info.gc`). Each
texture lives in its city section's own `<section>-minimap` page, loaded only with the section
(`lookup-minimap-texture-by-name`). To show another section's, carry its texels: 8-bit indices plus a
16 x 16 ct32 palette with entry i at position i with bits 3 and 4 swapped (the PS2 order, as in the
world map's palette), uploaded with `pc-upload-map-texture` like the map above (`tex0` `:tbw` the
width / 64, `:tw` and `:th` its log2) and drawn as one sprite. The battlefront mod's
`scripts/battlefront/make_minimaps.py` builds each war zone's whole district from the decompiled
PNGs: the bounding box of its sections' cells (`city-map-bits`), each texel taken from its cell's
texture sampled at that texture's own size, other levels' cells black, at most 256 texels a side
(34 colors for four zones).

**The center has no minimap texture.** Jak 3 ships no `map-ctygena`, `map-ctygenc`, `map-ctypala`,
`map-ctypalb`, `map-ctymarkb` or `map-stadium` in any texture page (`map-ctymarka` sits only in
`mhcityb-minimap`): a city map made of minimap tiles has black holes over the ruined center, the
stadium and the west. For the whole city, read the START menu's map from the disc instead:
`iso_data/jak3/STR/WOMAP.STR` starts with a table of 32-bit sector numbers (2048 bytes a sector),
the map of `bigmap-id` i (`level-h.gc`, `city` 0) at sector i; there, the header and data above
(the city's: palette at +16, texels at +16 + 1024, 512 x 832, columns 256 wide), texel 0 at world
x -640 m, z -1088 m, 4 m a texel (`*bigmap-info-array*` 0). The battlefront mod's
`make_minimaps.py` crops it to the city's outline and resamples it to 384 texels high.

Verified: jak3, `engine/ui/minimap.gc`; `iso_data/jak3/STR/WOMAP.STR` decoded offline (the city
map, 512 x 832, drawn in the battlefront mod's war zone page, 2026-11-08) and compared with
`decompiler_out/jak3/textures/ctyport-minimap/map-ctyportc.png` (same streets at the same place,
different ground); the battlefront mod's four zone pages drawn from that data (screenshots,
2026-11-01); `decompiler_out/jak3/textures/ctyport-minimap/` (`map-ctyportd`, `-e`, `-f` 256 x
128), `engine/level/level-info.gc` (`city-map-bits`), 2026-11-01.

## 4.29 Text the game draws: escapes, size, glyphs

`print-game-text` reads `~` codes in any string it draws, and the game's own texts are full of
them: `lookup-text!` returns French "Appuyez" as `A~+7Vp~-7V~+7Vp~-7Vu~+7Vy~-7Vez` (letters
with a descender moved down) and accents as `~Y~-26H~-5V~Z` sequences drawn over a letter. Keep
the game's texts as they are, and strip or replace `~` in text a player typed (a chat line, a
name), or it draws as codes. A custom text's `<PAD_X>`-style tags become button icons when the
text compiler builds it.

- **Size: use the large font.** A `font-context` without `large` in its `font-flags` draws the
  game's small font: at scale 0.7 a line was about 5 pixels high in a 1280x720 capture, unreadable.
  With `(font-flags shadow kerning large)` the line height is the scale times the size passed
  to `print-game-text` (44 in the multiplayer mod's calls) instead of 28 (`engine/ui/text.gc`);
  that mod's readable lines use `large` at 0.45 to 0.55.
- **No `>` glyph.** The font draws `>` as an accent mark (`´`): a prompt such as `"> "` before
  typed text shows as `´ `.

Verified: jak3, `game/assets/jak3/text/game_custom_text_fr-FR.json` compiled, the multiplayer
mod's chat answers and lines (`chat` scenario, logs and screenshots), 2026-10-29; the small font
and the `>` glyph seen in that mod's README captures (`readme-free-roam` scenario, English,
960x540), 2026-11-09.

**A page drawn by GAME code.** `draw-string-adv` with `(font-flags middle)` centers the text on
`origin x`: the screen is 512 units wide, so x 256 centers it on the screen (`width` plays no
part). Draw the panel (`draw-sprite2d-xy`) and its texts into one `with-dma-buffer-add-bucket`,
panel first: from separate buffers of the same bucket, the panel covered the text. Nothing wraps:
break long lines yourself. The game's texts follow `(-> *pc-settings* text-language)`
(`pc-language`), not `*setting-control*` `language`, which a loaded save sets to its own. The
game's font draws the ASCII apostrophe as an acute accent (`d'autrefois` showed `d´autrefois`).

```lisp
(with-dma-buffer-add-bucket ((buf (-> (current-frame) global-buf)) (bucket-id debug-no-zbuf2))
  (draw-sprite2d-xy buf 56 70 400 250 (static-rgba 0 0 0 200) #x3fffff) ;; panel first
  (let ((font (new 'stack 'font-context *font-default-matrix* 0 0 0.0 (font-color default)
                   (font-flags shadow kerning large middle))))
    (set! (-> font scale) 0.6)
    (set! (-> font origin x) 256.0) ;; centered on the screen
    (set! (-> font origin y) 100.0)
    (pc-encode-utf8-string "Choisissez la zone de guerre" *pc-encoded-temp-string*)
    (draw-string-adv *pc-encoded-temp-string* buf font)
    )
  )
(= (-> *pc-settings* text-language) (pc-language french)) ;; the language the game's texts use
```

Verified: jak3, `engine/gfx/font.gc`, `pc/pckernel-impl.gc`; the battlefront mod's menu pages
(screenshots before and after each fix), 2026-10-31.

## 4.30 Haven City traffic: unit pools, their skins, rebuilding them

- **A unit is spawned once.** The traffic manager fills each type's pool up to its `want-count`
  (`spawn-all`, `get-process *default-dead-pool* <type> #x4000`, `traffic-manager.gc`) and trims
  the inactive ones above it (method 18). A unit that dies sends `'child-killed` and goes back to
  its pool (`go-inactive`, `citizen/citizen.gc`); it comes out again later and is never respawned.
  What its init picked (the skeleton of `init-enemy!`, its mesh masks) stays for its whole life.
  The pools still churn: the manager trims and spawns units as the wanted counts move (faction
  state, sections), so a mod that gives new units a look in turn drifts (a battlefront guard pool
  went from 7 blue of 14 to none); count the live units of each look and give the rarest. Rarest
  in proportion to each look's share (live / share, compared as `live_a * share_b < live_b *
  share_a`), not in units missing: with shares 10 blue, 4, 3, 2, 1 red, the units missing gave the
  first six units of a new pool the blue look, and the traffic wakes a pool's first units first.
- **Sleeping units.** A pooled unit waits in state `inactive`, hidden, at its last place or at the
  origin. A search for live units skips the state name `inactive`.
- **A pool sleeps 20 units at most.** Each type's sleeping units are listed in 20 slots of
  `inactive-object-array` (580 = 29 types times 20, `reset-and-init-from-manager`), and
  `add-reserved-process` lists none past 20. With a larger `want-count`, every unit past 20 goes
  unlisted, the pool counts too few and spawns another every frame, and each new one keeps a
  nav-control on the default nav mesh (4.44): at 44 guards it rose by 30 a second until "too many
  users for nav-mesh #f" crashed the game; at 22 it leaked a few dozen. Give the type an array of
  its own (`(set! (-> engine object-type-info-array 6 array) ...)` after the stock loop) and raise
  the limit of `add-reserved-process` for it.
- **The attack controller holds 64 attackers.** `initialize-cty-attack-controller`
  (`levels/wascity/cty-attack-controller.gc`) calls `(init! *cty-attack-controller* 64 17 22)`:
  every unit, the player and the city's own take one. Past 64, `allocate-attacker` prints "Unable
  to allocate new attacker for mission squad!", the units without one corrupt memory (invalid type
  pointers), and the game fell to 4 frames a second with 80 units. 128 ran at 60.
Verified: jak3, `levels/city/traffic/traffic-engine.gc`, `traffic-manager.gc`, `levels/wascity/cty-attack-controller.gc`, `mission-squad-control.gc`; the battlefront mod's round 20 runs, 2026-11-09.
- **Rebuilding the pools: respawn the player like a death.** A death in the game calls
  `initialize!` with `'life` (`target-death-reset`, `engine/target/target-death.gc`), whose
  `reset-actors` (`engine/entity/entity.gc`) kills every entity actor and every process of
  `*entity-pool*` and `*pusher-pool*` (the traffic manager and its units among them), then runs the
  activate function of each active level: `ctywide-activate` restarts the traffic manager, and each
  unit level (ctypesa, ctypesc, ctypepa...) gives it its types from `lwide-activate`
  (`*traffic-info*` `traffic-object-levels`, copied into the engine by `update-level-info`), so
  every pool is spawned again. Do the same from a mod:

  ```lisp
  (set-continue! *game-info* (get-continue-by-name *game-info* "ctyinda-bbush-1") #f)
  (initialize! *game-info* 'life (the-as game-save #f) (the-as string #f) (the-as resetter-spec #f))
  ```

  A `start` alone does not reset the actors: to a continue of an already active Haven City it
  does not even run `ctywide-activate`. `ctywide-activate` called on its own leaves the restarted
  engine with no unit level, so no pool (the unit levels stay active and do not run
  `lwide-activate` again); `update-level-info` after it brings the pools back (all seen in game).
- **Waking a pedestrian type.** `activate-one-vehicle` (the pedestrian picker, `traffic-engine.gc`)
  wakes the type furthest below its `target-counts` entry for the section's faction state
  (`cty-faction-manager` method 10). The faction manager writes them for its own types only:
  another type (wastelanders brought into Haven City) needs its three `target-counts` as well as
  its `want-count`, or it never wakes.
- **Wastelanders do not hold up in Haven City's traffic.** Woken that way (code and art in
  CTYPESA, a pool of 8), they went with three failures in the battlefront mod's long runs: two
  freezes and one crash of the engine thread inside the background collision query
  `fill-bg-using-line-sphere-new` (mips2c, `game/mips2c/jak3_functions/collide_hash.cpp`). No run
  without them failed there. The cause inside was not found: keep them out of Haven City's
  traffic.
- **No process-tree walk in a unit's init.** The traffic manager spawns its units on its own small
  stack (`stack-size-set!` 128 in `traffic-manager.gc`), so `init-enemy!` runs there. A walk of the
  whole process tree from a hook in `init-enemy!` ended the game (illegal instruction) as a zone
  loaded; the same walk from the mod's own process every frame ran ten minutes. Count in your own
  process and only read the counts in the init.
- **Capping a pool.** For the pedestrian and robot types (traffic types below 17: guards, Metal
  Heads, roboguards, spydroids...), `activate-by-type` wakes a unit while `active-count` is below
  `want-count`, without looking at `target-count`; `want-count` is also the pool size the manager
  spawns and trims to. `traffic-manager-method-22` writes every `want-count` from the faction
  spawn levels (`*want-count-levels*`) every frame, so a cap set from outside lasts one frame:
  change the value in that loop (one `define-extern`'d call there). The battlefront mod caps
  guard-a 14, flying-turret 6, roboguard 2, spydroid 2; units awake on the street then stayed at
  12 per side or fewer.
- **Waking units.** `activate-one-vehicle` (the unit picker, `traffic-engine.gc`) wakes a type
  only while its awake count is below its `target-counts` entry for the faction mode of the
  segment's territory (`cty-faction-manager` method 10), and the squad controls
  (`ff-squad-control.gc`, `kg-squad-control.gc`) rewrite those entries from their alert state: at
  the Port they allowed 7 guards, 1 flying turret and no roboguard, whatever the pool size. The
  traffic also wakes units in two places only: on the segments of cells that just became active
  (`handle-new-vis-cell`), and each frame on one random segment entering the active cells from an
  inactive one (`update-traffic-amount`). Around some places (the Port's `ctyport-warzone` and
  `ctyport-bombbot`) such segments are rare: with the player standing, dead units were not
  replaced and 13 of 14 guards slept. A mod that needs a steady crowd can wake units itself:
  search the pedestrian segments of an area with `callback-on-nav-segments-in-sphere` (tracker 1),
  keep one that passes the stock tests (`nav-mesh-id` nonzero, branch `density` below 128,
  `can-dest-be-used?` false) away from the player, and call
  `(activate-by-type (-> *traffic-engine* tracker1) type segment 0.0)` on it, which keeps every
  stock check of a spawn. Choose the type yourself: the stock picker `activate-one-vehicle` weighs
  every traffic type and left guards asleep in the Industrial Zone. `activate-by-type` returns
  nothing and can refuse a spawn (a unit on the spot, the territory's faction rules): compare the
  type's `active-count` before and after, and try another segment. The battlefront mod wakes up to
  three units four times a second within its battlefield, three segments tried for each: its armies
  of 20 per side (pools 22 guards, 10 flying turrets, 4 roboguards, 4 spydroids) reached 20 per side
  in all four of its zones; one unit a tick, with no retry, held them at 14 to 19. Some places have no unit lanes at all: in the town center around `ctygenb-bbush-1` and
  `ctygenb-bbush-3` no segment passed, and no unit ever woke.
- **Lane data lives in each city section.** The traffic's segments come from the
  `city-level-info` of each loaded level's `bsp` (`ctyport`, `ctyinda`...), registered in the
  engine's two `level-data-array` slots when the level logs in. `callback-on-nav-segments-in-sphere`
  only sees the sections loaded: from the Port, a sphere over the Industrial Zone, the Slums or
  Freedom HQ returned no segment. A mod that needs another area's lanes must read them while that
  area is loaded (the battlefront mod read each zone's once in game and stores them).
- **Two sections in the stock game.** Haven City keeps ctywide and two city sections loaded, as
  the traffic engine's two `level-data-array` slots do; walking toward a third unloads one. A third
  section linked anyway gets no slot (`level-link` prints `level-data not found`) and no lanes. The
  battlefront mod holds all six street sections in a battle: six more entries appended at the end
  of `traffic-engine` (its fields have fixed offsets), which the engine's loops over the sections
  read through one macro; the level slots, banks and draw lists it also needed are in 4.15. A
  section's edge
  can stand on the next section's collision: at the Port's war zone (`ctyport-warzone`, vis-nick
  `ctyport`) Jak had ground only with `ctyinda` loaded, which the stock loads there.
- **Which section a lane belongs to.** `level-link` points each segment's `branch` into its
  section's nav-graph `branch-array`; a lane belongs to the section whose array holds its address
  (`branch-count` entries of 16 bytes). A unit's lane is its `controller` `branch`. `nav-node`
  `level` is read by `deactivate-all-from-level`, but nothing in the code writes it. The
  battlefront mod keeps its reinforcements and its units kept out of view on its district's lanes
  this way: 0 awake units on another section's lanes in each zone.
- **`start` while the city alias changes.** A `borrow` setting that changes Haven City's alias
  (ctywide-ff, -kg, -ff-kg) makes `start` reload ctywide; Jak can be placed before the ground's
  collision is back, fall, and die some 25 m down. Hold him at the spawn (`move-to-point!` each
  frame while he is not `on-surface` and below it) and count him landed in `target-stance` only:
  the continue state leaves a stale `on-surface` bit.

Verified: jak3, `levels/city/traffic/traffic-manager.gc`, `traffic-engine.gc`,
`citizen/citizen.gc`, `levels/city/common/ctywide-init.gc`, `levels/wascity/traffic-util.gc`
(`callback-on-nav-segments-in-sphere`); the battlefront mod's in-game runs (pools respawned with
the battle's skins on half of the 13 guards, stock skins after the battle, wastelanders from 0 to 8
active, deployments without a fall; lanes of each war zone read only once it was loaded: 43
segments at the Port from `ctyport`, 188 in the Industrial Zone from `ctyinda` and `ctyport`),
2026-10-31; `traffic-engine.gc` (`level-link`, `deactivate-all-from-level`,
`sphere-in-loaded-city-infos?`); the battlefront mod's district runs (ground at `ctyport-warzone`
only with `ctyinda` loaded; 0 awake units on another section's lanes in each of its four zones),
2026-11-01.

## 4.31 A model over the game: HUD icons, a preview in front of the camera

**The HUD's models are not drawn by Merc2.** `hud-create-icon` (`engine/ui/hud.gc`) gives a
`manipy` the draw function `dma-add-process-drawable-hud`, which calls `foreground-draw-hud`
(mips2c), which calls `foreground-generic-merc`: the PS2 generic renderer, whose output the PC
renderer draws as direct GS packets (`hud-draw-pris2`, a `ProgressRenderer`). A build-actor model
has no data for it: the battlefront mod's red guard drawn that way crashed the game in native code
at once (a stock model, the robotguard, drew). Stock icons also flip the model with a negative y
scale (`hud-skill` 0.009 x -0.018).

**Draw it with the world instead, in front of the camera.** `*math-camera*` `trans` is the eye,
`inv-camera-rot` rows `rvec`, `uvec`, `fvec` its x (pointing to the screen's left, see below), up and
forward axes.
`transform-point-vector!` gives a world point's screen position in the 512 x 416 screen, offset
by (1792, 1840) (as `engine/ui/bigmap.gc` does), and #f off screen. To show a model at screen
point (x, y), h screen units tall: take the point d meters ahead (`trans + fvec * d`), project it
and the points 1 m to its right and above it, which gives the screen units per meter at that depth
across and up (`per-x` is negative: `rvec` points left on screen), move the point by
`(x - sx) / per-x` meters along `rvec` and `(sy - y) / per-y` along `uvec`, and scale the model
by `h / per-y / model-height`. Face the camera with
`(forward-up->quaternion q (vector-negate! v fvec) uvec)`.

Place it after the frame's camera is set, not from a process. The display loop runs
`execute-math-engine` (the joints of every `ja-post` of the last process run), then
`update-camera` (`*math-camera*` from the camera processes), then the drawing (`main.gc`); the
processes run after that. A model placed from a process uses the previous frame's camera and is
drawn with joints computed before the camera moved: it trails a moving camera. Connect the model
to `*cam-post-hook-engine*` (run right after `update-camera`, 4 slots, a dead process's
connection goes with it) and, there, place it again and redo its joints:

```lisp
(add-connection *cam-post-hook-engine* self my-cam-post #f #f #f)   ;; in its init
(defun my-cam-post ((proc process))
  (place-in-front-of-camera proc)
  (do-joint-math (-> (the-as process-drawable proc) draw) (-> (the-as process-drawable proc) node-list)
                 (-> (the-as process-drawable proc) skel))
  0)
```

Measured with the camera swung at full right stick and after a camera teleport: the drawn model
stayed within 0.06 screen units of its spot; placed from the process run, a screenshot taken
while the camera settled showed it half its height off.

Model heights in game meters, from the bind pose of the rips: crimson-guard 3.8 (the Jak 2 red
guard the same), Jak 2 jetpack guard 4.3, wastelander 3.1, roboguard-city 4.8, spydroid 2.0,
Jak about 3. Taking human heights (2 m) draws a guard twice too large.

Such a model is lit by the level, and scenery nearer than d covers it. HUD sprites are drawn after
the world, so they cover it too: a HUD sprite with `scale-x` 0 is skipped (`hud` `draw`), which
lets a mod clear room (Jak's health HUD keeps its pips with sprites 0 to 4 at 0; stock scales
11.0, 8.0, 8.0, 1.0, 1.3). A hidden HUD shows again on `'show`; it stays 2 seconds, so a mod that
wants it up sends `'show` every second.

**A copy of Jak** needs `(target-skel-group)`, the skeleton group of his current outfit's geometry
(as his mirror image, `logic-target.gc`): `skel-jchar` alone draws one flat quad. The outfit has
cloth (`skel-jchar-c`): `initialize-skeleton` makes the cloth instances, so give the process a
large heap (`:stack-size #x20000`, as the mirror) and set their flags `need-reset active`
(`target-continue-init-art`). With a 16 KB heap nothing was drawn.

A process that must move while a menu pauses the game clears the `menu`, `pause` and `progress`
bits of its mask and runs on `(-> *display* real-clock)` (moving the clock's `ref-count`, as
`hud-init-by-other` does).

**A preview on black.** `drawable.gc` copies `(-> *display* vu1-enable-user-menu)` into the
frame's renderer mask every frame (the stock progress menu's "menu mode" sets it to merc alone,
`blit-displays.gc`). Setting it to the foreground renderers only turns off the sky, the scenery,
the ocean, the particles and the shadows; the screen behind is black. The other models still draw:
set `no-draw` on every drawn `process-drawable` but the preview, and keep their handles to clear
the flag on exactly those afterwards (a model hidden before must stay hidden).

```lisp
(set! saved (-> *display* vu1-enable-user-menu))
(set! (-> *display* vu1-enable-user-menu) (vu1-renderer-mask generic merc emerc))
(iterate-process-tree *active-pool* (the-as (function object object) hide-one) *kernel-context*)
```

**Jak's health HUD and its eco gauges.** `hud-health` draws its sprites in index order with depth:
0 the dark disc, 1 and 2 the light and dark eco gauges (white squares with a negative `scale-y`
from the eco amount), 3 the ring, 4 the shine, 7 to 22 the pips. The ring writes its depth on its
whole square, so its windows shape gauges drawn before it, and a sprite drawn after it (5 or 6,
which keep the `#f` texture of `hud-init-by-other`) is hidden wherever the ring is; 5 and 6 only
show with the ring hidden (`scale-x` 0). To show a gauge of the mod's own in the ring, take one of
the stock gauges: a one-line hook in `hud-health` `draw`, just before `((method-of-type hud draw)
this)`, can set sprite 1's or 2's `scale-y` (-9.8 or -9.7 for a full one) and color after the
stock code set them. The stock pulse, `0.5 + 0.5 sin`, takes the color to black: keep it at 0.5
or more for a gauge that must stay readable. The `endless-dark` and `endless-light` secrets keep
Jak's eco, so the stock gauges, full (4.2).

**A mod's weapon in Jak's gun HUD.** `hud-gun` (top right: the gun icon, a column of shells and an
"ammo/max" count) shows only while Jak's gun is out: its `update-values!` copies
`*target* gun using-gun-type` into `values 0` and that gun's ammo into `values 1`, and hides the
HUD otherwise. A `define-perm` wrapper of that method can feed it instead, while the mod needs it:
set `(-> this values 0 target)` to a gun pickup-type (26 to 37: 29 yellow 1, 33 blue 2, 28 red 3),
`values 1 target` to the ammo, clear `(hud-flags disable)`, set `(hud-flags show)`, then call
`(method-of-type hud update-values!)`; call the stock method the rest of the time, so it hides
again. The stock `draw` picks the textures, scales, shells and count from those two values. With
`(set-setting! 'gun #f ...)` its gun arrows stay hidden.

Verified: jak3, `engine/ui/hud-classes.gc` (`hud-gun` `draw`, `update-values!`), `engine/ui/hud.gc`;
the battlefront mod's guards (yellow 1, blue 2, red 3 with their shells and counts, gun HUD
`hud-in`, hidden for a class with no ammo weapon), 2026-11-01.

Counts of the mod's own (a magazine and a reserve, no count at all): wrap `draw` too. Set
`(-> this strings 0 scale)` and `(-> this sprites 14 scale-y)` to 0 for the stock call (sprite 14
is the shell; the stock draw resets its `scale-x`, not its `scale-y`), put them back, then, unless
`(hidden? this)`, draw your shells with sprite 14 placed from `(-> this sprites 0 pos)` with
`set-as-offset-from!` (5 units apart upward, as the stock column) and your text at
`(-> this strings 0 pos)`, scale 0.4. The icon keeps the stock look and slide.

Verified: jak3, `engine/ui/hud-classes.gc`; the battlefront mod's guards: the rifle "10/20" over 10
shells, the grenades "3" over 5 shells, the tazer's icon alone (screenshots), 2026-11-06.

**One DMA bucket at a time.** A `with-dma-buffer-add-bucket` opened inside another one's body, on
the same frame buffer, froze the game: draw sprites and text in separate blocks, one after the
other.

Verified: jak3, `engine/ui/hud.gc`, `engine/draw/drawable.gc`
(`dma-add-process-drawable-hud`), `game/mips2c/jak3_functions/foreground.cpp`,
`game/graphics/opengl_renderer/OpenGLRenderer.cpp`, `engine/target/target-util.gc`
(`target-skel-group`), `engine/process-drawable/process-drawable.gc`, `engine/gfx/blit-displays.gc`,
`engine/ui/hud-classes.gc` (`hud-health`); the battlefront mod's class page preview, health HUD
icon and gauges, and its pages on black, measured and seen in game, 2026-10-31; its class gauges
in the stock gauges and its gun icon (a sprite 5 gauge hidden by the ring, a nested bucket freeze),
2026-11-01.

## 4.32 Stopping attacks on the player: the shield level

The health ring lights `(the int health)` segments (`hud-classes.gc` `hud-health` `update-values!`):
a hit that leaves less than 1 shows an empty ring with the player alive. To make an empty ring
mean death, raise such a hit's damage to the health left. Verified: the battlefront mod's
`target-attacked` wrapper, a blow at 3.6 of 8 killing, 2026-11-11.

While `(-> *target* fact shield-level)` is above 0, `target-attacked`
(`engine/target/target-handler.gc`) ignores every attack and flashes `(-> *target* lightjak
shield)` with `'hit`; the `'touched` handlers turn an enemy's touch into a `'shield` attack on it.
That is all of Light Jak's shield the rest of the game sees, so a mod gets the same shield by
setting the level and spawning its model like `target-lightjak.gc` does:
`(process-spawn lightjak-shield :init lightjak-shield-init :name "lightjak-shield" :to *target*)`
stored in `(-> *target* lightjak shield)`, `'open` to raise it again, `'close` to fade it out (it
dies at the end). Its `hit` state takes light eco from the player on each stopped hit.

To stop some attacks only, wrap `target-attacked` (a GAME function: `define-perm` the stock one,
[4.10](#410-wrapping-a-stock-method-from-a-game-file)) and set the level before calling it, from
the attacker: shots and blasts arrive from a `projectile` or an `explosion` process (`guard-shot`
mode `guard-shot`, `explosion` mode `explode`), blows from the attacker itself (a crimson-guard,
mode `generic`). Do not restore the level after the call: the stock handler may change state
(`go target-hit`), and the code after it may not run.

Verified: jak3, `engine/target/target-handler.gc`, `engine/target/target-lightjak.gc`; the
battlefront mod's robotguard shield, with a logging wrapper (guard shots and a blast in at level
1 and stopped, a guard's blow in at level 0), 2026-10-31.

**A shield that breaks: the Dark Makers' sphere.** `shield-sphere` (`engine/common-obs/shield-sphere.gc`,
GAME code; its three art groups are in CTYPESA, also in the far levels of the `dp-bipedal`
troopers) is a process with a collision sphere: shots hit it and heat it by damage over strength,
the heat decays at 0.2 a second, and past 1.0 it goes to `explode` (its shard model). Around the
player, use a subtype with the mech shield's collision (`target-mech.gc` `mech-shield`: collide-as
`obstacle impenetrable-obj`, no collide-with, no action): the stock one is deadly to Jak. Spawn it
`:to *target*` with a `shield-sphere-spawn-params` (owner `*target*`, `track-joint` -1 and an
`offset-vec` up, `shield-type-0`, a strength), from an init that first sets `(-> self level)` to
the level holding the art and then calls `shield-sphere-init-by-other`: `initialize-skeleton` looks
the art up in the process's level, and without it the process goes straight to
`process-drawable-art-error`. Its stock post tints it red as it heats (`color-emissive`); a
`shield-post` override that calls the parent can keep a constant red. `'active` answers #t while it
stands and nothing once it explodes or dies. Its own collision stops the player's shots too (5.9 m
out from a shield of 3 m around the player). With the shield level (above) stopping the attacks, a
sphere for the look alone takes no collision: after the spawn, set its root's `backup-collide-as`
and `backup-collide-with` to `(collide-spec)`, and its root prim's: its states take the collision
back from the backup (`toggle-shield`). Verified: the battlefront mod's robotguard, its shots 19 to
100 m out behind the shield, 2026-11-11.

Verified: jak3, `engine/common-obs/shield-sphere.gc`, `engine/target/mech/target-mech.gc`,
`goal_src/jak3/dgos/ctypesa.gd`; the battlefront mod's robotguard shield (strength 3: a guard shot
heated it to 0.42, the second broke it into red shards, Jak's health untouched), 2026-11-01.

## 4.33 Holding the player in the air: a jetpack

A mod can fly Jak by setting `(-> *target* control transv y)` every frame while he is in the air;
his air state integrates it with gravity after. To hold a height, set it from the gap to that
height rather than switching a thrust on and off (which jerked up and down): `4.0 * (held - y)`,
clamped, settles within a second and stays still (the battlefront mod's jetpack held 10.4058 m to
the 0.1 mm for 4 s). Keep a faster jump's own speed on take-off (`fmax` with the climb speed), and
cap the fall when the fuel is out (`fmax` with minus the sink speed) for a slow descent.

Do not trust `(collide-status on-surface)` alone to tell he landed: held 2 m above the ground, it
came on for a frame about once a second, which reset the held height each time and let the
jetpack sink a step. Count him landed only when it is set and his feet are within half a meter of
the ground probe's point, `(-> *target* control gspot-pos y)`.

The flag also goes off for single frames on the run: a model that picks a jump clip from it alone
flashed that clip (the battlefront mod's spider robot looked as if it hopped). The same test, off a
surface and more than 0.6 m above `gspot-pos`, keeps the run clip.

**Holding the player still.** `speed-mult` (`user-default`, read every frame) scales his surfaces'
`target-speed` and `transv-max` (`engine/target/logic-target.gc`); his walk then divides by them:
with 0 the game crashed once while he walked. A small value (0.01) and his horizontal `transv` set
to 0 each frame on the ground hold him (0.1 m/s with the stick pushed); `(set-setting! 'jump #f 0.0
0)` stops his jump.

Verified: jak3, `engine/collide/collide-shape-h.gc` (`gspot-pos`); the battlefront mod's jetpack
(climb at 6 m/s, hover still, Circle down at 7 m/s, out of fuel down at 4 m/s, a 1 Hz sink before
the ground probe test, none after; the spider's clips; the robotguard standing still behind its
shield), 2026-11-01.


## 4.34 The player hidden: his bones, and his collision

`no-draw` (the target's `'draw #f` event sets it) hides a model, and also stops its bones:
`evaluate-joint-control` skips its clips and `do-joint-math` returns at once while `no-draw` or
`no-draw-temp` is set (`engine/process-drawable/process-drawable.gc`). The player's collision
follows his bones: hidden by a mod that draws another model in his place, Jak walked as usual, but
once moved by `move-to-point!` or by a stock respawn (`target-continue`) he fell through the ground
at many places (5 of the Port's 9 continue points). The same points held him outside the mod's
hiding, and with his bones computed each frame while hidden:

```lisp
;; once a frame, after the player moved and before drawing (a *cam-post-hook-engine* connection)
(when (and *target*
           (= *master-mode* 'game)
           (not (focus-test? *target* dead))
           (nonzero? (-> *target* skel))
           (> (-> *target* skel active-channels) 0)
           (not (logtest? (-> *target* draw status) (draw-control-status uninited))))
  (let ((hidden? (logtest? (-> *target* draw status) (draw-control-status no-draw))))
    (logclear! (-> *target* draw status) (draw-control-status no-draw))
    (update-anim-data (-> *target* skel)) ; as Jak's edge-grab code does (logic-target.gc)
    (do-joint-math (-> *target* draw) (-> *target* node-list) (-> *target* skel))
    (if hidden?
        (logior! (-> *target* draw status) (draw-control-status no-draw)))))
```

Keep the tests: the same math without them, run from the frames after a menu page closed on a
deployment, crashed the game (twice, at the same moment); with them it ran through the battlefront
mod's proof runs.

Verified: jak3, `engine/process-drawable/process-drawable.gc` (`evaluate-joint-control`,
`do-joint-math`), `engine/target/target-handler.gc` (`'draw`), `engine/target/logic-target.gc`;
the battlefront mod's runs: Jak hidden, falling through at the Port's `ctyport-start`,
`-bbush-1`, `-bbush-5`, `-gungame`, `-bbush-8` after `move-to-point!` and after `'continue`; a ray
down found the ground 0.01 m under each; standing and running 20 to 30 m from each with Jak drawn,
or hidden with the bones computed, 2026-11-01.

## 4.35 Build-actor models: their effects, the death cloud, another mesh at runtime

**No mesh segments.** `build-actor` writes a 0 `seg-table` (`goalc/build_actor/jak3/build_actor.cpp`),
and `setup-masks` builds the effects it hides from that table (`process-drawable.gc`: a set
`seg-mask` bit hides a segment, `effect-mask` gathers the hidden segments' effects): with no table
the effect mask stays 0 and the whole model shows, whatever the stock code masks. A stock unit's
armor, weapon or dark-guard variant masks do nothing on a build-actor model; a stock model's
segments keep working.

Verified: jak3, `engine/process-drawable/process-drawable.gc` (`setup-masks`),
`goalc/build_actor/jak3/build_actor.cpp`; the battlefront mod's runs, the guards' `seg-mask` read
in game, 2026-11-06.

**A death without the merc dissolve.** A build-actor model cannot take the `death-default` effect
(the merc dissolve crashed in `mercneric_convert`, above). Its look: the dissolve launches part 53
of `*part-id-table*` (`merc-death.gc`, global) from each vertex with the sound `enemy-fizz`;
`(launch-particles (-> *part-id-table* 53) pos)` from a few joints each frame for a second, then
the model hidden, looks the same. The Krimzon Guard explosions are global too
(`engine/common-obs/enemy-part.gc`): `group-kg-explosion` (the robotguard's, with `robo-explode`)
and `group-kg-small-explosion` (the flying turret's, with `fly-turret-die`), spawned with
`part-tracker-spawn part-tracker :to self :group ...` after setting `(-> *launch-matrix* trans)`.
Verified: the battlefront mod's player deaths (guard, robotguard, jetpack, screenshots), 2026-11-11.

**Particles placed by the stock code.** A stock unit's code puts its particles at fixed offsets from
its root: drawn as another model, they miss it. The flying turret draws its two thruster flames
(`draw-beam` of part 4709 in `flying-turret-method-229`) under its root; on Jak 2's hover guard
(`crimson-guard-hover-lod0`, built for the flying-turret code), a hook just before each `draw-beam`
moves them to its turbines, node-list 15 (`Lturbine`) and 17 (`Rturbine`), along each joint's -z
axis (`vector<-cspace!`, then the bone transform's `vector 2` negated). The mod Haven City: New
Dawn did it first.

Verified: jak3, `levels/city/protect/flying-turret.gc`; the battlefront mod's screenshot of a red
jetpack guard, both flames at its turbines, 2026-11-06.

**No effect control.** A model built by `build-actor` has no `effect-name` lump, so
`initialize-skeleton` leaves `(-> skel effect)` at `#f` (4.12): the effect frames of the clips it
plays (footsteps, gun sounds, `death-default`) never run, even when its draw borrows a stock art
group for its clips. Make one by hand right after `initialize-skeleton`, on the process's own heap
(from its init, before the heap is shrunk to fit):

```lisp
;; after (initialize-skeleton this <skeleton-group> (the-as pair 0)), in the process's init
(when (not (-> this skel effect))
  (let ((ec (the-as effect-control ((method-of-type object new) 'process effect-control
                                                             (the-as int (-> effect-control size))))))
    (set! (-> ec process) this)
    (set! (-> ec last-frame-group) #f)
    (set! (-> this skel effect) ec)))
```

The process heap walk and `relocate` (`engine/entity/relocate.gc`) then move it like a stock one.

**`death-default` crashes a custom model.** The effect sets the draw's `death-timer`, which sends the
mesh to the generic renderer for 15 frames (`merc-death.gc`; on PC, `foreground.gc`'s reassign to
merc keeps generic for a nonzero `death-timer`). A build-actor model has only a dummy `merc-ctrl`
for it (`goalc/build_actor/jak3/build_actor.cpp` `generate_dummy_merc_ctrl`, fake fragments), and
`gk` crashes in `mercneric_convert` (mips2c `generic_merc.cpp`, a halfword store at offset 18 of a
null pointer). The effect only runs once the enemy's death has set `effect-control-flag ecf1`
(`enemy-states.gc`), from the death clip's `'event-death`. Two ways out:

- clear the timer when the enemy gets `'death-start` (`enemy.gc`'s handler, after the stock flags)
  and launch the cloud yourself (below);
- or, when the units run stock code you cannot hook per unit (Haven City's guards), skip the death
  effects of your models in a `do-effect` wrapper: the body then goes with its death clip, with no
  dissolve.

```lisp
(define-perm *my-stock-do-effect* (function effect-control string float int none)
             (method-of-type effect-control do-effect))

(defmethod do-effect ((this effect-control) (arg0 string) (arg1 float) (arg2 int))
  ;; arg0 is a symbol or a string (effect-control.gc)
  (let ((name (cond ((= (rtype-of arg0) symbol) (symbol->string (the-as symbol arg0)))
                    ((= (rtype-of arg0) string) arg0)
                    (else (the-as string #f)))))
    (if (and name (string-prefix= "death-" name) (-> this process)
             (my-build-actor-model? (-> this process)))  ;; e.g. its draw jgeo name
        (none)
        (*my-stock-do-effect* this arg0 arg1 arg2)))
  (none))
```

The cloud, when you keep the effect for a mod's own `nav-enemy`. Part 53, the purple spark, has no
texture of its own: `sp-launch-particles-death` gives it the death texture, so launch it through
`merc-death-spawn`; `launch-particles` with part 53 draws nothing:

```lisp
;; one frame of the cloud: a spark by each joint, scattered by up to 30 cm
(let ((pos (new 'stack-no-clear 'vector)))
  (dotimes (i (-> pd node-list length))
    (vector<-cspace! pos (-> pd node-list data i))
    (+! (-> pos x) (rand-vu-float-range -1228.8 1228.8))
    (+! (-> pos y) (rand-vu-float-range -1228.8 1228.8))
    (+! (-> pos z) (rand-vu-float-range -1228.8 1228.8))
    (merc-death-spawn 53 pos pos)))
```

Run it for 15 frames from a small process holding the dying one's handle, and set the draw's
`no-draw` status from the fourth frame, as the generic renderer hides the mesh.

**A `nav-enemy` playing a guard's death clip.** The clip does not run `death-default` itself: it
sends its process `'event-death`, and Jak 3's guard runs the effect from its event handler
(`crimson-guard` `event-handler`, `levels/city/traffic/citizen/guard.gc`). A `nav-enemy` of a mod
that borrows the guard's clips ignores the event, so it dies without the cloud. Handle it the same
way in the mod's own `event-handler` (before passing the other events to `nav-enemy`'s):

```lisp
(('event-death)
 (if (and (= (-> this hit-points) 0.0) (-> this skel effect))
     (do-effect (-> this skel effect) "death-default" 0.0 -1)))
```

**Another mesh at runtime.** Two build-actor models made from the same rip (a recolored copy) share
their skeleton and clips: a process can switch to the other one without a second
`initialize-skeleton`, which would allocate on a heap already shrunk. Look the art group up in the
process's level, rebuild the lods from it, and set `mgeo` yourself: `lod-set!` only sets it when the
lod index changes, so with one lod the old mesh stays drawn:

```lisp
(let ((ag (load-to-heap-by-name (-> this level art-group) (-> sg art-group-name) 'test global
                                (-> sg version))))
  (when (and ag (setup-lods! (-> this draw lod-set) sg ag (-> this entity)))
    (set! (-> this draw art-group) ag)
    (set! (-> this draw mgeo) (-> this draw lod-set lod 0 geo))))
```

Both art groups must be in the process's level (`'test` only looks; it loads nothing).

**A particle takes the clock of the process that launches it.** The time-of-day process launches
the weather's rain and snow on `bg-clock` (`*display*` clock 9): a mod that stops that clock to hold
the hour freezes the drops in the air. Give them `part-clock` (8) back, or launch from a process on
another clock.

Verified: jak3, `engine/gfx/foreground/merc/merc-death.gc` (`death-default`, part 53,
`merc-death-spawn`), `engine/gfx/sprite/particles/sparticle-launcher.gc`
(`sp-launch-particles-death`), `engine/process-drawable/process-drawable.gc` (`setup-lods!`,
`lod-set!`, `skeleton-group->draw-control`), `engine/ai/enemy.gc`; Haven City: New Dawn's war mode
(Jak 2's Krimzon Guards as build-actor models on Jak 3's `crimson-guard`): the crash in `gk` on the
first `death-default`, then the cloud invisible with `launch-particles`, seen in screenshots through
`merc-death-spawn`; its guard turrets switched from red to blue (the art group changed, the red mesh
drawn until `mgeo` was set), 2026-11-04. The crash site and the `do-effect` wrapper: jak3,
`engine/gfx/foreground/foreground.gc` (the PC reassign), `engine/game/effect-control.gc`
(`do-effect`, `ecf1`), `engine/common-obs/enemy-states.gc`; the battlefront mod (red guards on the
stock guard code with the guard's clips and a hand-made effect-control): three crashes seconds into
battles, a vectored exception handler at `gk.exe` +#x383fe6, which disassembles to
`mercneric_convert`'s `sh t1, 18(a3)` with `a3` 0; with the wrapper, smoke runs of 131 s and 184 s
with 23 and 41 losses of that side and no crash, 2026-11-05.

## 4.36 The Mods menu: a greyed-out flag

Since master-dev `9c3fc0d4c`, the shared popup menu (`pc/util/popup-menu.gc`, Jak 2 and Jak 3) honors
an entry's `entry-disabled?` everywhere: while it answers `#t`, the cursor skips the entry, it is
drawn grey (a `popup-menu-flag` without its check mark), and `confirm!` ignores it, even when the
cursor rests on it as the first row. A plain `popup-menu-flag` with an `:entry-disabled?` lambda is
enough. Jak 1's port always did all three ([2.4](jak1.md#24-in-game-mods-toggle-the-same-registry-a-smaller-popup)).

A mod branch older than that master-dev draws a disabled flag like an enabled one and runs its
`on-confirm`: give the flag a subtype whose `draw-entry` calls `popup-menu-entry`'s while it is
disabled, and test the condition again in `on-confirm` (Haven City: New Dawn's `hj2-menu-flag`).
Merging master-dev makes both unnecessary.

Verified: jak2 compiled, jak3 written to the same lines (`draw-entry` of `popup-menu-flag`,
`confirm!`), 2026-11-04; the subtype: Haven City: New Dawn's war entry, greyed out with the mod off
(screenshot), 2026-11-04.

## 4.37 Forcing a guard's weapon: the off-screen weapon change

A guard's tazer strike is three clips of `crimson-guard-ag` (`guard-tazer.gc`): `stab-idle-to-attack`
(21, the thrust) once, `stab-attack` (22, a 2-frame zap loop) for 0.5 s, `stab-attack-to-stab-idle`
(23) once. Played alone, clip 22 is over in 7 frames. Verified: the battlefront mod's guard class,
the three clips seen in turn, 2026-11-11.

Every weapon change of a Jak 3 guard goes through `crimson-guard-method-267`
(`levels/city/traffic/citizen/guard.gc`), so a mod that forces a weapon puts its hook at the top
of that method. Read its callers too. Off screen, every 16 frames, `crimson-guard-method-270` asks
the squad for another weapon (`ff-squad-control-method-55`), calls method 267 when it differs from
the guard's, and answers `#t`; its callers (`guard-states.gc`) then send the guard to its hostile
state again (`go-hostile`). With the weapon forced, method 267 changes nothing, yet 270 still
answers `#t`: every guard out of view restarts its hostile state 4 times a second. In Haven City:
New Dawn's war (Krimzon Guards forced to the rifle) the game crashed within one or two minutes,
in places unrelated to the guards (the kernel's dispatcher resuming a guard, a transport or the
minimap whose saved thread held floats; a guard whose squad target was a `part-tracker`). Compare
the weapon the hook would give in method 270 instead:

```lisp
;; crimson-guard-method-270, the off-screen change (my-forced-guard-type: the mod's hook, which
;; returns its second argument when the mod is off)
(when (!= (my-forced-guard-type this (the-as int a1-3)) (the-as int (-> this guard-type)))
  (crimson-guard-method-267 this (the-as guard-type a1-3) #f)
  (return #t))
```

**A guard type no guard on foot has kills the guard.** A guard woken with the spawn params'
`guard-type` 11 ("keep yours", a transport's drop, behavior 6) keeps the type `citizen-method-194`
gave it from the squad (`ff-squad-control-method-54` for the traffic type), which can be a vehicle
gunner's. Landing from its jump, `go-hostile` (`guard-states.gc`) picks its state by weapon with a
`case` over rifle, tazer and grenade only: with any other type it goes nowhere, the jump's code runs
to its end, and a process whose state code returns is deactivated (the guard calls `deactivate` on
itself, hit points intact). A weapon hook must map every type that is not a ped weapon to one.

Verified: jak3, `levels/city/traffic/citizen/guard.gc` (methods 267, 270, 194, `go-from-behavior`),
`levels/city/common/guard-states.gc`, `levels/city/common/ff-squad-control.gc`; Haven City: New
Dawn's war mode: with the Krimzon Guards' weapons forced, 8 runs of 17 crashed; with the squad's
weapons, 4 of 4 went through; rifles only, 3 of 4 crashed; with the test in method 270, 16 of 16
went through (60 to 90 s of war each); the blue guards of a transport deactivated by themselves in
state `jump`, 8 of 8, then landing and fighting with the hook mapping every other type to the
rifle or the launcher, 2026-11-04.

## 4.38 A guard that hunted the player: letting it go

Jak 3's attack controller (`*cty-attack-controller*`, `wascity/mission-squad-control.gc`) gives each
guard its enemy. A guard Jak hits gets a hatred callback (`'member-attacked`: `attacker-info`
`hatred-map-callback`, `guard.gc`) that makes the player its enemy; the stock hatred table has no
entry from a guard to the player. `mission-squad-control-method-12` looks for a new enemy only while
the guard keeps some hatred for its current one (a table entry, or the callback). Clearing the
callback alone leaves the player as its enemy for good: the guard goes on attacking, and goes back
to its attack each time `'end-pursuit` sends it to `active` (`active` goes hostile again while the
guard's `target-status` holds a process). Drop the enemy the way `remove-attacker` does, then clear
the target and send `'end-pursuit`:

```lisp
;; g: a crimson-guard; the player's own entry loses one attacker
(let ((ai (-> g attacker-info))
      (ctrl *cty-attack-controller*))
  (when ai
    (set! (-> ai hatred-map-callback) #f)
    (when (= (handle->process (-> ai enemy)) *target*)
      (let ((k (-> ai enemy-index)))
        (if (and (>= k 0) (< k (-> ctrl attackers length)) (> (-> ctrl attackers data k num-current-attackers) 0))
            (+! (-> ctrl attackers data k num-current-attackers) -1)))
      (set! (-> ai enemy-index) -1)
      (set! (-> ai enemy) (the-as handle #f))
      (set! (-> ai enemy-type) -1)
      (logior! (-> ai flags) (city-attacker-info-flag cai2))))
  (if (= (handle->process (-> g target-status handle)) *target*)
      (set! (-> g target-status handle) (the-as handle #f)))
  (send-event g 'end-pursuit))
```

A unit keeps the enemy it chose until something makes it look again; with the `chi1` flag on
the hatred that chose it (no look again on the controller's timer), until one of them dies. A
mod that writes its own `hatred-indices` changes no choice already made: the battlefront mod's
Krimzon Guard units that took the player with the stock table, while the zone loaded under the
mod's menu, kept him after the mod's table made him their ally. Call
`mission-squad-control-method-13` on such a unit's `attacker-info` to make it choose again with the
table as it is now.

Verified: jak3, `levels/wascity/mission-squad-control.gc` (method 13); the battlefront mod's
runs, a red guard and a red jetpack on a Krimzon Guard player, then none, 2026-11-06.

The city's own alert end does not work either: `ff-squad-control-method-45` waits during its ending
for `ff-squad-control-method-48`, the count of guards still in the alert, which is decompiled as
returning `none`: the value is whatever its last expression left.

Verified: jak3, `levels/wascity/mission-squad-control.gc` (methods 12, 13, `remove-attacker`),
`levels/city/traffic/citizen/guard.gc` (`'member-attacked`, `'end-pursuit`),
`levels/city/common/guard-states.gc` (`active`), `levels/city/common/ff-squad-control.gc`; Haven
City: New Dawn's alert: with the callback cleared, 5 guards kept the player as enemy and target for
70 s after the alert ended; with the enemy dropped, none, 2026-11-04.

## 4.39 Haven City traffic: what comes out on which street

- **The street's territory.** Each street of the city's navigation graph is a `nav-branch`, whose
  `clock-type` is its territory (`nav-territory-type`, `city/common/cty-faction-h.gc`: `port-*`,
  `inda-*`, `indb-*`, `sluma-*` to `slumc-*`, `genc*`, `mhcity-*`, `special*`). The faction manager
  groups them in four (`cty-faction-manager-method-16`: port, industrial, slums with genc, Metal
  Head city).
- **Who may come out.** `activate-by-type` (`traffic-tracker`, `city/traffic/traffic-engine.gc`)
  asks the faction manager, for the street's territory and the unit type's level
  (`cty-faction-manager-method-9`, cached; computed by method 14 from the faction, territory and
  global strengths: positive spawns, zero only fights). The same answers decide which borrow levels
  load (method 20: `ctypesa` for the Freedom League's guards, `ctypesc` for the KG's, `ctypesb` for
  the Metal Heads) and which territories are at war (method 13). A hook at the top of method 14
  overrides a territory for every type of a faction; a test in `activate-by-type` itself, which
  knows the traffic type, tells types of one faction apart (a KG robot from a flying turret).
- **Which type comes out next.** `activate-one-vehicle` picks, on a street, the type furthest below
  its count (active / wanted) among those the faction check allows, then `activate-by-type` sends
  one. A type refused only in `activate-by-type` stays the pick: with none out it wins every time,
  and nothing comes out on that street. Test the type in the pick too (Haven City: New Dawn's
  conflict zones: KG robots refused, 1 guard out in 2 minutes; tested in the pick as well, 15 to 17).
- **Which sleeping unit.** `activate-from-params` wakes the first unit of the type's inactive array
  (`get-from-inactive-by-type`). To choose (a look, a side), scan that array, move the chosen handle
  to index 0, then call `get-from-inactive-by-type`, which keeps the pool's counts.
- **The street is only known in `activate-by-type`.** It sets the spawn params' `nav-branch` before
  calling `activate-from-params`; the other callers (`'activate-object` from a parking spot, a
  transport, a mission) leave the field unset, garbage on the stack. Reading
  `(-> params nav-branch clock-type)` in `activate-from-params` crashed on the first parked car.
  Pass the territory from `activate-by-type` instead (a global set around its call).

Verified: jak3, `levels/city/traffic/traffic-engine.gc` (`activate-by-type`,
`activate-from-params`, `get-from-inactive-by-type`), `levels/wascity/cty-faction.gc` (methods 9,
13, 14, 16, 20), `levels/city/common/cty-faction-h.gc`; Haven City: New Dawn's Praxis KG invasion
(Krimzon Guards woken only on the port's streets, by side), and the crash reading the branch of a
parking spot's spawn params, 2026-11-04.

## 4.40 Guards fighting guards: the grenade guard's reload loop

A Jak 3 grenade guard (`crimson-guard`, guard type grenade) reloads in
`grenade-throw-post-reloading` (`levels/city/common/guard-grenade.gc`). Its `trans` breaks the
reload off when its target stands within half its keep-away distance
(`grenade-guard-min-player-dist`): to back away, or after 2 s of backing away straight back to its
grenade state. With another guard as its target (a mod that makes guards fight each other), standing
that close, the states chained within one frame: the reload broke off, the backing away was already
over and went back to the grenade state, which, with no grenade and the guard at its spot, went to
reload again. Each step is a `go` from a state's `trans`, which runs the next state's `trans` at
once, and each reload spawns a new `reload-grenade`: the stack overflowed and the game crashed, in
native collision code (`collide-cache-prim` reading a bad pointer) or as a `reload-grenade` entered
its first state. Break the reload off for the player only:

```lisp
;; guard-grenade.gc, grenade-throw-post-reloading :trans (my-reload-break-ok?: #t out of the mod,
;; else #t only when the guard's focus is the player)
(when (and (>= (/ (-> self grenade-guard-min-player-dist) 2) (-> self target-self-xz-dist))
           (my-reload-break-ok? self))
  ...)

(defun my-reload-break-ok? ((this process-drawable))
  (or (not (my-mod-on?)) (= (handle->process (-> (the-as enemy this) focus handle)) *target*)))
```

Verified: jak3, `levels/city/common/guard-grenade.gc` (`grenade-throw-post-reloading`,
`grenade-throw-post-retreating`); Haven City: New Dawn's war with two extra troop transports: 2
crashes in 7 runs of 3 minutes before, 0 in 3 after, the rule holding a reload 235 and 241 times a
run, 2026-11-04.

## 4.41 A third-person shooter camera, and making Jak jump at a set velocity

**The point-of-view camera on a rig.** The stock `cam-pov` mode (`engine/camera/cam-states.gc`, the
city's sniper turret uses it) looks from a bone of a process: its place is the bone's translation
plus `pov-offset` turned by the bone, its rotation the bone's `rvec`, `uvec`, `fvec`. A mod's camera
can therefore be a small process-drawable with a node list of one cspace on a bone of its own, set
every frame. Keep the bone in the process (an inline field): `cspace-array`'s `relocate`
(`engine/entity/relocate.gc`) moves every cspace's bone pointer with the process when its dead-pool
heap compacts. Spawn it under `*target*`: a child runs after its parent and before `*camera-pool*`,
so the view never lags Jak's move of the frame. Its settings go with it when it dies, and the
stock camera comes back.

```lisp
(deftype my-cam-rig (process-drawable)
  ((cam-bone  bone  :inline))
  (:state-methods idle))

(define *my-cam-offset* (new 'static 'vector :w 1.0)) ;; static: the setting keeps its address

(defbehavior my-cam-rig-init-by-other my-cam-rig ()
  (set! (-> self root) (new 'process 'trsqv))
  (set! (-> self node-list) (new 'process 'cspace-array 1))
  (let ((cs (-> self node-list data 0)))
    (set! (-> cs parent) (the-as cspace #f))
    (set! (-> cs joint) (the-as joint #f))
    (set! (-> cs joint-num) 0)
    (set! (-> cs geo) (the-as drawable #f))
    (set! (-> cs bone) (-> self cam-bone))
    (set! (-> cs param0) (the-as (function cspace transformq none) #f))
    (set! (-> cs param1) #f)
    (set! (-> cs param2) #f))
  (matrix-identity! (-> self cam-bone transform))
  (my-cam-rig-update)                         ;; the bone set before the camera first reads it
  (set-setting! 'mode-name 'cam-pov 0.0 0)
  (let ((pp (process->ppointer self)))
    (set-setting! 'pov-handle pp 0.0 (-> pp 0 pid)))   ;; bone 0
  (set-setting! 'pov-offset 'abs *my-cam-offset* 0)
  (set-setting! 'interp-time 'abs 0.0 0)
  (set-setting! 'allow-look-around #f 0.0 0)
  (set-setting! 'matrix-blend-max-angle 'abs (degrees 360) 0)
  (go-virtual idle))

;; every frame (the idle state's :trans): look along yaw and pitch from behind a pivot
(let ((mat (-> self cam-bone transform))
      (fwd (new 'stack-no-clear 'vector)))
  (set-vector! fwd (* (sin yaw) (cos pitch)) (sin pitch) (* (cos yaw) (cos pitch)) 1.0)
  (forward-down->inv-matrix mat fwd (new 'static 'vector :y -1.0 :w 1.0))
  ;; place = pivot - fwd * distance - rvec * shoulder (rvec points to the screen's left) + rise,
  ;; pulled in front of a wall with fill-and-probe-using-line-sphere from the pivot
  (vector-copy! (-> mat trans) place)
  (set! (-> mat trans w) 1.0))
```

Three facts set the signs. A heading `yaw` looks along `(sin yaw, 0, cos yaw)` (`vector-y-angle`)
and grows toward +x. The camera's `rvec` (`forward-down->inv-matrix`: forward x down) is +x when it
looks along +z, and +x is on the screen's left: the screen's right is `-rvec`, and a turn to the
right lowers the heading. With the PC settings' third-person inversions at their defaults
(`third-camera-h-inverted?` and `-v-` `#t`, the original game's controls), the stock camera turns
the view left for the right stick pushed right; for a shooter, subtract
`analog-input-horizontal-third` and `analog-input-vertical-third` (`engine/ps2/pad.gc`) from the yaw
and the pitch: pushed right the view turns right, pushed up it looks up, and a player who changes an
inversion setting gets it turned the other way. Jak's stick reads the camera's matrix
(`matrix-local->world`, `logic-target.gc`), so the left stick moves him relative to the new view
with no change.

**A jump at a set velocity.** An upward speed written into `(-> *target* control transv)` from
another process does not lift Jak off the ground in his stance. His standard event handler
(`target-handler.gc` `target-standard-event-handler`) takes `'launch` with a minimum and maximum
jump height, a surface (`#f`) and a velocity, copies the velocity and goes to `target-jump`:

```lisp
;; a sideways leap: 0.8 m high, 11 m/s along RIGHT (a level unit vector)
(let ((vel (vector-float*! (new 'stack-no-clear 'vector) right (meters 11))))
  (set! (-> vel w) 1.0)
  (send-event *target* 'launch (meters 0.8) (meters 0.8) #f vel))
```

Keep setting the across speed each frame while he flies, or his air control bends the path.

Verified: jak3, `engine/camera/cam-states.gc` (`cam-pov`), `engine/game/settings.gc`
(`pov-handle`, `pov-offset`), `levels/city/sniper/cty-sniper-turret.gc`,
`engine/geometry/geometry.gc`, `engine/ps2/pad.gc`, `engine/target/target-handler.gc`; the
battlefront mod's smoke runs: the camera mode `cam-pov` on the rig, the crosshair's ray along the
view (dot 1.0000), the class model at x 171 of 512 with the camera moved along `-rvec` (x 341 along
`+rvec`), the stock camera turning -0.94 to the right for the stick pushed right, the rig +0.99;
the leap 1.18 m along the ground with an upward speed set by hand, 3.80 m with `'launch`,
2026-11-05.

## 4.42 Haven City: where a district's streets leave it, a force field across them, a light pillar

- **A section's exits are in its nav graph.** Each city section's lane graph,
  `(-> lev bsp city-level-info nav-graph)` (`levels/city/common/nav-graph-h.gc`), lists in its
  `link-array` (`link-count` entries) the lanes that leave it for another section. A
  `nav-graph-link` holds `dest-graph-id` (the `id` of the graph it goes to), `src-branch-id` (the
  index in `branch-array` of the branch going out; that branch's `src-node` is the last node inside)
  and `dummy-node` (a copy of the node it reaches in the other section). Skip the links whose
  `dest-graph-id` is another section of your own area, then merge the exits a few meters apart
  into one street. The copied node lies in the next section: a wall put on it stood in the next
  district, out of the player's reach (the Industrial Zone's in the Port); halfway between the
  last node inside and the copied one is the border. A node can stand well above the street just
  inside it: at the Port, 13 m.
- **A force field wall from code.** `spawn-security-wall` (`levels/city/ctywide-obs.gc`, CWI)
  spawns a `security-wall-child` between two points, standing `height` tall from their height,
  child of the process given, and returns its handle. It is the district border of Jak 2: solid to
  Jak (`obstacle blocking-plane camera-blocker`), drawn with ctywide's animated texture
  (`*security-texture-anim-array*`, ctywide's texture callback 6), so it shows wherever ctywide is
  loaded. Its color is one global (`set-security-color!`, `ctywide-texture.gc`): every wall on
  screen takes the last color set, and the child sets red at its init and each frame. Code linked
  before `ctywide-obs.o` in CWI declares it.
- **The ruins' own force field.** The game closes the street from Freedom HQ's district (ctyslumb)
  into the ruins of the old town (ctygenb) with `security-wall-12` until its first task: x 389.75 m,
  z -18.44 to -70.72 m, y 7.94 m, 40 m tall (`decompiler_out/jak3/entities/ctyslumb-actors.json`,
  its `path`). `spawn-security-wall` between those ends puts it back. The Metal Head nest (mhcitya,
  mhcityb, where Jak 2's farms were) opens on the city by the street from the Port's west end, up
  to `mhcity-dark-eco-door-1` (-381 m, 1199 m), and by its airlock.
- **Every door to an interior is an airlock.** The city's doors to interiors and the sewers are
  `com-airlock` subtypes (`cty-door`, `hip-door-a`, `vin-door-ctyinda`, `com-airlock-outer-mhcity`,
  `com-airlock-inner`/`-outer`, `engine/common-obs/airlock.gc`); each opens only while
  `(-> *setting-control* user-current airlock)` holds: `(set-setting! 'airlock #f 0.0 0)` keeps them
  all shut. The game starts a door's process only near it: `process-by-ename` finds none until Jak
  stands by it (`entity-by-name` gives its place).
- **A vertical beam.** `draw-beam` (`engine/target/gun/gun-util.gc`) launches a beam particle along
  a direction, as long as the launcher's `scale-y`, oriented by `forward-up->quaternion` against
  `*y-vector*`: straight up gives no frame. Lean it a hair. It draws only while the segment is in
  the view frustum, and only for the frame: call it every frame.

```lisp
;; a 150 m light pillar at pos, every frame: part 5200 is a copy of the blue gun's beam (part 231)
;; with :scale-y (meters 150) and its own color
(draw-beam (-> *part-id-table* 5200) pos (new 'static 'vector :y 1.0 :z 0.01 :w 1.0) #f)

;; a red force field across a street, 30 m wide, 40 m tall from 10 m below the exit node
(define-extern spawn-security-wall (function process vector vector float symbol handle))
(set! (-> my-walls 0) (spawn-security-wall my-parent left-point right-point (meters 40) #f))
```

Verified: jak3, `levels/city/common/nav-graph-h.gc`, `levels/city/ctywide-obs.gc`,
`levels/city/ctywide-texture.gc`, `engine/level/level-info.gc` (ctywide's callback list),
`engine/target/gun/gun-util.gc`; the battlefront mod's smoke runs: walls placed from the links in
the four districts it fights in (the Port and the Industrial Zone found the same street between
them, as did the Slums and Freedom HQ), a red wall on screen at the Port, white, blue and red beams
on screen, 2026-11-08; the ruins' and the nest's walls holding Jak running at them, six doors
`close` with Jak 3 m off, 2026-11-08.

## 4.43 Haven City traffic: units out of a transport, and back to sleep

- **Out of the hatch.** A traffic unit woken with spawn params `behavior` 6 and the transport's
  handle runs its `exit-transport` state. A guard (`guard-states.gc`) is pinned to the hatch every
  frame (`crimson-guard-method-265`) until `crimson-guard-method-266` finds, 4 or 6 m beyond the
  hatch's end and within 70 degrees of its way out, ground on its own nav mesh whose 2 m sphere no
  other unit blocks; then it walks out and jumps there. When none is found it waits for ever, and
  rides the ship away when the ship leaves. A jetpack (`flying-turret`) hovers until it is within
  4 m of its exit point. Choose the hatch's way by the ground behind it, and turn the ship while a
  guard waits: no test outside method 266 predicts it (a crowd blocks it too).
- **Back to sleep from mod code.** Find the unit in its tracker's `active-object-list`
  (`*traffic-engine*` `tracker-array` 0 or 1) and call `deactivate-object` with its index: it
  takes the unit out of the active list, sends `'traffic-off-force` (`'traffic-off` without the
  flag) and puts it back in its pool. The event alone puts the unit to sleep while the tracker
  still lists it awake: it is counted awake, never woken again, and the lists drift from the
  units (the game crashed in 3 of 6 runs that way).
- **A full nav mesh crashes a first activation.** A unit holds one nav-control on the mesh it last
  walked, asleep too (`change-to` frees the old one when it moves to another). A mesh holds
  `max-nav-control-count`; when none is free, `new-nav-control` prints "too many users", the
  stock `'traffic-activate` handler (`citizen.gc`) calls `go-inactive` and then reads the unit's
  `nav state` through `#f`. Before waking a unit onto a mesh, check
  `(< nav-control-count max-nav-control-count)`, an entry whose `process` is `#f`, or that the unit
  already holds one there.

```lisp
;; put a traffic unit back in its pool the traffic's way
(defun my-unit-sleep ((proc process))
  (let ((h (process->handle proc)))
    (dotimes (ti 2)
      (let ((tr (-> *traffic-engine* tracker-array ti)))
        (dotimes (i (-> tr active-object-count))
          (when (= (-> tr active-object-list i) h)
            (deactivate-object tr i #t)
            (return #t))))))
  #f)
```

Verified: jak3, `levels/city/common/guard-states.gc` (`exit-transport`),
`levels/city/traffic/citizen/guard.gc` (methods 265, 266), `levels/city/protect/flying-turret.gc`,
`levels/city/traffic/traffic-engine.gc` (`deactivate-object`, `activate-from-params`),
`levels/city/traffic/citizen/citizen.gc` (`'traffic-activate`), `engine/nav/nav-mesh.gc`
(`new-nav-control`, `change-to`); the battlefront mod's smoke runs: 10 of 12 units stuck in a
ship at Freedom HQ, then 16 stuck units a run released and no crash in 3 runs with
`deactivate-object`, against 3 crashes in 6 with the event alone, 2026-11-08.

## 4.44 Haven City traffic: routing a unit, the default nav mesh, clocks

- **A traffic unit's lane.** A citizen walks its lane to the lane's end node: its controller's
  `dest-point` (`turn-exit-point`, `levels/city/traffic/vehicle/vehicle-control.gc`), which the
  `'active` post gives its nav control as target each frame; within 4 m it asks the controller for
  the next lane through `choose-branch-callback`, a `(function vehicle-controller vehicle
  nav-branch)` (the stock one, `vehicle-controller-method-15`, a random lane `dest-node-id-at-max?`
  allows), and at a dead end `citizen-method-208` puts it on any lane 20 to 120 m behind it.
  `citizen-method-199` (a unit put on a segment) starts it over at the segment's first vertex.
- **The lanes cannot route a unit across the city.** The walking lanes are loops within a section:
  ctyslumc has 138 lane nodes and no link to another section's graph, and 173 of ctyslumb's 177
  nodes have one lane out or none. Neither a callback searching the lanes breadth first (256
  nodes) nor a distance field over every loaded lane (83 of 315 nodes reached the goal) took units
  from the Freedom League's base to a point 240 m away: they circled 136 to 290 m from it.
- **Marching a unit over the nav meshes.** The meshes join the sections: a nav-enemy standing on a
  mesh's link poly (`pat` 4, its `link` < 255) moves onto `(-> mesh link-array link dest-mesh)`
  (`nav-enemy-method-188`, from method 187 in its post). Twice a second, for a unit in `'active`,
  find the way through the fewest meshes to the goal's mesh (`find-nearest-nav-mesh`): breadth first
  from `(-> unit nav state mesh)` over each mesh's `link-array` (`dest-mesh` #f while the next
  section is not loaded); every frame, set the unit's `dest-point` to a spot 2 m past the link poly
  that starts the way (`poly-centroid` of that poly, toward the next mesh's `dest-link-poly-id`
  poly), or to the goal on its mesh. Drop it from the march once its mesh changed. Aimed at the
  poly's centroid, units stopped 4 m short (the lane-end distance), went back to their lanes, and
  piled up there until the game printed `touching-prims-entry-pool::alloc-node() failed!` thousands
  of times. To make it run, set `(-> unit nav target-speed)` from `speed-run` each frame (the travel
  clip blends walk and run on the speed); a large `wait-time` keeps it from its idle stops. A
  marching unit's lane callback can answer the stock random lane: the post asks for a lane each
  frame near its `dest-point`.
- **The city's nav meshes do not all join.** With the six street sections loaded (4.15) there are
  nine nav meshes, one or a few a section: ctysluma links ctyindb, ctyindb ctyinda, ctyinda the
  Port's east mesh, the Port's three linked meshes one to the nest; ctyslumb links ctyslumc and an
  unloaded mesh (the ruins'). The south (Freedom HQ, ctyslumb and ctyslumc) joins the north only
  through the unloaded ruins: no march leaves it. Its streets meet ctysluma's near (747, -70) with
  no link, the nearest corners of their polys 2.1 m apart (poly vertices are relative to the mesh's
  `bounds`). A mod can bridge such a seam: when a unit stands within 2.5 m of a point 1.5 m inside
  one mesh, `(change-to other-mesh unit)` and `move-to-point!` 1.5 m inside the other. A search
  breadth first from each goal point's mesh over every listed mesh (a field of the first link
  toward the point, kept per mesh) costs less than one per unit, and needs no bound.
- **80 units run out of touching records.** `*touching-prims-entry-pool*` holds 64
  (`engine/collide/collide-touch-h.gc`): units crowding at their ships and points printed
  `alloc-node() failed!` thousands of times (the touches dropped). 256 records, the pool's `nodes`
  array and its `init-list!` count, gave none.
- **The default nav mesh fills.** A new traffic unit takes its first nav-control on the engine's
  `*default-nav-mesh*` (`citizen.gc`, its enemy info's `nav-mesh`; `engine/nav/nav-mesh.gc`, 128
  of them) and keeps it until it is first woken onto a street's mesh (`change-to` frees the old
  one, a dying process too). When none is free, `new-nav-control` prints "too many users for
  nav-mesh #f" and the new unit's `setup-enemy!` (nav-enemy) reads its missing nav-control: the
  game crashes. A reset that makes the traffic spawn its pools again while the old units still hold
  theirs gets there. Before such a reset, `(remove-process-drawable *default-nav-mesh* unit)` on the
  pools' units; for more room, give the mesh a larger `nav-control-array` and user-list engine at
  load, 255 at most (`max-nav-control-count` is a uint8). Its `nav-control-count` is a high-water
  mark: freed entries (`process` #f) below it are reused. Waking a unit with `activate-by-type` on a
  street whose nav mesh is not loaded fills it too: the traffic gives the unit `(get-nav-mesh
  nav-mesh-id)` of the segment, #f there, so the unit never leaves the default mesh ("failed to
  locate ground for citizen"). Check `(get-nav-mesh (the-as actor-id (-> seg nav-mesh-id)))` before
  waking a unit on a segment (a battle that took streets 540 m off reached 96 taken, then crashed).
  A pool larger than 20 fills it the same way (4.30).
- **Spider robots and robotguards on the march.** A spider robot the faction manager has at war
  walks its lanes in `'hunting`, the child of its `'active` (`spydroid.gc`: `'active` goes there
  with a nonzero `faction-mode`), following its `dest-point` the same way: a march that takes
  `'active` alone never moves one. A robotguard with a foe it cannot reach stands in `'hostile` and
  `'hostile-stand` for good (400 to 700 m from its goal, seen for minutes). Each attacker's
  `callback` (`city-attacker-info`, a `(function process city-attacker-info int)`) filters the
  foes `mission-squad-control-method-13` scores: #f drops one, any other value keeps it; a callback
  keeping foes within 50 m leaves a robotguard without one, and
  `(go-process unit (method-of-object unit active))` sends it back to its lanes. With no foe left,
  method 13 sets `enemy-index` -1 and `enemy` #f, and the controller's update asks again every
  frame. Doing that to a unit in the middle of an attack (`enemy-flag lock-focus`, set by the
  guards' grenade and rifle states) crashed the game twice in three runs (an access violation, no
  report); sparing those units, none in two.
- **A foe behind a wall.** Haven City's controller picks foes by distance alone, through walls. A
  probe of the background between the two (`fill-and-probe-using-line-sphere`, `collide-with`
  `backgnd`, radius 0.2 m, from 1.6 m above the unit's `root trans` to 1.2 m above the foe's)
  returns a fraction of 0 or more when blocked: in a battle it hit 68 of 391 unit-to-foe lines, 56
  on walls (the hit triangle's normal level) and 12 on floors (jetpacks above a ledge). A crimson
  guard already tests its line of fire before it shoots (`crimson-guard-method-258`, the same
  probe at 0.3 m, blocked when the hit triangle faces away from the target).
- **Waking a unit anywhere on a nav mesh.** Send the traffic manager `'activate-object` with spawn
  params whose `behavior` is 2 (`'active`), `nav-mesh` the mesh, `position` the `poly-centroid` of
  one of its polys (world coordinates), `nav-branch` #f and `proc` #f: the woken process comes back
  in `proc`. Skip the link polys (`pat` 4). Some polys put a unit where the game prints "failed to
  locate ground for citizen" (platform edges); the unit still comes and fights. A poly's middle lies
  0.5 to 1.4 m under the street's collision: a probe down that stops less than 2 m over it meets
  the street.
- **Bringing a unit down from the sky.** An event sent to a unit in the frame it is woken reaches
  its sleeping state: a guard sent the stock `'jump` at once stayed in the air, walking. Wait for
  its `'active` (`(-> unit next-state name)`), then raise its `root trans` and send
  `(send-event unit 'jump 0 ground-point)`: the guard's jump (wind-up, in air, its `notice` clip as
  it lands) carries it down, its trajectory a parabola 0.5 m over the higher end. Traffic behavior 6
  (exit-transport) needs no ship for a robotguard or a spider (their `go-from-behavior` only copies
  `position`): the robotguard falls as a ball and lands in `'roll-chase`, the spider tumbles
  (`knocked0`) and lands in `'hostile`. Never send a spider `'jump`: its jump clips are -1 in its
  enemy info, an art index the jump would read. A jetpack guard's flight keeps it 3 to 18 m over the
  ground, seeking its height at 0.7 m/s: raised higher, it drops to 18 m at once, then sinks.
  `enemy-falling-post` collides with whatever its root prim's `collide-with` names, and exit-transport
  ends on the first surface touched: spiders falling from 30 m stopped on the jetpacks hovering over
  the spot (2 in 8 stayed 12 m up), and, with `(collide-spec backgnd)` alone, on the scenery under a
  ship's hatch 13 and 22 m up (6 and 14 m over the street): landed there, a spider walks its lanes
  at that height, in the air. Give the root prim's `collide-with` 0 (nothing) while the unit is more
  than 3 m over the nav poly it falls onto, then `(collide-spec backgnd)` to land on the street, and
  the old mask once out of exit-transport: 16 of 16 fell at once, none in the air. The
  city's floors have no underside: a probe going up passes through them (48 of 48 probes went up
  30 m clear, 7 of them under a floor a probe down met), so test what is over a spot with a probe
  down. A unit killed goes back to sleep in the traffic's pool (`'inactive`): its handle still
  names a process.
- **Nav poly types; the Industrial Zone's walkways.** A nav poly's `pat` (`engine/nav/nav-mesh.gc`
  `debug-draw`): 1 a gap (black; walkers cross it only through a jump,
  `plan-over-pat1-polys-using-route`), 2 avoided (grey), 4 a link to another mesh, 8 another kind
  (magenta); the travel rays of `engine/nav/nav-control.gc` ignore 1 and 2 (`ignore` 3).
  `adj-poly` holds each edge's neighbour, 255 for none. The nine street meshes of Haven City have no
  poly of types 1, 2 or 8, and each is one island: every poly can be walked to. The Industrial
  Zone's two meshes (ctyinda, ctyindb) carry its walkways, 30 and 23 polys at 27 m, over its 8 m
  streets, joined by stair polys at 12, 17.5 and 21.8 m; a unit's shortest way keeps to the
  streets, so a unit goes up only when sent to a walkway poly as a waypoint.
- **The city's war music.** `(set-setting! 'music 'portattk 0.0 0)` plays the music of the game's
  assaults on the city (`levels/city/protect/assault-task.gc`, `levels/city/port/attack/ctyport-attack.gc`)
  in every district, in place of each section's own; `remove-setting!` gives them back.
- **A robotguard rolling on its walks.** `citizen-travel-anim` plays the run clip from
  `get-run-anim` (a citizen's `anim-run`) once the nav speed passes 6 m/s: set `anim-run` to the
  ball clip (13), `dist-run-anim` to 43690 (roll-chase's own pace: speed / 40960), `interp` 1.0 and
  the `run-mode` flag for it to show at once; `speed-run` sets how fast it rolls. The stock run clip
  is 11.
- **A unit 1.5 times as fast, its legs at the stock pace.** `citizen-travel-anim` plays the walk
  and run clips at the nav state's speed over `dist-walk-anim` and `dist-run-anim`: raising
  `speed-walk`, `speed-run` and both clip grounds by the same factor keeps the stock cadence (the
  guards at 18 m/s ran their legs at the player-guard pace, 0.38 frames a frame). Raising
  `speed-run` alone played the clips that much faster; the nav control's `speed-scale` left the
  spiders at their stock 8 m/s and the guards at 16 instead of 18. A unit's init sets its stock
  values each time it is woken: set them again on a timer.
- **Each process has its own clock.** `current-time` reads the calling process's clock: the START
  menu's code runs on the menu's, which is not the game's. A time taken on one and tested on the
  other never passes; for a time several processes compare, read `(-> *display* game-clock
  frame-counter)` explicitly.
- **Sending Jak into a state from another process:**
  `(send-event *target* 'change-state target-death 'my-mode)` (`target-handler.gc`), his state's
  code getting the mode as its argument.

Round 26 of the battlefront mod, 2026-11-12: `engine/nav/nav-mesh.gc` (`debug-draw`, `nav-poly`), `engine/nav/nav-control.gc` (the rays' `ignore`), `levels/common/enemy/spydroid.gc` (exit-transport), `engine/ai/enemy.gc` (`enemy-falling-post`); the meshes dumped in game (islands, types, heights), 16 spiders and robotguards out of a ship watched as they fell, guards on the walkways (8 in 2 minutes once sent there), the music read back from `*setting-control*`.
Verified: jak3, `levels/city/traffic/vehicle/vehicle-control.gc` (methods 14, 15),
`levels/city/traffic/citizen/citizen.gc` (methods 199, init, active state),
`engine/nav/nav-mesh.gc` (`new-nav-control`, `change-to`), `engine/nav/nav-enemy.gc`
(`setup-enemy!`), `engine/process-drawable/process-drawable.gc` (`deactivate`),
`engine/target/target-handler.gc`, `levels/common/enemy/spydroid.gc`, `levels/city/protect/roboguard-city.gc`,
`levels/wascity/mission-squad-control.gc` (method 13); the battlefront mod's smoke runs: the crash named by a temporary
vectored exception handler (`nav-enemy method 119`), the default mesh at 41, 85, 115 of 128 then
the crash after a far reset, 17 of 255 with both fixes; defenders brought to their point by the
march over the meshes (from the Freedom League's base to Freedom HQ's point: 14 guards on it 60 s
in, none in 180 s by the lanes); a START menu entry tested on the game clock, 2026-11-08; 12 of 12
spider robots and robotguards 10 m closer to their goal in 25 s, guards at 19.7 m/s, spiders at
12.4, jetpacks at 38.8 (stock 12, 8, 25), 2026-11-09. Round 24 of the mod, 2026-11-11: `levels/city/common/guard-grenade.gc`, `levels/city/traffic/citizen/guard.gc` (`crimson-guard-method-258`); a battle's probes per unit and foe (where they hit, the normal's height), units with a foe behind a wall 1 in 5 then 1 in 14 with the blind look, the two crashes with every unit blinded and none in two runs sparing `lock-focus`; 100 units woken on random polys around a control point in 3 minutes. Round 25, 2026-11-12: `engine/ai/enemy.gc` (the `'jump` event, `setup-jump-trajectory!`, `enemy-falling-post`), `engine/physics/trajectory.gc`, `levels/city/traffic/citizen/guard.gc` (`jump-anim-handler`), `levels/common/enemy/spydroid.gc`, `levels/city/protect/roboguard-city.gc` (`go-from-behavior`, exit-transport, roll-chase), `levels/city/protect/flying-turret.gc` (its height), `levels/city/traffic/citizen/citizen.gc` (`citizen-travel-anim`); the battlefront mod's runs: 21 of 21 guards in their jump and landed, robotguards and spiders falling, 1005 of 1005 far-marching robotguards on the ball clip, the sky probes over 48 polys.

## 4.45 Filled 2D triangles over the HUD

`draw-sprite2d-xy` (`engine/gfx/hw/display.gc`) draws rectangles only. A triangle is the same
packet with a triangle primitive: a DMA `cnt` tag with a VIF `direct`, a GIF tag in `reg-list`
mode, one loop of six registers (`prim`, `rgbaq`, three `xyzf2`, `rgbaq` again: a register list
must fill whole quadwords), `(new 'static 'gs-prim :prim (gs-prim-type tri) :abe #x1)`, each
vertex at `x` = 16 (1792 + x) and `y` = 16 (1840 + y) in the 512-wide screen units the sprites use,
`z` `#x3fffff`; then the quadword count after the tag into the DMA tag and the VIF `direct`, as the
sprite function patches it. In the `debug-no-zbuf2` bucket the PC renderer draws it with alpha.
Discs are fans of eight; scale x radii by `(-> *video-params* relative-x-scale)` to keep them round
on a wide screen. The battlefront mod draws its maps' units as the minimap marks a guard with them
(a disc and a 60 degree view cone), 80 units and the points every frame at 60 frames a second.
Verified: jak3, `engine/gfx/hw/display.gc`; the battlefront mod's round 20 runs, 2026-11-09.

## 4.46 Another player's gun sounds: one mode sound bank

A gun mod's sounds (`red-shot-fire`, `yellow-shot-fir`...) live in a "mode" sound bank, one per
mod (`modegur1`... `modegud3`), like the jet-board's (`modebord`), the lizard's (`modeflut`), Dark
and Light Jak's (`modedark`, `modelit`), the mech's and Daxter's mode (`modeidax`).
`target-gun-type-set!` (`engine/target/target-gun.gc`) sets the `mode-sound-bank` setting to the
bank of the mod in hand, and the Jak 3 overlord has one mode slot (`gBanks[1]`,
`game/overlord/jak3/sbank.cpp`): a sound of another mod played here (a remote copy of another
player's shot) finds no bank and is silent; a playable Daxter holds `modeidax`, so shots he fires
through the gun functions are silent on his own machine too. The PC sound player keeps each bank's
data apart (SPU addresses are ignored), so a mod can add mode slots in `AllocateBankName`, skip
the unloads of mode banks in `srpc.cpp` (`UNLOAD_BANK`), and load the 18 banks once with
`sound-bank-load` (mode 3, priority 2, while `(rpc-busy? 1)` is 0). The overlord looks a sound up
by its name in capitals with `-` as `_` (`strcpy_toupper`, `soundcommon.cpp`): a C++ check of a
GOAL sound name must convert it the same way. `red-shot-hit` is in no bank. The wasteland
vehicles do the same through the levels: each has a bank (`wcar.gc` `bank-replace`, the toad's
`'((wasall1 wastoad))`) that `*vehicle-manager*` `'extra-bank` sets for the driver, so a vehicle
another player drives is silent unless its bank is loaded as well; the levels follow their own
slots, so a bank a mod loaded elsewhere must be freed when the levels want it (the multiplayer mod's
`vehicle-sounds`: 696 of 777 toad sounds in no bank on the other machine, then none but the two no
bank holds).
Verified: jak3, `engine/target/target-gun.gc`, `game/overlord/jak3/sbank.cpp`, `srpc.cpp`; the
multiplayer mod's `daxter-gun-hold` run (30 of 30 shot sounds in no bank on the other machine,
then 22 of 32 found, the rest `red-shot-hit`), 2026-11-10.

## 4.47 A focusable that never probes the ground: nav enemies ignore it

`process-focusable` `get-trans` mode 1 returns `gspot-pos`, the ground probe's point
(`engine/process-drawable/process-focusable.gc`), and a nav enemy turns hostile only to a focus
whose mode-1 point lies within its notice heights (`can-become-hostile-to?`,
`engine/nav/nav-enemy.gc`); its chases aim there too. A focusable moved by setting its root (a
network puppet, a scripted body) never probes the ground: its `gspot-pos` keeps the constructor's
value, about -10000 m (`collide-shape-h.gc`), and enemies notice it (`aware2`) but never attack it.
Override `get-trans` to return the root's position for mode 1. Separately, an enemy keeps its focus
while that focus lives unless it has `multi-focus` (`update-focus`, `engine/ai/enemy.gc`); calling
`find-best-focus` every frame makes it take the most aware, then the nearest, candidate.
Verified: jak3, `engine/process-drawable/process-focusable.gc`, `engine/nav/nav-enemy.gc`,
`engine/ai/enemy.gc`; the multiplayer mod's puppet, its ground point logged at its height after
the override, 2026-11-10.

## 4.48 A death that pins the camera: tf6, tf12, tf14

Dying in lava, melting or burning (`target-death`, `engine/target/target-death.gc`) sets
`target-flags tf6` and copies the camera position to `alt-cam-pos`; drowning sets `tf14`.
`target-cam-pos` (`engine/target/target-util.gc`) then returns `alt-cam-pos` (`tf6`, `tf12`), so
the camera stays where the player died. The stock reset spawns a new player, without the flags; a
mod that brings the same player back (`go target-stance` from the end of its death) must clear
them, `(logclear! (-> self target-flags) (target-flags tf6 tf12 tf14))`, and send `*camera*`
`'teleport`, or the camera keeps looking at the place of death.
Verified: jak3, `engine/target/target-death.gc`, `target-util.gc`; the multiplayer mod's co-op
respawn (`coop-respawn-lava`: pinned during the death, following the player after), 2026-11-10.
