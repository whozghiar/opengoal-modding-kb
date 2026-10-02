---
name: custom-actors-levels
description: "Comprehensive guide to custom entities, 3D asset pipelines, adapting/backporting Blender 3D models between Jak games (Jak 1, 2, 3), Blender glTF/GLB exports, armatures, joint channels, non-destructive texture injection, custom sound banks (SBK), Merc .FR3 injection, and custom levels."
---

# Custom Actors, Entities, 3D Assets, Rig Adapter & Levels

OpenGOAL allows importing custom 3D models, skeletal meshes, custom
animations, dedicated sound effect banks (SBK), adapting models across
games, and building entire custom levels into Jak 1, Jak 2, and Jak 3. This
skill covers the 4-layer entity pipeline, cross-game model adaptation and
retargeting, non-destructive texture workflows, the two-circuit rendering
model, Blender glTF/GLB export rules, audio bank packaging, and level
creation.

GOAL/Lisp code examples belong in the Lisp wiki, not here — this skill
explains the pipeline and points to the exact call syntax in
[`goal-lisp/wiki/`](../goal-lisp/wiki/index.md).

---

## 1. Role & scope

As a 3D technical specialist and entity engineer for OpenGOAL:
- Adapt character, vehicle, and enemy models between installments of the
  Jak trilogy (e.g. backporting a model from Jak 3 or Jak 1 to Jak 2),
  manage articulation skeletons, retarget armatures, apply non-destructive
  textures without breaking meshes or vanilla animations, and engineer full
  custom actors and levels.
- Use Blender tooling (`mcp:blender` when active) alongside file inspection
  tools to check joint hierarchies, vertex weights, material slots, and
  export flags.

The hardest case this pipeline handles is a **grafted** entity — one that
attaches to the player process, swaps Jak's skeleton, and drives animation
in real time (the Jak 3-to-Jak-1 Jetboard backport is the reference example
this pipeline was built against). Most imports are much smaller: a pickup,
a projectile visual, a HUD icon, a particle effect, or a handful of sound
effects. Section 10 below is the lightweight path for those — no Blender,
no skeleton, no full decompiler run required.

---

## 2. Critical engine rules & guardrails

1. **Strict armature hierarchy and joint channels.** The GOAL engine expects
   a strict joint hierarchy. Never rename, delete, or add free bones in the
   main tree without adapting the joint table of the target GOAL actor. Any
   model adapted from one Jak game to another must be re-parented/retargeted
   to exactly match the armature expected by the destination `defskelgroup`.
   Custom `.glb` files for characters must contain the armature.
   - **Root bone (`align` rule):** joint 0 must be located at `(0, 0, 0)`
     and named `align`. Omitting `align` as joint 0 offsets every bone
     index by 1, causing immediate mesh dislocation or engine crashes when
     applying animations. In the Jak games, `align`'s per-frame delta is
     not purely visual — the engine adds it to the character's velocity and
     rotation, so it drives physical movement too.
2. **Cross-game model adaptation & retargeting (Jak 1 <-> Jak 2 <-> Jak 3).**
   When porting or backporting a model, verify bone counts and joint order
   against the target game's skeletal definition. Use the `retarget_anim`
   tool (`cmake --build out/build/Release --target retarget_anim --config Release`)
   or the GUI Reskin Tool when adapting joint matrices. Target `.glb` files
   go under `custom_assets/<target_game>/models/<area_name>/<model_name>.glb`.
3. **Non-destructive textures & UV management.** Never modify base
   decompiled archives or retail textures directly — place custom PNGs
   under `custom_assets/<target_game>/texture_replacements/<texture_folder>/`.
   Preserve existing UV coordinates when adding a texture, or create a new,
   clean material group without overwriting original UV maps. After
   modifying an asset or texture, run `task extract` to bake it.
4. **Export format & transformations.** Export all meshes and animations as
   binary glTF (`.glb`) without data loss — include vertex weights, normals,
   and UVs. Apply all base transformations (scale `1.0`, rotation
   `(0, 0, 0)`) before exporting.
5. **Traceability.** Record each modified asset, armature, or added texture
   in the current mod's Tier-2 technical README
   (`docs/modding/current_mod/<mod_slug>_readme.md`) — describe the change
   and link to the wiki section for its exact syntax; do not paste the GOAL
   code itself into that README.
6. **Clean mod isolation.** Keep a mod's own files in a dedicated
   subdirectory (e.g. `engine/target/<feature>/`). Prefer a dedicated
   overrides file loaded last that redefines whole functions over patching
   vanilla files inline. Tag any unavoidable edit to a shared vanilla file
   with a short marker comment naming the mod and what changed. Prefix
   every mod-owned state, variable, and art-group name with the mod slug to
   avoid collisions with other mod branches.

---

## 3. The 4-layer entity ingestion model

Every external asset traverses four architectural layers:

```
LAYER 1 -- C++ Tooling (goalc, decompiler, common)
  build-actor: converts glTF (.glb) into an engine art-group (.go)
  build-sbk: converts WAV audio files into a sound bank (.SBK)
  decompiler: extracts retail game models & sounds into glTF/WAV

LAYER 2 -- Asset Pipeline (GOOS / game.gp)
  (build-actor ...) describes 3D models, bones, and animation maps
  (build-sbk ...) compiles custom sound banks
  Asset locations: custom_assets/jak[x]/models/ and sounds/sfx/

LAYER 3 -- Container Packaging (DGO / CGO / ISO)
  game.gd / <level>.gd: packs art-groups (.go) into GAME.CGO/DGO
  the sound-bank iso group in game.gp: packages banks for Overlord

LAYER 4 -- GOAL Code (goal_src/jak[x]/...)
  deftype & defstate: state machine, physics, and behavior hooks
  animation and sound-play calls: see the per-game Lisp wiki
```

Everything that produces or parses a binary format (a model, an animation,
a sound) is C++ in `goalc`/`decompiler`/`common`. Everything that
orchestrates the build (what to build, in what order) is GOOS in `game.gp`.
Everything that is gameplay logic is GOAL in `goal_src/`.

---

## 4. The 2-circuit architecture for skeletal actors

Every rendered actor depends on two independent pipelines:

```
CIRCUIT 1 -- GOAL Heap (Game Logic & Skeletons)
  Source: target_level.gd / all_objs.json -> <model>-ag.go
  Loaded into: RAM by the GOAL heap loader
  Contains: joint hierarchy (*-jg), animations (*-ja), bone matrices
  Looked up by: art-group-get-by-name & initialize-skeleton
  Failure symptom: crash with process-drawable-art-error

CIRCUIT 2 -- PC Merc2 Renderer (3D Geometry & Textures)
  Source: decompiler/config/jak[x]/*.jsonc -> extra_art_groups_by_dgo
  Baked into: out/jak[x]/fr3/<level>.fr3 via 'task extract'
  Loaded into: GPU VRAM by the PC renderer (Merc2)
  Contains: 3D mesh vertices (*-mg), UVs, material assignments
  Failure symptom: model is completely invisible (silent skip)
```

Golden rule: loading an art group into GOAL memory (Circuit 1) makes the
actor logically exist, but it stays completely invisible unless its
geometry was baked into the active level's `.fr3` file (Circuit 2) via
`extra_art_groups_by_dgo` during `task extract`.

A texture baked this way (Circuit 2's `TexturePool`) is a **separate**
system from the classic `texture-page`/`*texture-page-dir*` lookup that
`hud-sprite`/`def-tex` need — a custom-baked texture never bridges into
that second system unless a real `texture-page` object uploads to VRAM. See
[§10.2](#102-texture-reachability-two-disjoint-pipelines) before wiring any
custom texture to a 2D/HUD element.

---

## 5. Merc `.fr3` injection (no-borrow workflow)

Injecting an existing or custom model's geometry into a level without
burning scarce level-borrow slots:

1. Find the model's art-group name in `goal_src/jak[x]/build/all_objs.json`
   (e.g. `"transport-ag"`) and its home DGO — the level that originally
   shipped it, needed to resolve its textures correctly.
2. Declare the pairing in `decompiler/config/jak[x]/jak[x]_config.jsonc`
   under `extra_art_groups_by_dgo`, mapping the target DGO to
   `"<art-group-name>:<HOME.DGO>"`. Omitting `:<HOME.DGO>` lets the
   decompiler pick the first level containing the art group, which may lack
   the right texture page.
3. Rebuild the decompiler and re-extract: `task build-release-decomp` then
   `task extract`.
4. Add the art group's `.go` (and its texture page `.go`) to the target
   level's `.gd` list, **before** the level's own `.go` entry (the BSP file
   must stay last).
5. A dynamically-spawned actor has no pre-placed BSP entity, so
   `skeleton-group->draw-control` cannot resolve one on its own — bind the
   process to a resident entity before calling `initialize-skeleton`. The
   exact binding calls (`ctywide-entity-hack` for Haven City,
   `process-entity-set!` elsewhere) and a troubleshooting table for
   invisible/white/zone-limited models are in
   [3.5 of the Lisp wiki](../goal-lisp/wiki/jak2.md#35-merc-geometry--fr3-residency).

---

## 6. Blender workflow for custom models (.glb)

Custom models must be exported from Blender as binary glTF (`.glb`).

### Armature & skeleton rules
- Joint 0 is `align` at `(0, 0, 0)` — see [§2](#2-critical-engine-rules--guardrails).
- Joints must match the bone index expectation of the target game. If
  replacing or extending an existing actor (Jak, a Crimson Guard), bone
  count, naming, and orientation must strictly match the original skeleton.
- Skeletal animations are compressed into joint channels (translation,
  quaternion rotation, scale) — apply scale and rotation in Blender before
  exporting (`Ctrl+A` -> Apply All Transforms).
- Blender units must match OpenGOAL world scale (1 Blender meter = 1 game
  meter).

### Required Blender plugins & standalone tools
- Blender add-ons under `custom_assets/blender_plugins/`: `opengoal.py`
  (mesh tools, surface collision flags, vertex-colour bake helpers) and
  `gltf2_blender_extract.py` (a custom exporter drop-in, placed inside
  Blender's `scripts/addons/io_scene_gltf2/blender/exp/`, needed for
  correct joint layout and vertex-colour export for `build-actor`).
- Standalone tools built via CMake targets: `retarget_anim`, `build_actor`,
  `build_sbk` (`cmake --build out/build/Release --target <name> --config Release`).

### Export settings (Blender glTF 2.0 exporter)
- Format: glTF Binary (`.glb`).
- Include: limit to selected objects, Armatures + Meshes. Character `.glb`
  files must contain the armature.
- Transform: `+Y Up` (standard glTF orientation).
- Geometry: apply modifiers, include normals, include UVs, include vertex
  colors if used.
- Animation: enabled if exporting animations, using NLA strips or Actions.

---

## 7. Animation mapping & master art groups

When grafting custom animations onto an existing character (Jak or
Daxter), `build-actor` takes a `:master-art-group` (the parent art group
name) and `:master-ag-map` (an explicit animation-name-to-slot-index table)
so the custom clips land in unused slots of the character's master art
group without recompiling its hundreds of native animations. In `game.gd`,
place the custom art group immediately after the parent art group so its
master-art-group indices stay coherent.

An art group built with `:master-art-group` needs an explicit registration
call at file load — without it, the engine's native "does this need
linking" check returns false and the animations never attach at level
load. The registration and the actual linking hook are documented in
[1.2.8 of the Lisp wiki](../goal-lisp/wiki/common.md#128-custom-art-groups--dynamic-animation-linking-link-art)
(Jak 1 has the same hook; Jak 3's engine code does not carry it — see that
game's wiki file for the alternative).

---

## 8. Custom sound banks (SBK audio pipeline)

Adding dedicated sound effects for a custom entity:

### Fast source extraction (porting sounds from another game)
When the WAV source lives in another game's retail `.SBK` banks, don't run
the full decompiler pass just to get audio — the decompiler's normal sound
extraction only runs as a side effect of loading every DGO of the source
game first (30+ minutes on a large game) and aborts the whole run if any
single bank is malformed. Use the standalone `extract_sbk` tool instead
(`cmake --build out/build/Release --target extract_sbk --config Release`,
then run it against `<source-game>/iso_data/SBK` with an output folder and
an optional `--banks NAME1,NAME2` filter) — it calls the same decoder
directly on a folder or file and never aborts on one bad bank. Its output
layout matches the decompiler's and is directly consumable by the pipeline
below. Sound names resolve case-insensitively with `-`/`_` folded at
runtime, so a retail name like `RED2_SHOT` plays from GOAL as
`"red2-shot"`.

### Source audio files
Place 16-bit 48kHz PCM WAV audio + a `metadata.txt` manifest under
`custom_assets/jak[x]/sounds/sfx/<BANK_NAME>/`. Generate a valid
`metadata.txt` layout by extracting an existing bank first, then edit it —
it is the human-readable source of truth for volume, pitch, and playback
flags per sound.

### Delivery routes
- **Route A — append to `COMMON`** (recommended for global sounds): merges
  your sounds directly into the resident `COMMON.SBK` with zero runtime
  management; play immediately anywhere. Requires removing `"COMMON"` from
  `copy-sbk-files` in `game.gp` first to avoid a duplicate-output build
  error.
- **Route B — a standalone bank:** placed into Overlord's 3-slot rotating
  level pool when loaded; can conflict with level sound banks if the pool
  is oversubscribed. Prefer Route A unless the bank is genuinely
  level-scoped.

The exact `append-sbk`/`build-sbk` call syntax is in
[1.2.12 of the Lisp wiki](../goal-lisp/wiki/common.md#1212-static-props-custom-levels-and-audio-banks).

### Looping sounds
For looped or frame-updated sounds, pre-allocate the sound ID once in the
actor's `-init` state, never inside the frame loop.

---

## 9. Custom level creation

Custom levels are built from `.glb` environment models and configuration
files.

### Directory structure
```
custom_assets/jak[x]/levels/<level_name>/
  <level_name>.jsonc     level properties, lighting, actor spawns
  <level_name>.gd        DGO package descriptor
  <level_name>.glb        3D environment mesh and collision, exported from Blender
```

### Build & registration
1. Declare the custom level build targets in `goal_src/jak[x]/game.gp`
   — the exact macro calls are in
   [1.2.12 of the Lisp wiki](../goal-lisp/wiki/common.md#1212-static-props-custom-levels-and-audio-banks).
2. Add level metadata (symbol, name string, bounding box, loading
   conditions, music/ambient sound) in
   `goal_src/jak[x]/engine/level/level-info.gc`.
3. Recompile (`task repl`, then hot-reload), boot the game, and warp to the
   custom level from the REPL using the call in the same wiki section.

Working example to copy from: `custom_assets/jak1/levels/test-zone/`. Note
that custom levels are still an early feature — for now, all meshes are
treated as ground collision, so walls without a dedicated "wall" material
produce buggy collision.

---

## 10. Cross-game object & sound bridge (lightweight imports)

Sections 6-9 cover the full grafted skeletal actor case (a
character-swapping entity like a vehicle or a worn item). Most cross-game
imports are much smaller — a single pickup, a projectile visual, a
HUD-space icon, a particle effect, or a handful of sound effects. This
section is the lightweight path for exactly that.

### 10.1 Lightweight static objects — no skin, no Blender required
`build-actor`'s tooling auto-synthesizes a trivial joint chain whenever the
source `.glb` carries no skin at all: a 3-joint `align`/`prejoint`/`main`
hierarchy plus a single-pose idle animation — the same fallback vanilla's
own simple static props use. A from-scratch, hand-authored, unanimated mesh
(even one with no armature) is a fully valid `build-actor` input.

For the common single-LOD, single-pose case, a one-call macro
(`def-actor`, present in every game's `engine/data/art-h.gc`) expands to the
full art-group registration boilerplate — see
[1.2.12 of the Lisp wiki](../goal-lisp/wiki/common.md#1212-static-props-custom-levels-and-audio-banks)
for the exact call. Reach for the manual multi-line registration only when
you need multiple LODs, several named animations, or non-default naming.

Where the `.glb` must live depends on where the object needs to render:
- To use it **inside existing (vanilla) levels**, `task extract`'s level
  baker only picks up textures/geometry from
  `custom_assets/jak[x]/models/common/` (the pseudo-level shared by every
  level). Place a copy there, named after the merc geometry
  (`<ag>-lod0.glb`, not the art-group name) — `build-actor` and
  `task extract` use different naming conventions for the same file, and a
  mismatch leaves the model invisible with no error.
- To ship it **only inside a purpose-built custom level**, a single copy
  under `models/custom_levels/` is enough (see §9).
- Either way, the `build-actor` declaration in `game.gp` always reads its
  source `.glb` from `models/custom_levels/`, regardless of where the baked
  runtime copy lives.

### 10.2 Texture reachability: two disjoint pipelines
Before wiring a custom-baked texture to any 2D/UI element (`hud-sprite`,
`def-tex`), confirm it can actually be reached — this is the single most
common reason a working, log-verified `task extract` bake produces zero
visible change in game.

There are two independent texture systems with only one narrow bridge
between them:

| | Classic GOAL (`texture-page`) | PC-port `TexturePool` |
|---|---|---|
| Populated by | Loading a real, retail-decompiled `tpage` object | `build-actor`/`task extract` baking `.glb` textures into the level's texture pool |
| Read by | `get-texture`/`def-tex` and runtime name lookups — what any `hud-sprite`'s texture field needs | Merc2's foreground renderer, via a raw texture ID baked into the model at build time |
| Custom-model textures reachable? | No, never — a custom model has no `texture-page` object at all | Yes — this is the pool `build-actor`/`task extract` writes into |

The only bridge intercepts the VIF packet emitted when a *real*
`texture-page` uploads to VRAM and matches it against the pool by name. A
custom model never emits that upload, so it never crosses the bridge — no
`texture-page`-based lookup, static or dynamic, will ever find it.

Decision rule:
- Need a custom-baked texture on-screen as a 2D HUD element? Draw it as a
  real, small, spawned 3D/merc model instead of a sprite — it reads the
  `TexturePool` directly, the same mechanism the rest of Merc2 already
  uses. Jak 2 ships a ready pattern for exactly this: `hud-create-icon`
  (`engine/ui/hud.gc`) spawns a process holding a named `defskelgroup`
  art-group, screen-locks it, and renders it through the normal Merc2 path.
- Need to repaint an *existing* retail texture instead? Use the
  `texture_replacements`/`texture_merges` pipeline (see the
  `texture-modding` skill) — it repaints a texture that already has a real
  `texture-page`, so it reaches the classic lookup for free.
- A "dynamic" runtime name lookup is not a fix for a custom-baked texture —
  it only bypasses the compile-time-symbol problem, not the reachability
  problem above.

### 10.3 Porting gameplay & particle code across games
When hand-porting state or particle-effect GOAL code from one game's
decompiled source into another: bitfield enums keep stable bit positions
across games but the decompiler assigns different names to the same bit
per game, so translate by position, never by name. Particle group/launcher
IDs are index slots sized by an explicit constant — compute the next free
slot from the destination table's own length, never reuse the source
game's raw numeric IDs. A field present on the source game's version of a
vanilla type may not exist on the destination's version of that type — add
it to the mod's own subtype, never widen a shared vanilla type's layout. A
referenced particle ID from the source may simply not exist in the
destination's tables — drop that reference rather than guessing a
substitute. Check for a same-purpose primitive already in the destination
engine before concluding a feature can't be ported — a newer game
frequently has helpers an older one lacks, and vice versa. Keep an explicit
source-name-to-destination-name substitution table next to a port script
for any shared texture whose content differs between games, so a later
pass can tell a deliberate substitution from a missing texture.

The GOAL-syntax-level traps this uncovers most often (untyped stack arrays,
`matrix` row-field assumptions, bit-flag position vs. name) are documented
with code examples in each game's Lisp wiki, "Known pitfalls" section.

### 10.4 Hand-authoring minimal `.glb` assets without Blender
For simple textured primitives — icons, markers, placeholder geometry — a
`.glb` can be hand-written in pure Python with no Blender step at all, as
long as it satisfies what the extractor's custom-model path actually reads:
one `POSITION` (VEC3 float) + `NORMAL` (VEC3 float — required; omitting it
doesn't error at export time but crashes `task extract` with an
out-of-range read) + optional `TEXCOORD_0` (VEC2 float) per
primitive/material. No skin, no joint weights, no vertex colors needed.
Each material needs a base-color texture whose glTF `name` field (not its
filename) becomes the runtime texture's debug name — set it to whatever
string later code will look the texture up by. Source images must already
be 8-bit RGBA; a `.glb`'s embedded images can be plain base64 data URIs in
the JSON chunk, with only vertex/index data in the binary chunk.

This is strictly for simple, unskinned/unanimated content — a real
character, vehicle, or anything needing a joint hierarchy still goes
through the Blender workflow in §6.

---

## See also

- [`goal-lisp/wiki/`](../goal-lisp/wiki/index.md) — every GOAL code example this skill references.
- [`texture-modding`](../texture-modding/SKILL.md) — the texture replacement/merge pipeline for repainting existing retail textures.
- [`docs/modding/guides/mods_menu.md`](../../../docs/modding/guides/mods_menu.md) — required in-game toggle for any new mod, including ones built with this skill.
