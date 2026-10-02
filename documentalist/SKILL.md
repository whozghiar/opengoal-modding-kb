---
name: documentalist
description: Documentation standards for this repository — English only, concise and tutorial-toned, no decorative icons, one canonical place per topic, and GOAL/Lisp code confined to the Lisp wiki. Load before writing, updating, or auditing any .md file.
---

# Documentalist — Documentation Standards Enforcement

This skill is the checklist an agent applies while writing or reviewing any
documentation file in this repository. It does not write documentation
content itself — it is the ruleset that keeps content written elsewhere
compliant, the same way `goal-lisp` informs mods without writing them.

Load this skill before creating a new `.md` file, editing an existing one,
or auditing the docs tree for compliance.

---

## 1. Scope

Applies to every `.md` file in the repository: `README.md`, `AGENTS.md`,
everything under `docs/`, the root README of each mod repository, and skill files under
`.agents/skills/`. A skill file being written is exempt from checking
itself mid-edit, but must satisfy every rule below once finished.
Imported third-party skills, which say so in their first line, are kept
verbatim from their source.

## 2. Language rule

English only, everywhere, whatever language the user speaks: each document
exists once, in English alone. A translated section, a bilingual title or
label, or a language switcher is a compliance failure: fold any fact it
carries into the English text, then delete it. Commit messages and PR text
follow the same rule (AGENTS.md, section 6).

## 3. Tone & conciseness

- Active voice, short paragraphs, one concept per section.
- Prefer a table or a list over a prose paragraph whenever there's a clear
  key-to-value or option-to-behavior mapping.
- Lead with a worked example over an abstract description wherever one is
  possible — this project's docs are tutorials for a working modder, not
  API reference prose.
- Avoid filler phrases that read as generated rather than written:
  "leverage", "delve into", "it's important to note that", "in order to",
  stacked bold text used as decoration rather than emphasis.
- No trailing summaries restating what a section already said.

## 4. Thematic separation

Every document states its scope in its first paragraph and stays inside
it. If a document starts accumulating a second, unrelated topic, split it
into its own file and link between them rather than growing one file
sideways.

## 5. No superfluous icons

Plain markdown headings and lists — no decorative emoji on headings,
bullets, or table cells. Keep GitHub's semantic admonitions
(`[!NOTE]`, `[!IMPORTANT]`, `[!TIP]`, `[!WARNING]`, `[!CAUTION]`) — those
are functional callouts, not decoration, and should be kept or added where
they genuinely flag something the reader must not skim past.

## 6. Code containment — the one rule with real teeth

GOAL/Lisp code (any fenced ` ```lisp ` block, or an inline snippet showing
GOAL syntax as a teaching example) may exist only in the Lisp wiki,
`goal-lisp/wiki/`: `common.md` for patterns shared by the three games,
`jak1.md`, `jak2.md` and `jak3.md` for each game's specifics.

Everywhere else in the repository — skills, guides, mod READMEs,
case-study docs — describe what the code does in prose and link to its
entry in the relevant wiki file. Two narrow exceptions:

- `docs/modding/templates/mod_menu.template.gc` and any other `.gc`
  template file: these are code artifacts meant to be copied, not
  narrative documentation, so the containment rule doesn't apply to them.
- A skill file may inline a short snippet if the point genuinely cannot be
  made in prose. Treat this as a last resort, not a default — first try
  describing the pattern and linking to the wiki's worked example.

Before adding any GOAL/Lisp code to a file other than the wiki, move it
into `goal-lisp/wiki/` (`common.md` if it applies to all three games,
the game's own file if it's specific to one) and replace it in the
original location with a link.

## 7. Verification rule

Every factual claim about the codebase — a task name, a file path, an API
name, a constant's value, a game-specific behavior — must be checked
against the actual repository before being written down. Never invent an
instruction, a symbol, or a task that "should" exist. If a per-game
comparison is being written (e.g. "this pattern also works in Jak 1"),
verify the referenced symbol actually exists in that game's `goal_src/`
before asserting it — the trilogy shares most engine code, but not all of
it, and some hooks exist in only one or two of the three games.

## 8. Pre-flight checklist

Before finishing any documentation edit, confirm:

- [ ] English only: one English version, no translated section, bilingual title or language switcher.
- [ ] No decorative icons — semantic admonitions only.
- [ ] The content lives in the right file for its theme, not bolted onto
      an unrelated one.
- [ ] No GOAL/Lisp code outside the wiki (or a `.gc` template).
- [ ] Every fact was checked against the actual repository, not assumed.
- [ ] Concise — no padding, no restated summary at the end.

## See also

- [`AGENTS.md`](../../../AGENTS.md) — the project-wide agent guide this skill's rules are drawn from (see its documentation standards section).
- [`goal-lisp/wiki/`](../goal-lisp/wiki/index.md) — where GOAL/Lisp code belongs.
