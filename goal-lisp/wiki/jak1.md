# Lisp wiki — Jak 1 specifics

What differs in Jak 1. Read [common.md](common.md) first: only real differences live here. Index: [index.md](index.md).

## 2.1 Vocabulary delta

Jak 1's `target` (the player process) derives directly from
`process-drawable` — unlike Jak 2/3/Jak X, which derive it from
`process-focusable` instead.

## 2.2 Memory

Kernel entry points: `goal_src/jak1/kernel/gcommon.gc` and
`goal_src/jak1/engine/level/level.gc`. On `master-dev`, Jak 1 ships the
same 512 MB PC memory expansion as Jak 2/3:

```lisp
;; goal_src/jak1/kernel/gcommon.gc
(defconstant END_OF_MEMORY #x20000000)
```

Jak 1 has a different level-heap architecture than Jak 2/3: there is no
page/multiplier scheme (`DEBUG_LEVEL_HEAP_MULT`) — its level heap size
comes from `LEVEL_HEAP_SIZE_DEBUG`, allocated straight from the global
heap.

Jak 1 could not originally boot with `-debug` at all: `InitMachine`
derived `debug_heap_end` from a 128 MB address mask, so once
`DEBUG_HEAP_START` moved for the expanded PC memory layout, the
subtraction underflowed to a multi-gigabyte heap size and segfaulted.
Jak 2/3 already used an explicit size; Jak 1 now does too — if you ever
touch debug-heap sizing C++ code, keep it an explicit constant, not a
subtraction between two addresses.

## 2.3 Compile / validate loop

```bash
task set-game-jak1        # pick Jak 1 once
task repl                 # then (mi) — incremental compile + hot reload
task boot-game             # boot and check the log
```

## 2.4 In-game Mods toggle: the same registry, a smaller popup

The unified `mods-menu.gc` registry ([1.2.11](common.md#1211-register-an-in-game-mods-toggle))
is live in Jak 1 too, with the same `mods-menu-register` call and the same L3 + SELECT bind
(`pc/features/mods-menu.gc`, ticked from `main.gc` through `*mods-menu-hook*`, in a retail boot).
Its popup (`pc/util/popup-menu.gc`) is a smaller port of Jak 2's: a plain object, ASCII labels in
the debug menu's font, `[x]` / `[ ]` flag rows, no `popup-menu-dynamic-submenu`. Never add a
mod's toggle to the debug root menu instead: appending to it at link time segfaults the boot
([2.5](#25-the-debug-root-menu-is-fragile-at-link-time)), and a debug-only toggle is out of a
launcher player's reach.

Verified: jak1, `pc/features/mods-menu.gc`, `pc/util/popup-menu.gc`, `engine/game/main.gc`,
`dgos/game.gd`, 2026-11-04.

## 2.5 The debug root menu is fragile at link time

Appending an item to Jak 1's debug ROOT menu during link-and-exec
segfaults the boot, reproducibly — `debug-menu-append-item` on the root
walks every existing item via `debug-menu-item-get-max-width`. Allocating
a fresh `debug-menu` elsewhere is fine; it is specifically the root
append that crashes. Jak 2/3 have identical menu bodies and are
unaffected. This is why a Jak 1 mods-menu port needs a deferred install
rather than the naive append.

```lisp
;; unsafe at link time in jak1:
(debug-menu-append-item (-> *debug-menu-context* root-menu) item)
```

