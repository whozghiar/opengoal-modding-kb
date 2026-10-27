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
  - [4.7 Start menu: adding a page](jak3.md#47-start-menu-adding-a-page)
  - [4.8 Making the player follow another process](jak3.md#48-making-the-player-follow-another-process-target-grab-ride)
  - [4.9 Playable Daxter (indax mode)](jak3.md#49-playable-daxter-indax-mode)
  - [4.10 Wrapping a stock method from a GAME file](jak3.md#410-wrapping-a-stock-method-from-a-game-file)
  - [4.11 Making a process hittable by the player's attacks](jak3.md#411-making-a-process-hittable-by-the-players-attacks)
  - [4.12 Drawing another process's model elsewhere](jak3.md#412-drawing-another-processs-model-elsewhere)
  - [4.13 Hooks for every effect, projectile, sound and story step](jak3.md#413-hooks-for-every-effect-projectile-sound-and-story-step)
  - [4.14 Keeping world objects alike on several machines](jak3.md#414-keeping-world-objects-alike-on-several-machines)
  - [4.15 Levels: memory banks, and what outlives a level](jak3.md#415-levels-memory-banks-and-what-outlives-a-level)
  - [4.16 Drawing the objects of another place, outside the banks](jak3.md#416-drawing-the-objects-of-another-place-outside-the-banks)
  - [4.17 A game loaded from memory, and saves turned off](jak3.md#417-a-game-loaded-from-memory-and-saves-turned-off)
  - [4.18 What the game removes out of its player's view](jak3.md#418-what-the-game-removes-out-of-its-players-view)
  - [4.19 A clip on part of a skeleton](jak3.md#419-a-clip-on-part-of-a-skeleton)
  - [4.20 The order processes run in](jak3.md#420-the-order-processes-run-in)
  - [4.21 How missions check the player: waypoints and races](jak3.md#421-how-missions-check-the-player-waypoints-and-races)
  - [4.22 Leaper lizards (`flut`): where their code lives, spawning one](jak3.md#422-leaper-lizards-flut-where-their-code-lives-spawning-one)
  - [4.23 A borrowed level on demand, and an engine shadow crash](jak3.md#423-a-borrowed-level-on-demand-and-an-engine-shadow-crash)
  - [4.24 Mission failures: the resetter](jak3.md#424-mission-failures-the-resetter)
  - [4.25 Burning bush challenges (`bbush`)](jak3.md#425-burning-bush-challenges-bbush)
  - [4.26 Places at the same coordinates](jak3.md#426-places-at-the-same-coordinates)
- [How to contribute](../../kb/SKILL.md)


To add or correct an entry, follow the [`kb` skill](../../kb/SKILL.md): find the topic, edit it in
place, cite the evidence, then commit and push from the knowledge-base submodule.
