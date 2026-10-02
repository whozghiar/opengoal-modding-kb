---
name: goal-lisp
description: Conceptual guide to OpenGOAL LISP syntax, typing system, processes, behaviors, virtual states, macros, and common language traps. Points to the per-game Lisp wiki for worked code examples.
---

# GOAL Lisp & Syntax — Conceptual Reference

OpenGOAL is an x86-64 native port and extension of GOAL (Game Oriented
Assembly Lisp), the compiled LISP dialect Naughty Dog wrote for the Jak &
Daxter series. This skill explains the mental model behind GOAL's syntax,
type system, process model, and state machine semantics.

It intentionally carries no code. Every GOAL/Lisp code example in this
project lives in the Lisp wiki:
[`goal-lisp/wiki/`](wiki/index.md). `common.md` is shared by all three
games; `jak1.md`, `jak2.md` and `jak3.md` cover each game's specifics.
Start from `index.md` and open only the file the task needs. Read this
skill for the "why" and the wiki for the "exact syntax to write".

---

## 1. Lexical conventions

- Comments: `;;` for a line, `#| ... |#` for a block.
- Booleans: `#t` and `#f`. Anything other than `#f` is truthy.
- Symbols are written bare (`my-symbol`) or quoted (`'my-symbol`); keywords
  are colon-prefixed (`:enter`, `:event`, `:inline`, `:virtual`).
- Floating point literals need a decimal point (`1.0`, not `1`).
- Unit macros convert real-world quantities into the engine's internal
  units: `meters` for distance, `degrees` for angle, `seconds` for time
  (ticks). Always use them instead of hardcoding raw internal values — see
  the wiki's "Core Lisp patterns" section for the exact call shape.

## 2. Type system

GOAL has a strongly typed, single-inheritance object system.

- `object` is the root of all types.
- `structure` is unboxed by default: stored as a 4-byte pointer unless
  marked `:inline`, in which case it is embedded directly in its parent.
- `basic` is a boxed object with a runtime type tag, checkable dynamically.
- `process` is the base for concurrent kernel "threads" (actors).
- `process-drawable` is a process that also has a position, skeleton,
  bounding box, and drawing components — the base most custom entities
  derive from.

`deftype` declares a new type: its fields, its methods (`:methods`), and the
states it can be in (`:states`). Vectors and matrices are 128-bit aligned;
a structure containing an inline vector or 128-bit field gets padded to
match. Misaligned fields corrupt memory or trigger CPU exceptions — when in
doubt, check a decompiled definition in `goal_src/jak[x]/` for the real
layout.

## 3. Processes and behaviors

A `behavior` is a function that runs in the execution context of a specific
process type. Inside a behavior, `self` is statically typed to that
process, so its fields and methods are known at compile time.

## 4. Virtual states and the state machine

Every process runs a finite state machine. A state can implement:

- `:enter` — runs once when entering the state.
- `:trans` — runs every frame before physics/collision.
- `:code` — the main coroutine loop; can suspend and resume across frames.
- `:post` — runs every frame after physics/collision, usually to flush
  animation or collision transforms.
- `:event` — an asynchronous message handler triggered by another process.
- `:exit` — runs once when leaving the state, even if interrupted.

A state transition can target a specific type's state directly, or dispatch
through the type's virtual table so a subclass's override wins — this
matters for residency (see §6 below and the wiki's pitfalls section).

## 5. Coroutine control

GOAL's kernel provides non-preemptive cooperative scheduling for processes:
a process can yield for exactly one frame and resume at the same point next
frame, or suspend indefinitely until an external event wakes it. This is
what lets a `:code` loop "wait" for an animation to finish without blocking
the rest of the engine.

## 6. Macros and conditionals

GOAL has the conditional and macro forms you would expect from a Lisp:
conditional execution (`when`/`unless`), multi-branch dispatch
(`cond`/`case`), and compile-time macro expansion (`defmacro`). The wiki's
worked examples use these throughout — there is nothing GOAL-specific about
their shape.

## 7. Why traps matter here specifically

GOAL's traps are not generic Lisp gotchas — they come from the engine
running as a simulated PS2 machine with hot-reloadable native code:

- Floating point equality comparisons fail on precision noise more often
  than in a typical scripting language, because gameplay code compares
  computed physics values every frame.
- A `basic` and a `structure` are allocated and checked differently — the
  compiler enforces this, but it explains why some engine helpers only
  accept one or the other.
- Hot-reloading via `(mi)` can leave "ghost" state in a running game session
  that a clean boot would catch immediately — this is the single most
  common source of "it worked in my session" bugs.
- A type's vtable slot is only filled once its defining file is linked into
  memory, which is a DGO-load-time event, not a compile-time one — this is
  why state/method placement (resident vs. level file) matters.
- The engine has three distinct memory heaps with very different
  lifetimes; picking the wrong one either leaks permanently or gets freed
  out from under you.

Every one of these has a concrete, verified code example and fix pattern in
the wiki's "Known pitfalls" section (Part 1.4) — read the trap names here,
then the code there.

## 8. Mod architecture: the in-game Mods menu

Every new mod (without exception for `jak[x]/features/*` branches) must be
toggleable at runtime and ship off by default, so it never changes default
game behavior unless a player explicitly turns it on. The mechanism is the
unified in-game Mods menu, opened with L3 + SELECT in both retail and debug
boots on Jak 2 and Jak 3 (Jak 1 does not have this menu yet — see the
wiki's Jak 1 section for the workaround). See
[`docs/modding/guides/mods_menu.md`](../../../docs/modding/guides/mods_menu.md)
for the architecture, the copy-paste template at
[`docs/modding/templates/mod_menu.template.gc`](../../../docs/modding/templates/mod_menu.template.gc),
and the exact registration call in the Lisp wiki ("Register an in-game Mods
toggle").

## See also

- [`goal-lisp/wiki/`](wiki/index.md) — every worked code example and trap, common patterns plus per-game specifics (including the engine model: memory heaps, DGOs, process lifecycle).
- [`engine-internals`](../engine-internals/SKILL.md) — the C++ runtime/compiler/build system this language compiles to and runs on.
