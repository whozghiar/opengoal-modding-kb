---
name: game-data
description: Regenerate what the editor knows of the game (data/jak3/game-data.json, particles, the game's prefab recipes, Jak II's creatures) or extract the game's levels. Use when the decompiled code changed, when the catalog or the game's prefabs need updating, or when the extraction is involved.
---

# The game's data

No asset of the game is in the repository: `data/` holds names and rules read from OpenGOAL's
decompiled code, and the levels are extracted from the game the user installed with OpenGOAL into
`library/` (ignored by git).

| File | Made by | From |
|---|---|---|
| `data/jak3/game-data.json` | `python tools/gen_game_data.py <jak-project>/goal_src` | the decompiled code (read only), `tools/object_categories.json` for the catalog's choices |
| `data/jak3/particles.json`, `data/jak3/sprite-pages.json` | `python tools/gen_particles.py <jak-project>/goal_src` | the `defpartgroup` / `defpart` of the code, and the particle curves (`particle-curve-settings`, `curve2d-fast`, `curve-color-fast`) |
| `data/jak2/texture-slots.json`, `data/jak3/texture-slots.json` | `python tools/gen_texture_slots.py <jak-project>/common/texture/texture_slots.cpp` | the animated texture slots of OpenGOAL's renderer, of the release the extraction uses (a model's draw names its animated texture by slot) |
| `data/jak2/creatures.json` | `python tools/gen_jak2_creatures.py <jak-project>/decompiler/config/jak2/all-types.gc` | Jak II's type table |
| `data/jak3/prefabs.json`, `data/jak2/prefabs.json` | `python tools/gen_prefab_recipes.py library --game jak3` (and `jak2`) | an extracted library, through a hidden editor (`target/release` or `target/debug/jak-level-editor`, or `--editor <program>`); `tools/prefab_review.json` keeps what was reviewed |

Rules:

- Read a checkout of the game's own code (not a level branch: a level replacing one of the game's
  changes that level's data). Never write into jak-project.
- Build the program again after changing `data/` (it reads the copy a build puts next to it, in
  `target/<profile>/data/`), then `cargo test -p jak-level-editor-core` (which reads `data/` itself).
- The recipes written by hand stay first in `prefabs.json`; the generated ones are marked `auto`.

## Extraction

`File > Extract game assets...` in the editor: OpenGOAL's own extractor of the game installed
(the OpenGOAL Launcher's folder, or the one of *File > Settings*) reads the levels by batches,
the editor writes the library, the models included (`crates/core/src/io/extraction/`,
docs/architecture.md section 20). Without the interface, the ignored test does the same:

```bash
JLE_GAME_INSTALL="<OpenGOAL Launcher folder>" JLE_EXTRACT_OUT=library [JLE_EXTRACT_LEVELS=CIA.DGO] \
  cargo test -p jak-level-editor-core --lib extracts_from_the_installation -- --ignored --nocapture
```

Jak 3 goes to `library/jak3/` (Jak II to `library/jak2/`, for its structures). Prefabs are in
`library/prefabricated/` (the user's) and `library/prefabricated/game/` (the game's, made level
by level from the recipes). All of `library/` but `prefabricated/.gitkeep` stays out of git.
