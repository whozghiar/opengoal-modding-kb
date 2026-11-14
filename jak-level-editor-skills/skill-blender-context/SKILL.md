---
name: skill-blender-context
description: Run Python in Blender for Jak Level Editor (the Blender MCP's execute_blender_code, blender -b): where it runs, context overrides, bmesh, evaluated meshes, orphan data, Blender 5.2 API changes. Use before sending any bpy code to Blender.
---

# Running bpy code in Blender

Blender only makes models here: levels are assembled in the editor
([docs/ai_pipeline_context.md](../../../../docs/ai_pipeline_context.md)). A model leaves Blender as
a `.glb` (rules: the user guide's *Your own models: from Blender*). For bpy beyond this page,
[`blender-core`](../../blender-agent-skills/blender-core/SKILL.md) (imported) routes to the
`blender-*` skills; for the game's models, the game's rules
([skill-style-match](../skill-style-match/SKILL.md)) come before their materials and renders.

## The connection

| Piece | Where |
|---|---|
| Blender | 5.2 LTS, `C:/Program Files/Blender Foundation/Blender 5.2/blender.exe` |
| Add-on | Blender Lab's *MCP* extension (v1.0.3), port **9877**, started with Blender |
| MCP server | `blender` (local scope of this repository): Blender Lab's `blender-mcp` v1.0.3 through `uvx`, `BLENDER_MCP_PORT=9877` |
| Other add-on | Siddharth Ahuja's *MCP for Blender* stays on 9876 for other tools: leave it alone |

The bridge takes one JSON request per connection, `{"type": "execute", "code": "...",
"strict_json": false}` followed by a NUL byte, and answers `{status, result, stdout, stderr}`
the same way. A script can talk to it directly when the MCP server is not loaded. Without a
Blender window open, start one serving the bridge:
`blender -b --command blender_mcp --port 9877`.

## The user's open file is production

1. Work in a scene of your own (`bpy.data.scenes.new("jle-...")`) or a collection of your own.
   Never touch the user's scene.
2. Save only a copy, and only to a path the user gave or to the scratchpad:
   `bpy.ops.wm.save_as_mainfile(filepath=..., copy=True)`.
3. These calls wipe or replace the open file, so keep them for `blender -b` runs on a file of
   your own: `read_factory_settings`, `read_homefile`, `wm.open_mainfile`, `wm.save_mainfile`.
   Imported third-party scripts call them (blender-game-skills' `init_master.py`, `roundtrip.py`):
   run those headless only.
4. Name what you create with a prefix (`jle-`) and remove it when the job is done, so a rerun
   starts clean.

## Data API first, operators with an override

- Create and link with the data API: `bpy.data.objects.new(name, data)`,
  `collection.objects.link(ob)`, `bpy.data.meshes.remove(me)`.
- An operator acts on the context. Give it the context it needs:
  `with bpy.context.temp_override(active_object=ob, selected_objects=[ob], object=ob):`.
- Find what an import operator made by comparing `set(bpy.data.objects)` before and after
  (`tools/ai_level_builder.py --blender` does this).
- Iterate over a copy when the loop changes the list: `for m in list(ob.modifiers):`.

## Exporting a glTF of one object

`bpy.ops.export_scene.gltf` run under `temp_override(scene=<a scene of your own>)` crashed
Blender 5.2 in the background (an access violation in the depsgraph). Export from the context's
scene instead, in a Blender of your own (`blender -b`):

1. Link the object to a temporary collection of `bpy.context.scene`.
2. Select only that object.
3. Call `export_scene.gltf(filepath=..., export_format="GLB", use_selection=True,
   use_active_scene=True, export_apply=True)`.
4. Unlink the object and remove the collection.

Without `use_active_scene`, every scene is exported. Objects an import left selected in another
scene go into the file: a reference level's 16,000 pieces made a 10 MB file of a 268-triangle
model.

## bmesh

- From a mesh: `bm = bmesh.new()`, `bm.from_mesh(me)`, the edits, `bm.to_mesh(me)`, `bm.free()`,
  `me.update()`.
- In Edit Mode: `bmesh.from_edit_mesh(me)`, then `bmesh.update_edit_mesh(me)`. Do not free it.
- A BMVert, BMEdge or BMFace is dead after `free()`: copy what you need (`v.co.copy()`).
- There is no `bmesh.ops.boolean`: use a Boolean modifier and apply it.

## Evaluated meshes (modifiers applied)

`deg = bpy.context.evaluated_depsgraph_get()`, `ob_eval = ob.evaluated_get(deg)`,
`me = ob_eval.to_mesh()`, read it, then `ob_eval.to_mesh_clear()`. To keep the result as a mesh,
use `bpy.data.meshes.new_from_object(ob_eval)`.

## Memory and orphans

- A datablock with 0 users (`me.users == 0`) is dropped when the file is saved. Link it or
  remove it.
- Collection instances (`ob.instance_type = "COLLECTION"`, `ob.instance_collection = coll`)
  share their meshes: 23 instances of 3 prefabs still hold 36 meshes. Prefer them to
  `duplicate()` or `copy()`.

## Blender 5.2 API

- Actions are layered: `action.fcurves` is gone. Use `action.layers` and `action.slots`.
- A new material already has a node tree holding a Principled BSDF and a Material Output.

## Sources

Adapted from [ra100/blender-claude-plugin](https://github.com/ra100/blender-claude-plugin) at
`78e9151` (`blender-python-scripting`, `blender-modeling-modifiers`) and
[majidmanzarpour/blender-game-skills](https://github.com/majidmanzarpour/blender-game-skills) at
`f0ef293` (`blender-image-to-3d`), both MIT. They are not imported verbatim because their API notes
have errors in 5.2, among them `action.fcurves`, `bmesh.ops.boolean` and modifiers applied while
looping over the live list. Their image-to-asset process is in
[skill-style-match](../skill-style-match/SKILL.md).
