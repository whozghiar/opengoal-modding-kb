# OpenGOAL Modding Knowledge Base

Verified knowledge for modding the Jak & Daxter trilogy with OpenGOAL, packaged as
[Agent Skills](https://agentskills.io): one folder per skill, each with a `SKILL.md`. It is
mounted as a git submodule at `.agents/skills/` in the host `jak-project` repository and in
every mod repository created from it, so every mod reads and feeds the same copy. A fork of
`jak-project` can use this repository as is to read, or fork it too and point its submodule at
the fork to record its own discoveries (`git submodule set-url .agents/skills <fork URL>`).

| Skill | Use it for |
|---|---|
| [`goal-lisp`](goal-lisp/SKILL.md) | GOAL syntax, types, processes and states; the Lisp wiki lives in [`goal-lisp/wiki/`](goal-lisp/wiki/index.md) |
| [`engine-internals`](engine-internals/SKILL.md) | The C++ runtime, compiler, decompiler and build workflow |
| [`custom-actors-levels`](custom-actors-levels/SKILL.md) | Custom models, animations, sound banks, FR3 injection and custom levels |
| [`texture-modding`](texture-modding/SKILL.md) | Texture replacement, merging and texture packs |
| [`documentalist`](documentalist/SKILL.md) | Documentation standards for these repositories |
| [`kb`](kb/SKILL.md) | Recording a verified discovery here without duplicating or contradicting an entry |
| [`verification-before-completion`](verification-before-completion/SKILL.md) | Evidence before any "done" or "fixed" claim (imported, MIT) |
| [`writing-for-agents`](writing-for-agents/SKILL.md) | Writing skills and agent instruction files that agents follow reliably (imported, MIT) |

## Which agents read it

Gemini CLI, Codex, GitHub Copilot and Cursor read `.agents/skills/` natively. Claude Code
reads `.claude/skills/`, which `task ai-link` in the host repository fills with links to
these folders.

## Contributing

Follow the [`kb` skill](kb/SKILL.md). Imported skills are kept verbatim from their
source: update them by re-importing, not by editing.
