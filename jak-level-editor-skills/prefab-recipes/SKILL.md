---
name: prefab-recipes
description: Write, update or check the recipe of a Jak Level Editor prefab (data/prefabs/). Use every time a prefab .glb of library/prefabricated/ is made, changed, renamed or deleted, whatever made it (a script of tools/level_prefabs/, Blender, the MCP, by hand).
---

# Prefab recipes

Users never receive a prefab's `.glb`: it is made of the game's meshes and textures. They receive
its **recipe**, `data/prefabs/<its path under library/prefabricated/>.json`, and the editor bakes
the `.glb` from their own extracted levels (`docs/architecture.md`, "Prefab recipes"). A prefab
without a recipe exists on your machine only.

## The rule: a prefab is done when its recipe is

After making, changing or renaming any prefab (the game's, a custom one, the user's own), from the
editor's repository root:

1. `cargo build --release -p jak-level-editor-core --bin prefab-recipes` (once per session, and
   after any change to `crates/core/src/io/recipes.rs`).
2. `./target/release/prefab-recipes update <part of its path>...`: done when every prefab prints
   `ok` and the last line says `0 prefab(s) without a recipe`. Without a filter it remakes all of
   them (a few minutes): do that after changing the tool.
3. `./target/release/prefab-recipes audit <same filters>`: done when it prints
   `0 element(s) with the game's shapes`.
4. `./target/release/prefab-recipes check`: done on
   `0 prefab(s) without a recipe, 0 recipe(s) without a prefab`. For a prefab deleted, delete its
   recipe.
5. Commit the recipes (`data/prefabs/**/*.json`) and the new textures
   (`data/prefabs/textures/*.png`) with the change that made the prefabs.

## What a recipe can tell

`update` writes a recipe only when baking it gives the prefab back: the same nodes, triangles,
texture names and lighting. It finds:

- a game piece by its node name, then by its triangles: moved, turned (joined into one mesh in
  Blender), cropped, given thickness by `tools/level_prefabs/blender_fill.py` thin(), its colors
  made neutral or lit, its textures swapped or renamed `jak2-<name>`;
- the collision's boxes and copies of the drawn pieces' triangles;
- anything else as the prefab's own triangles, with the game's textures by name.

A scale applied to a piece's mesh data hides it: keep the scale on the node. A texture made for
the editor's prefabs is named `custom-<name>` (or `jle-<name>`): it becomes a file of
`data/prefabs/textures/`; a texture of the game is always used under its game name.

## When it fails

| Message | Cause | Fix |
|---|---|---|
| `named as pieces of the game are not made of their triangles` | a game piece whose vertices were edited | keep the piece whole and transform its node, or model the change as a separate element |
| `the texture ... is in none of the levels` | a texture of the prefab's own, not named as the editor's | name it `custom-<name>`, or use the game's texture by its name |
| `baked from its recipe it differs` | an operation the recipe cannot tell (a modifier other than thin()'s Solidify, a scale applied to a mesh) | redo the operation as a node transform or as thin() |
| `audit`: `GAME <recipe>: <element>` | the game's geometry left as the prefab's own (a piece scaled, or edited past recognition) | the same fixes: a prefab ships once `audit` no longer lists it |
