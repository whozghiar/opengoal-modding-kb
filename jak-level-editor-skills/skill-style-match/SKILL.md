---
name: skill-style-match
description: Give new geometry a level's look in Jak Level Editor: the game's textures and lighting instead of shaders, shapes and structures in a level's style, and Blender models textured with a level's own materials, derived from reference images. Use when adding geometry that must look like a level, or making a model in Blender for the editor.
---

# Matching a level's look

The editor draws everything with the game's own renderer. So a level's look comes from its
**textures**, named after the game's (`city-ind-grnd-cobl-01`), and from its **lighting**, stored as
vertex colors (`og_tod`: 8 colors per vertex, one per light of the day). There is no shader graph
to copy. Matching a level means using its textures and letting the editor light the result.

## In the editor

Take the first of these that fits:

1. **A whole structure**: `build_structure` (wall, floor, house) or `build_house` with `level`.
   It is built from the level's own pieces and textures.
2. **A mass**: `place` a `shape` with `style` (the level) and `surface` (`floor`, `wall`,
   `building`, `roof` or `trim`). It takes the level's texture, its light and its bumps.
3. **Details**: `building_kit {"level": ...}` gives the parts by role (window, wear, pipe...) and
   their textures. Place them as `decor` of that level.
4. **An existing element in the wrong texture**: `list_textures`, then `set_material`.
5. **A structure of Jak II**: `match_lighting` brings it to the level's brightness.

Then check it with `capture_view` at the player's height (1.6 m above the ground), at a few
`game_hour`s. Rules 2 to 4 of docs/mcp.md still apply.

## Every part its texture

The game has a texture for every kind of part: plank floors, beams, railing slats, chairs and
tables, barrels, crates, metal pipes, stone, plaster, roof plates, doors, window frames, cloth.
A model looks right when each of its parts wears the texture the game gives that kind of part, in
the zone where it goes. Grey everywhere, or one wood for everything, reads as untextured beside
the game's pieces.

1. **List the parts** of the model by kind: walls, base, posts, beams, floor, stairs, railing,
   furniture (each piece), containers, pipes, doors, window frames and shutters, roof, cloth,
   lamps, rubble.
2. **Ask the game for each kind:**

   ```text
   python tools/game_textures.py --role railing --level ctysluma
   python tools/game_textures.py --role furniture --game jak2      # nothing in the level: the game
   python tools/game_textures.py --piece chair --game jak2         # by the name of the game's pieces
   ```

   - **The index:** `--build` makes it once, into `library/textures-index.json`. It lists each
     texture's color, alpha and meters per repeat, and the game's pieces (`on`) that wear it.
   - **Pick** the texture worn by the game's parts of the same kind, in that level first, then in
     its zone, then in the game.
   - **Tune:** `--color #rrggbb` ranks them by a color (one sampled on a reference picture);
     `--alpha BLEND` keeps the decals.
3. **Check:** each kind of part has its own texture, close to the color of the game's parts of
   that kind. A part left in a plain color or in a texture of an unrelated kind is a defect.

## In Blender, for a model of your own

`tools/blender_kit.py` (run inside Blender) gives the building blocks:
- **Parts:** `box`, `cyl`, `beam`, `decal`, each wearing the game's texture at the game's scale.
- **Images:** loaded from the extracted levels, their alpha doubled.
- **Collision:** a collision for every visible part, sorted by surface.
- **Export:** the `.glb` files the editor imports.

Its docstring shows the calls. It runs on the core of the add-on Jak Structure Creation Helper
(`tools/blender/jak_structure_creation_helper/core.py`): a person building in Blender's interface
gets the same from the add-on's Jak tab. The steps below are what it does, for a model made without
it.

1. Bring the level's materials into Blender. Import one of its prefabs
   (`library/prefabricated/ctyinda-floor-16m.glb`) into a scene of your own: its materials are
   named after the game's textures and carry their images.
2. Assign those materials to the new mesh (`me.materials.append(bpy.data.materials["..."])`) and
   unwrap it so the texture keeps the level's scale. The texture tile `building_kit` gives is how
   many meters one repeat covers.

   **Copy each image with its alpha doubled first.**
   - The extracted images keep the game's colors, but their alpha is the PS2's: 0.5 is opaque.
   - The game's materials say `baseColorFactor` 2. That factor is for their vertex lighting and
     their alpha, not for their colors.
   - The editor halves the alpha of a model's own images and draws their colors as they are. So
     an image used unchanged turns half transparent, and one with its colors doubled too comes
     out twice too bright (verified in the game rendering: `crates/core/src/game/view.rs`,
     `own_texture`).

   ```python
   import numpy as np
   px = np.empty(w * h * 4, np.float32); src.pixels.foreach_get(px)
   px[3::4] = np.minimum(px[3::4] * 2, 1)
   img = bpy.data.images.new("jak2-" + name, w, h, alpha=True); img.pixels.foreach_set(px); img.pack()
   ```

   Name the copy `jak2-<texture>` for a texture of Jak II (the editor's name for them).

   **Decals: as the game's materials say.** A level's richness comes from them: stains, streaks,
   vines, dirt, bricks showing. Read each one's `alphaMode` in the level's decor `.glb`:
   - **The slums' stains, streaks, vines and dirt** are `BLEND`, extras
     `{"og_translucent": true}`, `alphaCutoff` 0.149. To export one from Blender:
     1. Link the image's alpha straight to the BSDF's Alpha.
     2. Set `surface_render_method = "BLENDED"` and the material's custom property
        `og_translucent = True`, and export with `export_extras=True`.
     3. Write `alphaCutoff` 0.149 into the `.glb` afterwards. Blender writes none for `BLEND`,
        and the editor would then take glTF's 0.5, dropping the faint stains.
   - **Cut-out pieces such as clotheslines** are `MASK` 0.149: put a Math *Greater Than* 0.149
     between the alpha and the BSDF.

   An opaque or `MASK` stain draws as a black hole in the wall.
3. Keep one Principled BSDF per material, fed by a Base Color image. Procedural nodes do not
   survive the glTF export. A plain color becomes a small image when the model is imported. Keep
   images at 1024 px or less.
4. Export glTF Binary in meters, standing at the origin, with the modifiers applied. Import it
   into the editor and read the inspector's *Imported model* checks.

## From a reference image

When a brief or a picture asks for a new asset, follow blender-game-skills' gates (MIT,
`blender-image-to-3d` at `f0ef293`), adapted to the editor:

| Step | Gate |
|---|---|
| Brief | ask only what the image cannot show; give a default for the rest |
| Measure | on orthographic views only, against a known size (Jak is about 1.6 m) |
| Blockout | 15 to 60 primitives; its silhouette matches the reference before any detail |
| Budget | the game's: a piece 36 to 64 triangles at the median (a beam 12 to 48, a window 8 to 48, a door 36 to 96, a prop 36 to 150), a building 2,500 to 9,500, 0.5 to 2 per m² of facade ([haven-city-jak2.md](../skill-zone-design/haven-city-jak2.md), section 2); a model whose shape would put over 460 triangles around Jak collides by its box: give it a collision model ([skill-zone-design](../skill-zone-design/SKILL.md), step 5) |
| Textures | the level's materials (above); bake to one Base Color image only when nothing fits |
| Names | `<Asset>_<Part>`, no `.001` |
| Check | export, import the `.glb` again: same height (within 1%), nothing below the ground, same triangle count |

Run its scripts with `blender -b` only: they reset the open file
([skill-blender-context](../skill-blender-context/SKILL.md)).
