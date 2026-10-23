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
| [`play-in-game`](play-in-game/SKILL.md) | Adding an exported level to a jak-project by hand and checking it in the game |
