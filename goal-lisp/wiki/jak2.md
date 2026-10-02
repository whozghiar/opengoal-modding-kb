# Lisp wiki — Jak 2 specifics

What differs in Jak 2. Read [common.md](common.md) first: only real differences live here. Index: [index.md](index.md).

## 3.1 Vehicles: flags, grab rails, driver methods

All ambient and player vehicles derive from `vehicle`
(`goal_src/jak2/levels/city/traffic/vehicle/vehicle.gc`). The
`rigid-body-vehicle-constants` `:flags` bitfield configures gameplay:

| Bit | Hex | Effect |
|---|---|---|
| 2 | `#x04` | `guard-vehicle` — Crimson Guard asset (Hellcat, Guard Bike). |
| 3 | `#x08` | `vehicle` — standard vehicle physics. |
| 5 | `#x20` | `allow-gun` — Jak can draw & fire guns while driving. |
| 6 | `#x40` | `allow-flight-zones` — R2 switches low/high altitude corridors. |

`#x6c` = flight + guns on a guard vehicle.

```lisp
;; unarmed guard-derived vehicle — override method-94 or it SIGSEGVs on the
;; missing turret when the player takes control
(defmethod vehicle-method-94 ((this paddywagon))
  ((method-of-type vehicle vehicle-method-94) this)
  0
  (none))
```

Grab rails: define `:grab-rail-count` + `:grab-rail-array` for
long-range edge-grab boarding (Triangle -> hang -> Cross -> cockpit).
Small bikes use `:grab-rail-array #f` and seat Jak instantly. There is
also a flight control-point / `cm-offset-joint` "turtle-flip" gotcha
documented in the git history of `jak2/features/transport_traffic`.

## 3.2 Traffic manager

`*traffic-manager*` owns ambient city actors. Change large dynamic state
by event, not by hand-spawning: `(send-event *traffic-manager*
'deactivate-by-type (traffic-type ...))` recycles existing actor slots.
When spawning ejected riders, always check
`(when (-> spawn-params nav-mesh) ...)` first — a missing nav-mesh causes
an infinite spawn-retry loop and memory exhaustion.

The tracker-array naming in the decompiled source is swapped:
`citizen-tracker-array` holds vehicles, `vehicle-tracker-array` holds
pedestrians. `activate-one-vehicle` is actually the pedestrian picker —
confirmed by the `tracker-index` assignment in
`reset-and-init-from-manager` (vehicle types get `tracker-index` 1,
pedestrian types get 0).

```lisp
;; traffic-type 21 is an "invalid/none" sentinel (ctywide-obs.gc,
;; get-random-parking-spot-type), not a spawnable slot — new custom traffic
;; types must start at 22; the per-type arrays are sized 0..20
(if (!= a1-5 (traffic-type traffic-type-21)) ...)
```

Spawn ratios are governed by `want-count` (-> `target-count`, the live cap
`activate-by-type` enforces), not by the random type draw:

```lisp
(set! (-> info target-count) (max 0 (+ want -1)))
```

## 3.3 Virtual method / state residency in practice

Jak 2 hits the [general vtable trap](common.md#147-virtual-methodstate-residency-the-vtable-trap)
most with traffic/vehicle actors: define **all** their `:virtual #t`
states and methods in a resident file (`vehicle.gc`, `car.gc`), never in
a mission DGO. Symptom in the log:
`sending traffic-on to #<... :state process-drawable-art-error>` or an
actor stuck `inactive`/invisible in free-roam.

## 3.4 Live re-skinning a process

Calling `initialize-skeleton(-by-name)` again on a live process swaps its
mesh and animation set while keeping its identity. Proven safe in the
base game on reconfigurable objects (`widow-extras.gc`,
`metalkor-setup.gc`). The risk is `target` only: `target` sets up
`joint-mod`s (neck look-at, gun aim, IK) against fixed joint indices of
`skel-jchar`. Re-skinning `target` to a skeleton with a different joint
layout makes those `joint-mod`s index the wrong joint or go out of
bounds. A stub process with no `joint-mod`s has nothing to desync.

```lisp
;; from the REPL — check whether the joint count actually changed before/after
(-> *target* node-list length)
(initialize-skeleton-by-name *target* "skel-crimson-guard-level")
(-> *target* node-list length)
```

## 3.5 Merc geometry & FR3 residency

Imported models built with `build-actor` produce a merc art-group whose
geometry (`.fr3`) must be resident wherever the actor is drawn. If an
actor is spawned globally but its `.fr3` is level-scoped, it renders as
nothing or crashes on relocate. Keep custom-model geometry in a resident
art-group, or bind the process's level to the level that owns the
geometry (see [1.2.5](common.md#125-bind-a-model-to-a-process-initialize-skeleton)).

To bake a model's geometry into a level that doesn't natively own it (a
"no-borrow" injection), declare the pairing in the decompiler config:

```jsonc
// decompiler/config/jak2/jak2_config.jsonc
"extra_art_groups_by_dgo": {
  "LWIDEA.DGO": ["transport-ag:LPROTECT.DGO"]
}
```

`"<art-group-name>:<HOME.DGO>"` — the `:<HOME.DGO>` part tells the
extractor which level's texture remap table to resolve textures against;
omitting it lets the decompiler pick the first level containing the art
group, which may lack the required texture page. Rebuild and re-extract:

```bash
task build-release-decomp
task extract
```

Then add the art-group `.go` (and its `tpage-*.go`) to the target level's
`.gd`, **before** the level's own `.go` (the BSP entry must stay last):

```lisp
  "tpage-2869.go"
  "transport-ag.go"
  "lwidea.go"
 ))
```

A dynamically-spawned actor has no pre-placed BSP entity, so
`skeleton-group->draw-control` cannot resolve one on its own. Bind the
process to a resident entity before `initialize-skeleton` — in Haven
City, `(ctywide-entity-hack)` does this; elsewhere, bind explicitly:

```lisp
;; Haven City
(defbehavior custom-actor-init-by-other custom-actor ((arg0 custom-actor-params))
  (ctywide-entity-hack)
  (initialize-skeleton self <skeleton-group> (the-as pair 0)))

;; a level without ctywide (e.g. the Strip Mine)
(with-pp
  (let ((lvl (level-get *level* 'strip)))
    (when (and lvl (> (-> lvl entity length) 0))
      (process-entity-set! self (-> lvl entity data 0 entity))
      (set! (-> self level) lvl)
      (set! (-> pp level) lvl)))
  (initialize-skeleton self <skeleton-group> (the-as pair 0)))
```

Troubleshooting:

| Symptom | Cause | Fix |
|---|---|---|
| Model is invisible, but the process runs and sounds play | Geometry not in the resident `.fr3` | Add to `extra_art_groups_by_dgo`, `task extract` |
| Crash with `process-drawable-art-error` | Art-group `.go` not in the active DGO, or entity binding missing | Add to `.gd`, call `ctywide-entity-hack` / `process-entity-set!` before `initialize-skeleton` |
| Model is white/shiny (untextured) | Wrong texture remap table | Append `:<HOME.DGO>` to the `extra_art_groups_by_dgo` entry |
| Visible in one Haven City zone, not another | Only baked into one of `LWIDEA`/`LWIDEB`/`LWIDEC` | Bake into all three DGOs the actor should exist in |

Target DGO selection for Haven City specifically:

| Target DGO | `.fr3` | Residency scope | Best used for |
|---|---|---|---|
| `GAME.CGO` | `GAME.fr3` | Every level globally | Global weapons, player skins, universal UI effects |
| `CWI.DGO` | `ctywide.fr3` | Permanent anywhere in Haven City | Core city systems, global city vehicles, persistent actors |
| `LWIDEA.DGO` / `LWIDEB.DGO` / `LWIDEC.DGO` | `lwidea.fr3` / `lwideb.fr3` / `lwidec.fr3` | Zone-swapped via traffic manager (`ctywide` slot 1) | Ambient traffic vehicles, zone-specific pedestrian actors |

## 3.6 Traffic engine & Haven City population

Haven City's whole ambient population is decided by which level is
borrowed into `ctywide` slot 1: `lwidea` = peace, `lwideb` = Metal Head
invasion, `lwidec` = roboguards. `lwide-activate` reacts to the swap. For
a permanent override, write `user-default` (not `set-setting!`, which
binds the setting to the calling process — a debug-menu pick-func dies
and takes it with it):

```lisp
(set! (-> *setting-control* user-default borrow) '((ctywide 1 lwideb special)))
(apply-settings *setting-control*)
```

`ctywide` borrow slot 0 is unused for the whole game except "destroy the
blast bots" (`(ctywide 0 lbombbot display)`) — free real estate for a mod,
with a memory budget retail already proved:

```lisp
'((ctywide 0 lbombbot display) (ctywide 1 lwideb special))
```

Swapping `lwideb` in also **clears** the `target-jak` alert flag, which
drops `jak` from every guard's focus collide-spec in
`crimson-guard::citizen-init!` — "guards ignore Jak" during an invasion
is retail behavior, not something a mod has to write:

```lisp
(logclear! (-> gp-0 alert-state flags) (traffic-alert-flag target-jak))
```

`reset-actors` (death / checkpoint restart / `(mi)`) calls every active
level's `activate-func` in `*level*` slot order, and nothing orders
`ctywide` before `lwidea`. When `lwidea` gets the lower slot,
`ctywide-activate` runs *after* `lwide-activate` and wipes the population
it just installed — Haven City respawns empty and guards stop reacting.
Fix by re-running `lwide-activate` at the end of your `init-params`:

```lisp
(dotimes (i (-> *level* length))
  (let ((lev (-> *level* level i)))
    (when (and (= (-> lev status) 'active) (= (-> lev name) 'lwidea))
      (lwide-activate lev 'life))))
```

`reset-and-init-from-manager` is the single wipe point for both halves of
the city's traffic config: it clears every
`object-type-info-array[0..19].level` (slot 20 alone gets `'ctywide`
back, so `spawn-all` skips every other type) *and* resets `alert-state`
(clearing `target-jak` and any forced war-zone alert). Restoring only the
level bindings fixes the empty city and leaves the guards deaf — reset
both:

```lisp
(set! (-> a1-3 level) #f)                              ;; the wipe (native)
(set! (-> this object-type-info-array 20 level) 'ctywide)
(reset (-> this alert-state))                          ;; the wipe (native)
```

## 3.7 Process heap & art-group lifetime diagnostics

A process printed by the REPL as `:heap N/N` (used == total) has already
run through `dead-pool-heap::shrink-heap`, called as soon as `init` ends
on a `go`. A `process-drawable` with only ~1.3 KB of heap use never
allocated its skeleton/draw-control — it went to
`process-drawable-art-error` even though nothing was printed (that state
only draws debug text). A healthy actor is several KB.

`skeleton-group->draw-control` resolves the art-group in `(-> pp level)`,
so a runtime-spawned actor must live in the level that ships its `-ag.go`
*and* its texture page. `lwidea`/`lwideb`/`lwidec` borrow `ctywide`'s
heap, so `ctywide` always outlives them — publishing a `ctywide`
art-group into an lwide level's directory with `set-loaded-art` is
lifetime-safe; the reverse dangles:

```lisp
(set-loaded-art (-> lwide-level art-group) (art-group from ctywide))
```

The `art-group-load-check` disk-loading fallback only runs inside
`(when *debug-segment* ...)` — a runtime-spawned actor whose art-group
lives in a different level than the spawning process works in a `-debug`
boot and silently fails in retail: `initialize-skeleton` goes to
`process-drawable-art-error`, which only draws debug text, so nothing is
printed either way. Ship every runtime-spawned actor's art-group already
present in its spawn level's `.gd`; don't rely on the disk fallback.

City traffic spawns (`vehicle-spawn` / `citizen-spawn`) and a bare
`(process-spawn ...)` both draw from `*default-dead-pool*` with a
`#x4000` stack. A menu action that sends `'kill-all` then `'spawn-all` in
the same frozen (`'menu` master mode) frame churns roughly 150 of those
slots and can starve anything else trying to spawn that frame —
`process-spawn` returns `#f` silently on exhaustion. Never spawn without
checking the result:

```lisp
(get-process *default-dead-pool* arg1 #x4000)
```

## 3.8 Memory

Kernel entry: `goal_src/jak2/kernel/gcommon.gc`,
`goal_src/jak2/engine/level/level.gc`.

```lisp
;; goal_src/jak2/kernel/gcommon.gc
;; raise the ceiling used by valid? to 512 MB
(defconstant END_OF_MEMORY #x20000000)

;; goal_src/jak2/engine/level/level.gc
;; level-heap size = multiplier x base. 15.0 (Jak 3's value) over-allocates
;; on Jak 2 (it loads ~35 MB resident first) and panics at boot. 12.0 is
;; the tested safe value: ~215 MB level heap, ~60 MB spare — a 10x increase
;; over the PS2.
(defconstant DEBUG_LEVEL_HEAP_MULT 12.0)
```

## 3.9 Additional verified facts

`enemy && !guard` on `process-mask` isolates city Metal Heads exactly:
`citizen-enemy::citizen-init!` sets `enemy`, `crimson-guard::citizen-init!`
clears it and sets `guard`, plain citizens have neither. No type check
needed:

```lisp
(and (logtest? (-> p mask) (process-mask enemy))
     (not (logtest? (-> p mask) (process-mask guard))))
```

`hover-enemy` needs a per-level `*nav-network*`, which some levels lack —
every network call in its methods sits behind the `honflags-0` bit
(retail's "following a scripted path" flag). Setting that flag
permanently gives free flight with no network, at the cost of obstacle
avoidance:

```lisp
(logior! (-> this hover flags) (hover-nav-flags honflags-0))
```

Resizing a decompiled runtime-only struct (e.g. adding a field) breaks
any sibling field declared with a hard-coded `:offset`. Re-express it as
an overlay on another field so the compiler recomputes the offset
instead:

```lisp
(vehicle-tracker-array traffic-tracker :inline :overlay-at (-> tracker-array 1))
```

