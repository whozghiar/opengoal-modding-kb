---
name: texture-modding
description: Guide to custom texture replacement, directory layout, PNG format requirements, texture merging, and task extract baking in OpenGOAL.
---

# Texture Modding & Replacement

OpenGOAL features native texture replacement and texture merging
mechanisms. Custom textures are processed during asset extraction and baked
directly into PC-optimized renderer packages (`.fr3` and texture
databases).

---

## 1. Texture replacement directory structure

Texture replacements are located under `custom_assets/`:

```
custom_assets/
  jak[x]/
    texture_replacements/
      <tpage-name>/
        <texture-name>.png      replaces the texture in that one tpage
      _all/
        <texture-name>.png      global fallback: replaces the texture across ALL tpages
    texture_merges/
      <tpage-name>/
        <texture-name>.png      merges non-transparent pixels onto the original texture
```

- **Specific tpage folder (`<tpage-name>/`):** only replaces the texture
  inside the designated texture page, e.g.
  `custom_assets/jak2/texture_replacements/tpage-1234/guard-armor.png`.
- **Global fallback folder (`_all/`):** if a texture isn't found in a
  specific `<tpage-name>` subfolder, the extractor checks `_all/` — useful
  for a texture shared across multiple levels, e.g.
  `custom_assets/jak2/texture_replacements/_all/jak-eyes.png`.
- Use `_all/` whenever a texture is genuinely shared: over a third of this
  project's retail textures are shared across multiple levels. Check
  `decompiler/config/<game>/ntsc_v1/tex-info.min.json` (or the
  `open-goal-texture-pack-generator` tool) to detect sharing conflicts
  before packaging a pack.

## 2. Image specifications

- Format: standard PNG.
- Color channels: 32-bit RGBA (8 bits per channel).
- Resolution: can match the original PS2 dimensions or be higher (HD
  textures).
- For texture merging (`texture_merges`), the merge PNG's dimensions must
  strictly match the source texture's dimensions.

## 3. Extraction & baking workflow

Textures are not loaded as loose `.png` files at runtime — they are baked
into binary texture pages and `.fr3` level files during offline extraction.

1. Identify the texture name and tpage (from `decompiler_out/jak[x]/textures/`
   or level files).
2. Place your edited `.png` under
   `custom_assets/jak[x]/texture_replacements/<tpage-name>/<texture-name>.png`
   (or `_all/<texture-name>.png`).
3. Run `task extract` to bake the textures.
4. Run `task boot-game` to inspect them in-game.

> [!NOTE]
> Editing or adding a PNG in `custom_assets/` does not update the game
> immediately. You must run `task extract` to bake the change.

## 4. Standalone texture pack distribution & packaging

To distribute custom textures as an official, one-click-installable
texture pack for the OpenGOAL Launcher:

1. **Package interactively:** open the
   [OpenGOAL Texture Pack Generator](https://github.com/whozghiar/open-goal-texture-pack-generator)
   desktop tool (separate repository). Select your texture replacements, fill in display
   name, author, version, and description, then export the `.zip` archive
   into `docs/modding/current_mod/texture_packs/<slug>-v<version>.zip`.
2. **Register the archive:** run `task modding-package-texture-pack` (alias
   `task modding-register-texture-pack`) to inspect the generated `.zip`,
   read its internal `metadata.json`, and register it in `index.json`.
   `--zip <path>` registers a specific archive directly; `--from-source`
   forces compiling directly from raw `custom_assets/<game>/texture_replacements/`
   PNGs instead of reading a GUI-exported archive.

The resulting `.zip` must have this exact structure (the GUI generator
guarantees it):

```
metadata.json     root metadata (author, version, description, tags)
cover.png         optional thumbnail
custom_assets/jak[x]/texture_replacements/   the replacement textures
```

`metadata.json` needs `author`/`authors` and `releaseDate`/`publishedDate`
at minimum. If the pack is releasing alongside a specific mod, add the
mod's slug to `metadata.json`'s `tags` and point `websiteUrl` at the mod
branch — this is what the launcher catalog schema uses to associate a
texture pack with its mod.

Archives in `docs/modding/current_mod/texture_packs/` are git-tracked (only
`custom_assets/jak*/texture_replacements/*` itself is gitignored). During
`.github/workflows/release.yml`, CI automatically picks up any `.zip` from
that directory, computes its SHA256 checksum, registers it in `index.json`,
and attaches it to the GitHub Release assets.

---

## See also

- [`custom-actors-levels`](../custom-actors-levels/SKILL.md) — the wider asset pipeline this texture workflow is part of.
- [`docs/modding/guides/mod_distribution_guide.md`](../../../docs/modding/guides/mod_distribution_guide.md) — the full release packaging and launcher catalog pipeline.
