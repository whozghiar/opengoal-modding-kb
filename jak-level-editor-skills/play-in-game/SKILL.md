---
name: play-in-game
description: Play a level made in the editor in the real game (Jak 3): F5 tries it in the game installed (no install, no compile), or export it and add it to a jak-project by hand. Use when the user wants to play or verify a level in the game.
---

# From the editor to the game

## Try it (F5)

**Level > Try in the game** (F5, the toolbar's Try; the engine's call `test_in_game`, its state in
`game_test_state`) plays the open level in the game the user installed: the OpenGOAL Launcher's
(`--game-install`), or the folder of *File > Settings, Games to try levels in* (the call
`set_game_install {game, folder}`; a Launcher installation, an OpenGOAL release or a jak-project).
It exports the level as `jle-test` and has the game's own `goalc` build it, its code and its DGO
(about 20 s more the first time of a session, while goalc learns the game), puts `JLT.DGO` and `jle-test.fr3` into the game's
`out/jak3/`, starts `gk` and brings Jak to the level's start. Nothing of the game is installed or
compiled; closing the editor (`stop_game_test`) closes that game and takes the two files out.
`docs/architecture.md` (24, *The try in the game*) tells how.

- Only one program can hold Jak 3's REPL port, 8114 (gk ignores `--port`): a try refuses to start
  while another game runs. Never stop another program's game (another agent's tests may run one):
  wait, or ask the user.
- When it fails, the log says why; `<prefs>/game-test/jak3/goalc.log` and `gk.log` hold what the
  compiler and the game printed.

## By hand

To keep the level in the game, or for a game installed elsewhere: the editor exports the level as
files to copy into a jak-project, and the guide's tutorial *Add your level to the game by hand*
(`manual`) walks the user through the steps below (gk has no option to start at a level: goalc takes
Jak there). Nothing installs the level for them (no manifest, no backups): it is done by hand.
Always give jak-project's programs `--proj-path` (else they find the project by the last
"jak-project" in their path).

0. **The editor**: `target/release/jak-level-editor.exe` (or `target/debug/`, or
   `npx yarn tauri dev`; build it first: the build-and-test skill), with `--library <folder>` when
   the levels are not in `library/`; or the MCP server, which starts it.
1. **Export**: `Level > Export for the game...` or the MCP tool `export_level` (`name`, `replaces`
   for a level taking the place of one of the game's). It writes `library/exports/<project>/` laid
   out like a jak-project checkout, with a `README.md` giving the exact lines to add.
2. **Into jak-project** (the user's own checkout: ask which branch before touching it, and say how
   to go back): copy `custom_assets/` and `goal_src/` of the export into it, add the lines of the
   export's `README.md` (`game.gp`; `level-info.gc`, or the one `:nickname` to change for a
   replacement). The exported `.glb` holds the game's data: keep it out of git (`.git/info/exclude`).
3. **Build**: the compiler must come from the same branch (`cmake --build out/build/Release
   --target goalc`), then `goalc -g jak3` and `(mi)` in its REPL.
4. **Play**: start `gk` (`-boot -fakeiso -debug` for a debug game), then in goalc `(lt)` and
   `(start 'play (get-continue-by-name *game-info* "<name>-start"))`.

Checks worth doing in the game: the level loads (`(level-status? *level* '<name> #f)`), Jak stands
on its floors (the export's report lists the places denser than the game's collision cache), doors
open, and nothing shows the void (look around it in the editor's game rendering first). Close `gk` and `goalc` when
done.
