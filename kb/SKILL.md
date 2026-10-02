---
name: kb
description: Record or correct a verified OpenGOAL modding fact (GOAL pattern, language trap, engine behavior, crash cause, asset-pipeline quirk) in the shared knowledge base. Use once a fact is verified and would help another mod; notes about the current mod alone stay in that mod's docs.
---

# Recording knowledge in the shared knowledge base

The folder this skill lives in is the knowledge base: the repository
[`whozghiar/opengoal-modding-kb`](https://github.com/whozghiar/opengoal-modding-kb),
mounted as a git submodule at `.agents/skills/` in the host `jak-project` repository and in
every mod repository created from it. Each fact has one home, shared by every mod.

## What belongs here

| Here | Elsewhere |
|---|---|
| A GOAL pattern, macro or language trap | What a mod changed and why: the change log of that mod's `docs/modding/current_mod/<slug>_readme.md` |
| An engine behavior several mods can hit (memory, DGOs, processes, traffic, art groups) | A mod's design notes: `docs/modding/current_mod/<slug>_readme.md` in that mod |
| The cause and fix of a crash another mod could reproduce | A hypothesis: nowhere until it is verified |

A fact is verified when it compiled (`task compile-check`), when the user observed it in
game, or when it was read in the actual `goal_src/` of the game it is attributed to.

## Procedure

1. **Find the topic.** Search before writing:
   `grep -rniE "<two or three keywords>" .agents/skills`. For GOAL facts, start from
   [`goal-lisp/wiki/index.md`](../goal-lisp/wiki/index.md).
2. **Pick its one home.** Shared by the three games: `goal-lisp/wiki/common.md`. One
   game: `goal-lisp/wiki/jak1.md`, `jak2.md` or `jak3.md`. Asset pipeline: the owning
   skill (`custom-actors-levels`, `texture-modding`, `engine-internals`).
3. **Edit in place.** When the topic exists, update its entry. When the new fact
   contradicts it, replace the old statement: one version only. Add an entry only when
   nothing covers the topic, and list it in `goal-lisp/wiki/index.md` if it is a wiki
   section.
4. **Cite the evidence** at the end of the entry:
   `Verified: <game>, goal_src/<game>/<path>.gc, <date>.`
5. **Commit and push from the submodule.** A submodule is checked out without a branch,
   so switch to `main` first or the commit is lost:

   ```bash
   git -C .agents/skills switch main
   git -C .agents/skills pull --ff-only
   # edit the files
   git -C .agents/skills add -A
   git -C .agents/skills commit -m "<game>: <topic>"
   git -C .agents/skills push
   ```

   Pushing needs write access to the knowledge-base repository named in `.gitmodules`.
   In a fork without it, fork the knowledge base, point the submodule at the fork
   (`git submodule set-url .agents/skills <fork URL>`, then commit `.gitmodules`), and
   push there.

   Say in the commit message when it replaces a previous statement. The git history of
   this repository is the changelog, so no file keeps a "recent discoveries" section.

Every host repository refreshes its copy at the start of each Claude Code session;
`task kb-update` does the same by hand.
