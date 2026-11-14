---
name: skill-zone-design
description: Design a new structure in the style of a zone of Jak 1, Jak II or Jak 3 (its shapes, construction, materials, wear and scale) before modeling it in Blender or building it in the editor. Use when asked for a building, prop or room "for" a level or district, or when a model must fit where it will be placed.
---

# Designing for a zone of the Jak games

A model fits a zone when the player cannot tell it from the game's own pieces: same silhouette
language, same construction, same textures at the same scale, same wear. This skill gives the
rules the three games share, the steps to follow, and one entry per zone:
[jak1.md](jak1.md), [jak2.md](jak2.md), [jak3.md](jak3.md). The entries name the levels'
codes (`ctysluma`), their textures as the game names them, and what the game's pieces measure.
Every level of a game, with its zone and what it holds (decor, a mission's objects, a cutscene),
is in [levels-jak1.md](levels-jak1.md), [levels-jak2.md](levels-jak2.md) and
[levels-jak3.md](levels-jak3.md): look a level's code up there first.

**Haven City and Jak II:** before modeling anything in Blender for Haven City or in Jak II's style,
read [haven-city-jak2.md](haven-city-jak2.md): the measured charter of its 34 levels (triangle
budgets by kind of piece, texture sizes and repeats, palettes and light per district, the pieces
each copies most). Its budgets are the limits to stay within; `tools/zone_charter.py` measures any
other level the same way.

How to get a level's textures onto a mesh and export it is
[skill-style-match](../skill-style-match/SKILL.md); running bpy safely is
[skill-blender-context](../skill-blender-context/SKILL.md).

## The design language of the three games

1. **Silhouette first.** The camera follows Jak from 6 to 10 m behind him: shapes read by their
   outline. Forms are chunky and soft: walls bulge or taper, posts flare at their foot, roofs
   overhang, edges are bevelled. Nothing is razor-straight or exactly repeated: lean a post 1 to
   3 degrees, vary a plank's length, turn a stone.
2. **Built, not sculpted.** Every structure shows how it stands: a **base** (stone plinth,
   stilts, rock), a **frame** (posts, beams, ribs, pipes), an **infill** (plaster, planks, plates,
   panels) and a **cap** (roof, cornice, dome). Joints are visible: straps, nails, bolts,
   brackets, ropes.
3. **One faction, one family of forms.**

   | Family | Forms | Materials |
   |---|---|---|
   | Precursor | smooth, round, symmetric; rings, domes, tubes, honeycomb plates; thin glowing lines | silver and dark metal, blue or teal glow, gold trims |
   | Villages and Wastelanders | hand-built, organic, rounded roofs, stilts, ropes | wood, thatch, plaster, stone, cloth, salvaged metal |
   | Haven City and the Krimzon Guard | large, heavy, ordered, rounded industrial masses; propaganda | painted metal (red, grey, green), concrete, pipes, cobbles |
   | Lurkers, Marauders | crude and spiky, scrap riveted together, chains, bones | rusty metal, hide, rope, bone |
   | Metal Heads | organic and armored: tentacles, ribs, eggs, gems | chitin plates, veins, dark rock |
   | Dark Makers | alien biomechanical: plates over tubes and tentacles | purple and black hard plates, organic tubes |

4. **Wear tells a story.** Wear sits where it would happen: stains under windows and roofs,
   plaster broken off at corners and bases showing the bricks, rust streaks under metal, mismatched
   patches over damage, rubble below a breach. Damage stays local: a ruined building still reads
   as the building it was.
5. **The game's scale, not ours.** Jak's numbers (Jak II's `*TARGET-bank*`): body radius 0.7 m,
   jump 3.5 m, double jump about 6 m, edge grab 2.3 m above the ground. The game's buildings are
   larger than real ones: a slum door of Jak II is 4.5 x 7.5 m, a storey 12 to 15 m. Interiors are
   15 to 25 m high (the Underground's hideout 17 m, Onin's tent 24 m) so that the camera fits.
   Take sizes from the zone's entry, or measure its pieces (below), never from real buildings.
6. **Detail is in the texture.** The game's images are 64 or 128 px, and a wall's repeats every
   1.6 to 9 m (4.4 m in most levels), its image upright 7 times in 10. A kit piece has 2 to 130
   triangles: a stain 2, a door 8, a window 32, a beam 20, a lamp 124. The relief worth modeling is
   the one the camera sees in silhouette: frames, beams, overhangs, rubble.
7. **Light comes from the editor.** It lights each vertex from the level around it (`og_tod`).
   Model no lights, bake nothing. A lamp is a model whose glass uses the level's lit texture
   (`city-slum-litwindow`).

## How the game builds, measured

`tools/game_buildings.py` measures the game's buildings (the game's prefabs built in Blender by
`tools/level_prefabs/game_build.py` into `<library>/prefabricated/game/<game>/<level>/`) level by level, into `<library>/building-patterns.json`. Read a
level's before designing anything for it:

```text
python tools/game_buildings.py --level ctysluma                         # its pattern, in words
python tools/game_buildings.py --level ctysluma --compare <prefab.glb>  # what a model does unlike it
```

What holds in every level (130 buildings of Jak 3, 71 of Jak II):

- **Assembled, not modeled.** A building is 50 to 250 copies of 20 to 40 of the level's own
  pieces: facade modules named for their size (`city-slum-facade-bottom-48w-12h-02`: 48 x 12
  units, used at scale 0.8 in the slums, 0.8 to 1.1 elsewhere), stacked a floor at a time, the
  ends closed by thin walls (`facade-divider`), then roof pieces, then details.
- **Few triangles, spent on details.** About 0.9 triangle per m² of facade, 6,700 a building
  (1,450 to 20,800). The facades are cheap (12 to 260 triangles a module); the triangles go to
  vines, roofs, pipes and beams.
- **Every copy differs.** The same piece is scaled differently on each copy, turned 5 to 40
  degrees off its wall, leant. The slums' roof slats ring the roof's edges, each turned out of it,
  so that the roof slopes to every side.
- **Pieces that go together:** a door always with its frame, at the same spot.
- **Wear is 2-triangle quads** laid once on the wall (blotches, stains, bricks showing): 20 to 60
  a building.
- **Textures:** a level's buildings wear its own few textures. Walls and wood repeat; doors,
  windows, stains and signs are laid once on their face. A texture of another level, or one
  repeated where the game lays it once, is what makes a model look wrong.

25 levels of Jak 3 and 12 of Jak II build with facade modules. For them, the closest to the
game's look is one of the level's own buildings placed in the editor as a **variant** of itself
(the MCP's `place` with `variant` and `mirror`: its structure as the game made it, its stains,
vines, signs and lamps changed). The others build with their shell: their masses are made, their
details are the level's pieces.

## Steps

1. **Find the zone.** Open the entry of the level where the model goes (the codes are in each
   entry's title); for Haven City or Jak II, its card in [haven-city-jak2.md](haven-city-jak2.md)
   too. Done when you can name its family, its five main textures and three details you will
   reproduce, and the triangle budget of each kind of piece you will make.
2. **Read its pattern, then look at it.** `game_buildings.py --level <level>` gives its
   buildings' sizes, pieces, budget and textures. Then import the level's decor
   (`library/<game>/<level>/decor/*.glb`) into a scene of your own and render views around its
   pieces (the script below). Hide the objects named `*collision*`: they are the level's
   collision and cover the decor. Done when you have looked at one of the game's buildings of the
   same kind from the street, and know which of its pieces you will use.
3. **Assemble the game's pieces first.** A building the level has: place it in the editor as a
   variant. A wall, a floor or a house of its pieces at your size: `build_structure`. In Blender,
   start from the level's pieces and buildings (`blender_kit.piece`, `blender_kit.building`, the
   add-on's **Add the piece** and **Add the building**). Model only what the level has no piece
   for, 15 to 60 primitives, at the level's budget per m².
4. **Detail** what you modeled; then the **wear**, as the level's 2-triangle quads. Give each part
   one of the textures the level's buildings wear (the add-on's palette), mapped as the game maps
   it (the add-on's materials and **Game scale UV** do: repeated at the measured meters, or laid
   once on the face).
5. **Collision on every part.** Jak must meet everything he sees: walls, floors, posts,
   railings, furniture, crates, barrels, pipes, roof.
   - **Why a model of its own:** the editor makes a model collide by its shape, or by the box
     around it when its shape is too dense (about `2 x triangles x 40 / footprint (m²)`
     triangles around Jak, over 460). A model Jak walks into needs a **collision model**.
   - **How:** the add-on's **Make the collision** (`blender_kit.kit_prefab`, `core.make_collision`)
     makes one collision model per surface (stone, wood, metal): one box a piece of the game (a
     facade module's stops 1 m out of its wall, so that its porch is no block in the street), a
     box per connected piece of what you modeled (each step of stairs its own).
   - **Left out:** the game's windows, vines, wires, lamps, signs and stains; decals; parts under
     0.25 m. Loose rubble is drawn with `col=None`, and its heap gets one `solid(...)`.
   - **Export a prefab:** **Prefab (ready to place)** (`kit_prefab`) writes
     `<library>/prefabricated/<name>.glb` with its collision set, its pivot at the bottom middle.
6. **Check** against the level: `game_buildings.py --level <level> --compare <prefab.glb>` lists
   what departs from its buildings (triangles, foreign textures, textures repeated where the game
   lays them once). Then in the editor: place it, `capture_view` from 1.6 m above the ground next
   to the game's own buildings of the zone (docs/mcp.md, rule 3).

## Looking at a zone in Blender

```python
import bpy, math, mathutils
LEV = "D:/.../library/jak2/ctysluma/decor/ctysluma-background.glb"   # the level's decor
sc = bpy.data.scenes.new("ref-ctysluma")
before = set(bpy.data.objects)
with bpy.context.temp_override(scene=sc):
    bpy.ops.import_scene.gltf(filepath=LEV)
for o in set(bpy.data.objects) - before:            # into our scene only
    for c in list(o.users_collection):
        if c != sc.collection: c.objects.unlink(o)
    if sc.collection not in o.users_collection: sc.collection.objects.link(o)
    o.hide_render = "collision" in o.name
cam = bpy.data.objects.new("ref-cam", bpy.data.cameras.new("ref-cam")); sc.collection.objects.link(cam)
cam.data.lens, sc.camera = 24, cam
sc.render.engine = "BLENDER_WORKBENCH"; sc.display.shading.light = "FLAT"
sc.display.shading.color_type = "TEXTURE"
door = next(o for o in sc.objects if o.name.startswith("city-slum-door-01"))   # a piece of the kind you make
c = sum((door.matrix_world @ mathutils.Vector(v) for v in door.bound_box), mathutils.Vector()) / 8
for k, az in enumerate((0, 90, 180, 270)):
    a = math.radians(az)
    cam.location = c + mathutils.Vector((math.cos(a) * 28, math.sin(a) * 28, 6))
    cam.rotation_euler = (c + mathutils.Vector((0, 0, 5)) - cam.location).to_track_quat("-Z", "Y").to_euler()
    sc.render.filepath = f"<scratch>/ref-{k}.png"
    bpy.ops.render.render(write_still=True, scene=sc.name)
```

The decor's objects are named after the game's pieces (`city-slum-facade-mid-48w-12h-01.mb`):
their names list the zone's kit, and their bounds measure it. Delete the scene afterwards.

## An example: what the measures say of a house modeled by hand

The prefab `slum-house-damaged` (2026-11-01) was modeled part by part in Blender: a 24 x 18 m ground
floor in plaster on a stone base, an upper storey jutting out, a gable roof of slats, a breach with
its rubble, decals. It looked like the slums from afar, and the user found it off: too many
polygons, not the game's look, textures badly laid. `--compare` says why:

```text
13800 triangles drawn, 3.60 per m2 of facade: ctysluma's buildings have 0.98 (7924 a building)
textures ctysluma's buildings never wear: barrel01, city-mark-wood-plain, ...
slum-stone-bottom: repeated, where the game lays it once on a face   (and six others)
```

One of the slums' own buildings placed as a variant (`Slum building (two floors)`: 316 copies of
the slums' pieces; their buildings have 7,900 triangles at the median) passes it by definition.
