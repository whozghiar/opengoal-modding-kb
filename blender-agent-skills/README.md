# Blender agent skills (imported)

Imported verbatim from [EL4CTEO/blender-skills](https://github.com/EL4CTEO/blender-skills/tree/fb92cfb/skills)
(release 1.2.0, MIT, see `LICENSE`): 28 skills for driving Blender 5.2 from an agent, one folder
each, `blender-core` routing to the others. Update by re-importing the `skills/` folder of a newer
release, not by editing.

Their advice is generic: physically based materials, bevels, Cycles and EEVEE renders, engines such
as Unity, Unreal, Godot and Roblox. For geometry meant for the game, the game's own rules come
first: [`skill-zone-design`](../jak-level-editor-skills/skill-zone-design/SKILL.md) and
[`skill-style-match`](../jak-level-editor-skills/skill-style-match/SKILL.md): the game's pieces
and textures, one Base Color image per material (no roughness, metal or normal maps), the game's
triangle budget, the game's light rather than Blender's.

The ones that serve the game's models:

| Skill | For |
|---|---|
| `blender-core` | running bpy through the Blender MCP server or headless, the 5.x API changes |
| `blender-asset-qa` | the review gate after making a mesh: normals, open or floating parts, z-fighting, review shots |
| `blender-game-optimization` | triangle budgets, decimation, cleanup |
| `blender-retopology-uv` | seams, unwrapping, texel density |
| `blender-addon-development` | Blender add-ons as extensions (manifest, operators, panels) |
| `blender-pipeline-automation` | headless batches over folders of files |
