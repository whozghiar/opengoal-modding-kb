# OpenGOAL Lisp Wiki

This wiki is the source of truth for writing OpenGOAL (GOAL) Lisp across the
Jak trilogy. Every instruction here is verified — compiled and seen working in
game, or confirmed present in the actual `goal_src/` of the game it's
attributed to. Consult it before writing or changing any `.gc` file so you
never invent an instruction that doesn't exist. Every GOAL/Lisp code
example in this project lives in these files — nowhere else in the
repository carries GOAL code snippets (the one exception is a `.gc`
template meant to be copied, e.g.
[`templates/mod_menu.template.gc`](../../../../docs/modding/templates/mod_menu.template.gc)).

Part 1 covers everything shared by Jak 1, Jak 2, and Jak 3 — the language
itself barely changed across the trilogy, and most engine subsystems are
identical. Parts 2-4 cover what's actually specific to each game: real
subsystem differences, not restatements of the shared material.

## Contents

- **Part 1 — Common to all games**
  - [1.1 Vocabulary](common.md#11-vocabulary)
  - [1.2 Core Lisp patterns](common.md#12-core-lisp-patterns)
  - [1.3 Engine model](common.md#13-engine-model)
  - [1.4 Known pitfalls](common.md#14-known-pitfalls)
- **Part 2 — Jak 1 specifics**
  - [2.1 Vocabulary delta](jak1.md#21-vocabulary-delta)
  - [2.2 Memory](jak1.md#22-memory)
  - [2.3 Compile / validate loop](jak1.md#23-compile--validate-loop)
  - [2.4 In-game Mods toggle: not ported](jak1.md#24-in-game-mods-toggle-not-ported)
  - [2.5 The debug root menu is fragile at link time](jak1.md#25-the-debug-root-menu-is-fragile-at-link-time)
- **Part 3 — Jak 2 specifics**
  - [3.1 Vehicles: flags, grab rails, driver methods](jak2.md#31-vehicles-flags-grab-rails-driver-methods)
  - [3.2 Traffic manager](jak2.md#32-traffic-manager)
  - [3.3 Virtual method / state residency in practice](jak2.md#33-virtual-method--state-residency-in-practice)
  - [3.4 Live re-skinning a process](jak2.md#34-live-re-skinning-a-process)
  - [3.5 Merc geometry & FR3 residency](jak2.md#35-merc-geometry--fr3-residency)
  - [3.6 Traffic engine & Haven City population](jak2.md#36-traffic-engine--haven-city-population)
  - [3.7 Process heap & art-group lifetime diagnostics](jak2.md#37-process-heap--art-group-lifetime-diagnostics)
  - [3.8 Memory](jak2.md#38-memory)
  - [3.9 Additional verified facts](jak2.md#39-additional-verified-facts)
- **Part 4 — Jak 3 specifics**
  - [4.1 Dark Jak stages](jak3.md#41-dark-jak-stages-darkjak-stage-bitfield)
  - [4.2 Secrets menu](jak3.md#42-secrets-menu-game-secrets)
  - [4.3 Powers/weapons interplay](jak3.md#43-powersweapons-interplay)
  - [4.4 Weapon system (gun)](jak3.md#44-weapon-system-gun)
  - [4.5 Custom art-groups: no dynamic linking hook](jak3.md#45-custom-art-groups-no-dynamic-linking-hook)
  - [4.6 Memory](jak3.md#46-memory)
- [How to contribute](../../kb/SKILL.md)


To add or correct an entry, follow the [`kb` skill](../../kb/SKILL.md): find the topic, edit it in
place, cite the evidence, then commit and push from the knowledge-base submodule.
