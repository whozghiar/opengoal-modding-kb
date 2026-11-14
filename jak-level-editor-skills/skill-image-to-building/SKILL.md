---
name: skill-image-to-building
description: Reproduce what a picture shows (a building, a facade, a ruin, a street of them) in a level of Jak Level Editor: the level's own buildings placed as variants where the picture has them, Blender only for what the level has no piece for. Use when the user gives an image with a request to make it, or something like it, for a level.
---

# From a picture to the level

The picture decides the **layout**: how many buildings, where they stand, which way they face,
their size against each other, what stands between them. The level decides the **buildings**, the
**scale** and the **textures**: the level's own buildings and pieces are placed, as the game's
streets are made ([skill-zone-design](../skill-zone-design/SKILL.md), *How the game builds,
measured*), and only what the level has no piece for is modeled. The MCP server
`jak-level-editor` does the whole route: `reference_image`, `list_prefabs`, `place` with a
`variant`.

## Steps

1. **See the picture and choose the level.** `reference_image {path}` shows it with its main
   colors. Choose the level by the user's words first ("for Spargus"), then by what the picture
   shows against the zones' entries of [skill-zone-design](../skill-zone-design/SKILL.md) (its
   architecture: plaster and wooden frames, stucco blocks and spikes, metal walls). Colors alone
   tell levels apart badly: the game's walls are all greys, browns and beiges. Done when the level
   is chosen and you can say why in one sentence.
2. **Read the picture into a table, before placing anything.** One row per structure: where it
   stands (in the picture, then in meters on the level's ground: the plan), which way it faces, its
   footprint and floors, and which of the level's buildings resembles it most (`list_prefabs`: the
   game's prefabs of that level, with what each is). Mark as a guess what the picture does not show.
   Done when every structure you can see has its row.
3. **Scale.** A building of the game keeps its size: plan the distances from the buildings' own
   sizes (`list_groups` gives each placed group's box), never from the picture's real-world sizes
   (a real door is 2 m, the slums' 5 m). Done when each row has its position in meters.
4. **Place the buildings.** `place_many` the level's buildings where the plan puts them, turned as
   the picture shows (`yaw_deg`). Give every copy of the same building another `variant` (1, 2,
   3...) and `mirror` some: the same building twice looks copied. Ground, streets and walls between
   them: `build_structure` and shapes in the level's style. Look from above (`set_camera` high,
   `capture_view`): buildings must not overlap. Done when every row stands in the level.
5. **What the level has no piece for** (a footbridge, a stall): `build_structure` / `build_house`
   when it is a wall, a floor or a house; otherwise model it in Blender (below). Done when nothing
   of the picture is missing, or the user has been told what is.
6. **Compare.** `set_camera` at the picture's viewpoint (the same side, the same height, about the
   same lens), `capture_view` with the game rendering at the picture's hour, and set it beside the
   picture. List what differs: placement, silhouette, proportions, colors.
   - Fix the list (another variant, moves, turns), at most two rounds.
   - Then show both pictures to the user before going further.

   Done when the list is empty or the user has seen it.
7. **Check in the level.** `capture_view` from 1.6 m above the ground among them (docs/mcp.md,
   rule 3); facades whose back the player could see get a wall behind (rule 2). In the game, only
   if asked: Level > Try in the game (its game runs on a port of its own: other games may run),
   never touching another game's process or window.

## What the picture cannot give

- **The game's scale**, from step 3 only.
- **Lighting and shadows:** they belong to the editor, which lights what is placed like its
  surroundings. Model none, bake none.
- **A texture of another level:** a building of the slums wears the slums'. When nothing of the
  level resembles a material of the picture, keep the level's closest of the same role and tell
  the user which and why.
- **Exact geometry:** the aim is a street the player reads as the picture's, built the game's way,
  not a copy of every brick.

## Modeling what the level lacks, in Blender

With Blender's MCP server (`execute_blender_code`, [skill-blender-context](../skill-blender-context/SKILL.md)),
in a scene of your own: `tools/blender_kit.py`'s `piece(level, name, at)` (the level's own pieces),
`building(level, name, at)` (one of its buildings as the game made it, to model next to), the
kit's shapes in the game's textures, then `kit_prefab(objs, name)` (collision made, a prefab of the
library, ready to `place`). [docs/blender-addon.md](../../../../docs/blender-addon.md) walks through
one (a footbridge).

Before saving one, run the gate of
[`blender-asset-qa`](../../blender-agent-skills/blender-asset-qa/SKILL.md) on it **and on the level's
building it stands next to**: its `game` profile fails the game's own buildings too (scaled pieces,
face winding, duplicate vertices, stacked UVs: how the game's pieces are made). Fix only what the
level's building lacks: z-fighting, faces back to back, floating parts. To compare in Blender, the
Blender MCP server's `get_viewport_screenshot` gives the view.
