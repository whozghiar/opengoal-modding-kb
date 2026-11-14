---
name: skill-modular-snapping
description: Place Jak Level Editor's modules on a grid: units, axes, pivots, module sizes, wall and floor conventions, and the layout builder tools/ai_level_builder.py (ASCII grid to MCP calls or Blender instances). Use when laying out rooms, streets or tiles, or converting positions between the editor and Blender.
---

# Modules, grid and pivots

## Units and axes

- Meters, Y up, in the editor, its MCP tools and its `.glb` files. The game counts 4096 units per
  meter.
- Blender is Z up. Its glTF importer turns the editor's `(x, y, z)` into Blender's `(x, -z, y)`,
  and a yaw about the editor's +Y becomes the same angle about Blender's +Z.

## Pivots

| What | Pivot | Source |
|---|---|---|
| The user's prefab (`library/prefabricated/*.glb`) | bottom center of its bounds (`ctyinda-floor-16m`: -8, 0, -8 to 8, 0.8, 8) | docs/architecture.md, *Prefab* |
| `build_structure` wall | middle of its base; it runs along x, its front towards +z; `yaw_deg` 0 faces +z | docs/mcp.md, *Walls and floors that grow* |
| `build_structure` floor | middle of its base, lying in x and z | same |
| `building_kit` part turned by a yaw alone | middle of its bottom; decals 2 cm in front of the wall | docs/mcp.md, *Build any structure from a level's kit* |
| A model made in Blender | the origin, standing on the ground | docs/user-guide.md, *Your own models* |

A game prefab goes by the name `list_prefabs` gives it (`Canal bridge`), not by its file's
(`game/jak3/ctygenb/canal-bridge.glb`). Its pivot is wherever the game had it: `Canal bridge`'s deck stands
0.78 m above it and 1.65 m off it along x. Measure the surface you join with `raycast`
(`200 - distance` from a ray cast down from y 200), then give the item an `offset`.

## Module sizes (the industrial zone, measured on the files)

| Prefab | Size x, y, z (m) |
|---|---|
| `ctyinda-floor-16m` | 16 x 0.8 x 16 |
| `ctyinda-floor-32m` | 32 x 1.5 x 33.7 |
| `ctyinda-wall-18m` | 21.7 x 41.1 x 10 (its pillars stand out of both ends) |
| `ctyinda-wall-35m` | 51.1 x 41.1 x 10 |
| `ctyinda-wall-with-windows`, `ctyinda-high-wall` | 31.2 x 41.1 x 10 |
| `ctyinda-house-1-floors` (2, 3) | 51 x 23.2 (34.3, 45.3) x 29.8 |

The sets of `ctygenb`, `ctysluma` and `ctyslumb` have the same names (`ctyslumb`'s walls are 20 and
39 m). Measure a module before you tile it: the box `place` and `list_groups` return, or the
POSITION bounds in the `.glb`. Its size, not its name, is the grid step.

## Snapping

- The gizmo moves by 0.5 m, turns by 15° and scales by 0.1 when snapping is on (`GizmoSnap`,
  off by default).
- Tile floors on a step equal to their size: 16 m for `ctyinda-floor-16m`.
- A wall or floor of `build_structure` takes any length and grows with `set_structure`. Make a
  run of edges one wall, and never stretch it with a scale.
- Turn modules by multiples of 90° on a grid. A wall's yaw puts its front towards the room:
  0 (room at +z), 180 (-z), 90 (+x), -90 (-x).

## The layout builder

`tools/ai_level_builder.py` turns a grid of characters into the editor's calls. It places the
legend's prefabs at `cell` meters per character with `place_many`. It joins the sides of walled
cells that face an empty cell into one `build_structure` wall per run, `inset` meters out and
lengthened by two insets so that the corners close. The script's docstring describes the format.

| Command | What it does |
|---|---|
| `python tools/ai_level_builder.py layout.json --calls` | prints the MCP calls: make them with `mcp__jak-level-editor__place_many` and `build_structure` |
| `--run --view out.png [--eye X Z LX LZ]` | builds the layout in a private editor (its own preferences: the user's editor and projects are left alone) and saves `capture_view`; `--eye` adds `out-eye.png`, the player's view 1.6 m above the ground (rule 3) |
| `--blender out.py` | writes a bpy script that imports each prefab once and places collection instances in a new scene `jle-layout`; run it with the Blender MCP's `execute_blender_code` |
| `--selftest` | checks the parser and the walls on the built-in demo |

The built-in demo has two 32 m rooms of `ctyinda` with a `Canal bridge` between them: 9
placements and 8 walls.
