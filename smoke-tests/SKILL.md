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
- A screenshot for every visual claim, taken after the scene settled.
- GOAL traps that bite scenarios (a lambda captures no local variable, a handle kept in a global
  symbol loses its pid): search the [Lisp wiki](../goal-lisp/wiki/index.md) first.
- Pad input, retail-boot screenshots, several instances at once, finding the GOAL function behind
  a crash: [`game-control.md`](game-control.md).

## 5. Not the last word

A smoke test is not the user's cold boot. Before calling a gameplay change done, still ask for
`task boot-game` (or `task boot-game-retail`) and what the user sees (AGENTS.md, section 3).
