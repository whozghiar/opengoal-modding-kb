---
name: level-building-mcp
description: Build or change a Jak 3 level in Jak Level Editor through its MCP tools (place prefabs and decor, build houses and structures in a level's style, make prefabs, place facades, make it playable, light and weather, save, export). Use whenever the user asks to build, decorate or fix a level with the editor, and before creating prefabs.
---

# Building levels through the MCP server

The editor's MCP server (`jak-level-editor-mcp`, the server `jak-level-editor` of `.mcp.json`: its
tools are `mcp__jak-level-editor__<tool>`) acts on the editor window the user sees, and starts the
editor when none runs (build both first: the build-and-test skill). The full guide, with
examples, is `docs/mcp.md` (its part 2): read it the first time. Positions are meters, Y is up;
every change is undone with `undo`.

## The four rules (always)

1. **Prefabs hold structures, not crumbs.** `create_prefab` leaves out elements under 0.5 m
   (`min_size`): keep that; never save loose bolts, debris, stains or tiny props in a prefab. Check
   its `left_out` answer.
2. **Facades are not buildings.** Most buildings of the game are only a front, open behind. Place a
   facade so that only its front can be seen (against a wall or a cliff, the edge of the level,
   between other buildings, back to back with another facade), never in the middle of a place the
   player can walk around. Look at it from the back and the sides (`set_camera` 1.6 m above the
   ground + `capture_view`): if its back shows, move it, turn it, or hide its back behind other
   pieces touching it.
3. **Look before you report.** `capture_view` from the player's height (1.6 m above the ground)
   around what you built, in the game rendering, and from above. Describe what you see, problems
   included.
4. **Build like the level, and solid.** Walls, floors and houses come whole from `build_structure`
   (`kind`, `level`: the one it must look like): solid on every side, unlike the game's facades;
   resize them with `set_structure`, never `transform_nodes` with a scale. Any other structure: `building_kit` first (the level's parts by role, their size and
   rotation, its buildings' textures), then the masses as shapes with `style` + `surface`
   (`building` for a building's walls, `roof`, `trim`) and relief (a plinth, bands, pillars out of
   the wall, overhangs), then the level's own parts on them (windows in openings their size, wear
   between the openings, pipes, roof plates), each turned by its `rotation_deg` plus the wall's yaw.
   Size from the kit: a floor is about two of the level's windows high (11 m in the slums).

## A session

1. `get_status`; `open_level` (or `new_level`); `find_nodes` / `raycast` to know where things are.
2. Choose: `list_prefabs`, `list_catalog` (`group`: objects, effects, interactive), `list_decor`,
   `list_models`; the user's own prefabs are in `library/prefabricated/`.
3. Place: `place` / `place_many` (`on_ground: true` drops onto the surface below). A prefab placed is
   a group: `list_groups`, `select` with `group`. Many alike: `repeat` (row, circle, scatter);
   in order: `arrange` (`ground` so that nothing floats). Playable: `place` with `kind: "gameplay"`
   (`start`, `checkpoint`, `creature` with its type in `creature`, `death_zone` under the level),
   `set_route` for a moving platform, `generate_navmesh` where creatures walk (before placing
   creatures: the export leaves out a creature with no nav mesh under it).
4. Rule 2: check the backs of the facades from where the player walks.
5. Light and weather: `set_view` (`hour`, `weather`: clear, few_clouds, haze, smog, overcast,
   overcast_fog, rain, storm, storm_rain, storm_fog), `match_lighting` for pieces of Jak II.
6. Rule 3, then `save_project` (only when the user asks to save) and `export_level` when asked.

## Do not

- Do not save, export or delete without being asked; never touch the user's own prefabs or
  projects unless asked.
- Do not put the game's files (`library/`, exports, `.glb`) in git.
