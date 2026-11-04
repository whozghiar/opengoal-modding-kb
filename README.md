# OpenGOAL Modding Knowledge Base

Verified knowledge for modding the Jak & Daxter trilogy with OpenGOAL, packaged as
[Agent Skills](https://agentskills.io): one folder per skill, each with a `SKILL.md`. It is
mounted as a git submodule at `.agents/skills/` in the host `jak-project` repository and in
every mod repository created from it, so every mod reads and feeds the same copy. A fork of
`jak-project` can use this repository as is to read, or fork it too and point its submodule at
the fork to record its own discoveries (`git submodule set-url .agents/skills <fork URL>`).

| Skill | Use it for |
|---|---|
| [`documentalist`](documentalist/SKILL.md) | Documentation standards for these repositories |
| [`kb`](kb/SKILL.md) | Recording a verified discovery here without duplicating or contradicting an entry |
| [`smoke-tests`](smoke-tests/SKILL.md) | In-game smoke tests of a mod: scenarios that drive the game, the mod's test notebook |
| [`writing-for-agents`](writing-for-agents/SKILL.md) | Writing skills and agent instruction files that agents follow reliably (imported, MIT) |
| [`remotion/`](remotion/remotion-best-practices/SKILL.md) | Making videos with Remotion, one skill per folder, `remotion-best-practices` routing to the others (imported from [remotion-dev/skills](https://github.com/remotion-dev/skills), v4.0.537) |
| [`jak-level-editor-skills/`](jak-level-editor-skills/README.md) | Working on [Jak Level Editor](https://github.com/whozghiar/jak-level-editor): building and testing it, its interface texts, its game data, its screenshots, building levels through its MCP tools, adding an exported level to the game |

The Lisp wiki, [`goal-lisp/wiki/`](goal-lisp/wiki/index.md), is not a skill: it is the verified
reference every skill and mod points to. Besides its GOAL pages (common, then one per game), it
holds background pages on GOAL's concepts, the engine and build, the REPL, custom assets and
textures.

## Which agents read it

Gemini CLI, Codex, GitHub Copilot and Cursor read `.agents/skills/` natively. Claude Code
reads `.claude/skills/`, which `task ai-link` in the host repository fills with links to
these folders (`yarn skills:link` in the OpenGOAL launcher and in Jak Level Editor, which also link the skills
grouped one folder down: `remotion/`, `jak-level-editor-skills/`).

## Contributing

Follow the [`kb` skill](kb/SKILL.md). Imported skills are kept verbatim from their
source: update them by re-importing, not by editing.
