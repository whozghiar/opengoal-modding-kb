# Jak Level Editor skills

The skills of [Jak Level Editor](https://github.com/whozghiar/jak-level-editor), the level editor
for the Jak and Daxter games (Jak 3 today), one folder per skill. Their paths (`crates/`, `ui/`,
`lang/`, `docs/`, `tools/`...) are those of the editor's repository, which mounts this knowledge
base at `.agents/skills/` like every other host; `yarn skills:link` there links them, with the
others, into `.claude/skills/`.

| Skill | Use it for |
|---|---|
| [`build-and-test`](build-and-test/SKILL.md) | Building the editor (Rust, Tauri, Svelte) and running its tests |
| [`interface-texts`](interface-texts/SKILL.md) | Adding or changing a text of the interface, in every language |
| [`game-data`](game-data/SKILL.md) | Making again what the editor knows of the game (`data/`), extracting a disc |
| [`doc-captures`](doc-captures/SKILL.md) | Taking the documentation's screenshots again |
| [`level-building-mcp`](level-building-mcp/SKILL.md) | Building or changing a level through the editor's MCP tools |
| [`prefab-recipes`](prefab-recipes/SKILL.md) | The recipe of every prefab made, changed or deleted (mandatory: users get the recipe, never the `.glb`) |
| [`play-in-game`](play-in-game/SKILL.md) | Adding an exported level to a jak-project by hand and checking it in the game |
| [`skill-blender-context`](skill-blender-context/SKILL.md) | Running bpy code in Blender through its MCP server without touching the user's file |
| [`skill-modular-snapping`](skill-modular-snapping/SKILL.md) | Modules, grid and pivots; laying out rooms with `tools/ai_level_builder.py` |
| [`skill-style-match`](skill-style-match/SKILL.md) | Giving new geometry, the editor's or Blender's, a level's textures and light |
| [`skill-zone-design`](skill-zone-design/SKILL.md) | Designing a structure in the style of a zone of Jak 1, Jak II or Jak 3: one entry per zone |
| [`skill-image-to-building`](skill-image-to-building/SKILL.md) | Reproducing a picture in a level: the level's own buildings placed as variants, Blender for what it lacks |
