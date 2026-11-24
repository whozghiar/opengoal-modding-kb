---
name: smoke-tests
description: Run in-game smoke tests of a mod with a scenario that drives the game, and record them in the mod's test notebook. Use after adding a big feature to a mod (mandatory before calling it done), when the user asks to test a mod in game, or when writing a scenario, a pad hook or a screenshot for a test.
---

# In-game smoke tests

A smoke test boots the game with a scenario: GOAL code that puts the mod in a situation, checks
what the mod should do there, and prints the results. Compiling proves the code builds; a smoke
test proves it runs. Run one after every big feature of a mod: a new system, actor, level or menu,
or a hook in shared engine code.

## 1. Permission first

Agents never launch the game on their own (AGENTS.md, section 3). Ask the user once whether you
may launch it for the tests. Without a yes, write and compile the scenario, then give the user
the exact `run` command and ask what it printed. With a yes:

- launch through `scripts/modding/smoke_test.py run`, one scenario per call;
- run only the tests the user asked for, never a loop of runs;
- after a crash, stop and report it before running anything else.

## 2. The test notebook

Each mod keeps its tests in `docs/modding/current_mod/<slug>_tests.md`, started from
[`docs/modding/templates/TEST_NOTEBOOK.template.md`](../../../docs/modding/templates/TEST_NOTEBOOK.template.md),
and their scenarios in one file, `docs/modding/current_mod/<slug>_smoke.gc`, started from
[`docs/modding/templates/smoke_test.template.gc`](../../../docs/modding/templates/smoke_test.template.gc):
one numbered scenario per test.

A new feature adds its tests to both. After a run, write its result in the notebook: the date,
PASS or FAIL, and the evidence (the `SMOKE` lines, a screenshot name). A failure is recorded as it
happened, never left out.

## 3. Running one test

```bash
python scripts/modding/smoke_test.py install jak3 docs/modding/current_mod/<slug>_smoke.gc --scenario 2
task compile-check
python scripts/modding/smoke_test.py run jak3 <test-id> --timeout 300
python scripts/modding/smoke_test.py remove jak3
task compile-check
```

- In a mod worktree, the session's first compile is forced (`(make-group "iso" :force #t)`,
  AGENTS.md, section 8).
- `run` boots in debug (save slot 1 loaded) unless `--retail`, the launcher's boot: use it for
  what only a retail boot shows (the Mods menu, code that must not depend on the debug segment).
- Its exit code is 0 when the scenario printed `SMOKE DONE` and no `SMOKE FAIL` line. Everything
  else is in `out/smoke/<test-id>/`: `stdout.log`, the screenshots (open them: a PASS line can't
  see a model drawn wrong), and `log/<game>-crash.txt` after a crash.
- `git status` after `remove`: no `smoke-test` file or line may reach a commit.

## 4. Writing a scenario

- Set the state through the mod's own functions: its switch, a warp to a continue point. Never
  save: saves and settings persist across boots (AGENTS.md, section 3).
- One fact per `smoke-check` line; a count or a position goes in the line itself.
- A screenshot for every visual claim, taken after the scene settled. The game runs in a 640 x 480
  window and the screenshots come out 1280 x 720: everything is stretched 1.33 times across, so a
  circle drawn round in the window shows as an ellipse. Judge proportions in the window, not in
  the picture (the battlefront mod's radar, 2026-11-14: `Setting window size to 640 x 480` in the
  run's log, `FBO Setup: requested 1280x720` at each shot).
- A page of the START menu takes its keys only once the menu's ring has opened
  (`menu-transition` 0): wait for that before pressing X, or the press is lost (the battlefront
  mod's result page, scenario 51, 2026-11-14).
- GOAL traps that bite scenarios (a lambda captures no local variable, a handle kept in a global
  symbol loses its pid): search the [Lisp wiki](../goal-lisp/wiki/index.md) first.
- Keep a unit or a class model as a handle across a wait and resolve it after each wait: a
  process pointer goes stale when the unit dies or its dead-pool heap compacts. An attack event
  sent "from" a stale pointer crashed Jak 3 (the battlefront mod's runs: the sender printed as
  `invalid type ptr`, 2026-11-06). Read without a crash, it gives wrong numbers: a watch loop
  keeping the class model's pointer across frames read her 1.1 km away, the same place in every
  run, once a compaction moved her process; the game itself was fine (the battlefront mod's round
  55, 2026-11-24).
- The save the scenario loads may have secrets on, invulnerability among them: a test of damage
  to the player lifts the secret for the measure and puts it back as it was.
- Pad input, retail-boot screenshots, several instances at once, finding the GOAL function behind
  a crash: [`game-control.md`](game-control.md).

## 5. Not the last word

A smoke test is not the user's cold boot. Before calling a gameplay change done, still ask for
`task boot-game` (or `task boot-game-retail`) and what the user sees (AGENTS.md, section 3).
