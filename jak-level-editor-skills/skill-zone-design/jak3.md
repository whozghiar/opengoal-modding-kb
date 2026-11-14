# Jak 3: its zones

Two worlds. The **Wasteland**: sun-bleached Spargus, built of stucco, sandstone and salvaged
metal, the desert's cliffs, palms and totems, and the Precursor temples. **Haven City** at war:
its districts of Jak II broken by the Metal Heads and the Krimzon Guard robots, with rubble,
craters and the Metal Heads' organic growth. The editor edits Jak 3: its levels, their decor is in
`library/jak3/`, their prefabs in `library/prefabricated/game/jak3/<level>/`, and `building_kit` and `build_structure` know them.

## The Wasteland

### Spargus City: `wascitya`, `wascityb`, the gates `wasdoors`, from afar `waswide`

- **What:** the desert city of Damas, between cliffs and the sea.
- **Shape:**
  - **Houses:** thick bleached stucco on stone bases, with flat roofs (the editor's
    `build_structure` knows Spargus' roofs are flat). Wall panels and armor of riveted metal,
    awnings of cloth and metal.
  - **Streets:** cement roads, ditches with edging, palms worn by the wind.
  - **Outer wall:** rock and metal plates.
- **Materials:**
  - Stucco and stone: `wascity-stucco-wall-bleached-01`, `wascitya-stone-top`,
    `wascity-stone-plain-wall-3`, `wascity-base`.
  - Ground: `wascity-ground-01`, `wascity-cement-road`, `wascity-ditch-wall-top-to-ground`.
  - Metal: `wascity-metal-dirty`, `wascitya-redish-metal`, `wascity-roof-1`,
    `wascity-outerwall-metal-b`, `wascity-outerwall-rock`.
  - Shore and palms: `wascity-palm-leaf-worn`, `wascity-ocean-shore-rocks`.
- **Details:** metal ladders, burning bushes, parking spots for the desert's vehicles, the
  palace's elevator shaft.
- **Avoid:** Haven's colors (red, green metal), pitched roofs.

### Damas' palace: `waspala`

- **What:** the king's palace above Spargus: pools, blue, yellow and red painted walls, metal
  window frames.
- **Materials:** `waspala-blue-01`, `waspala-yellow-01`, `waspala-red-01`, `waspala-pool-floor`,
  `waspala-metal-plate01`, `waspala-window-frame`, `waspala-cliff-rock-01`.

### The arena: `wasstada`, `wasstadb`, `wasstadc`

- **What:** the gladiators' arena dug in the rock: stands in rock shells, a throne, torch bowls,
  scaffolds of struts and bars, spears.
- **Materials:** `wstd-rockwall-01`, `wstd-stands-shell01`, `wstd-interior-rock01`,
  `wstd-floor-panel01`, `wstd-torchbowl-01`, `wstd-throne-wall01`, `wstd-scaffold-strut`,
  `wstd-scaffold-wall-02`, `wstd-ladder`, `wstd-spear01`.

### The desert: `desert`, `deserta` to `deserth`, the glider's sky `hanga`, `hangb`, the walker `deswalk`

- **What:** dunes, mesas and canyons. Oases with palms, Marauder corrals of scrap, Precursor
  totems.
- **Shape:**
  - **Land:** cliffs in layered bands with a sandy foot.
  - **Marauder corrals:** poles braced with rope and scrap plates.
  - **Totems:** stone with carved trims.
  - **The Dark Makers' walker** (`deswalk`): hard plates over tubes and tentacles.
- **Materials:**
  - Land: `des-mount-01`, `des-mount-02`, `des-mount-bottom-01`, `des-cliff-01`,
    `des-cliff-trans-01`, `des-rock-01`, `des-beach-01`, `des-sand-grass-01`.
  - Palms and shrubs: `des-palmtree-trunk-02`, `des-palm-top`, `des-shrub-pebbles`.
  - Totems: `des-totem-stone-01`, `des-totem-stone-trim`.
  - Marauder scrap: `des-corral-metal-02`, `des-corral-metal-04`, `des-pole-01`, `des-pole-brace`,
    `des-wasmetal01` to `25`, `des-plainrope`.
  - Ruins: `des-ruins-wall-01`.
  - The walker: `desw-hardplate-01`, `desw-tubes-segment-01`, `desw-tentacle-01`.

### Monk Temple: `templea` to `templee`, `templex`

- **What:** the Precursor monks' temple in the desert: courts, pillars and towers.
- **Shape:** massive sandstone blocks, pillars and trims around Precursor metal inlays.
- **Materials:** `templea_sandstone01`, `temple_sandstone_pill02`, `temple_sandstone_pill03`,
  `temple_sandstone_pill07`, `temple_sandstone_trim01`, `temple_sandstone_brick-01`,
  `temple_sandstone_base01`, `temple_sandstone_ground02`, `temple_pre-01-envmap`,
  `temple_metal01`.

### Volcano: `volcanoa`, `volcanox`

- **What:** a green, burnt island volcano: lava rock, grass and vines, burnt trees, metal plates,
  ladders, the Dark Makers' organic pipes.
- **Materials:** `vola-lava-rock-01`, `vola-grass-floor-01`, `vola-rock-side-wall`, `vola-vine`,
  `vol-bark-burnt`, `vol-plate-01-envmap`, `vol-metal-01`, `vola-dp-organic-pipe-envmap`,
  `vol-ladder-wood`.

### Mar's Eco Mine: `minea` to `mined` (`minee` is its sewer)

- **What:** an old mine of rusted machinery, blue paint, stone idols and carts.
- **Materials:** `minc-rust-01`, `minc-blue-paint-rust01`, `minc-blue-paint-rust05`,
  `minc-door-metal-01`, `minc-slab-01a`, `minc-cliff-face-01`, `minb-rock01`, `minb-stone26`,
  `minb-brok-edge-02`, `mina-idol-01-envmap`, `minc-train-pipe-cap-02`.

## Haven City at war

### Slums: `ctysluma`, `ctyslumb`, `ctyslumc`, `slumbset`

- **What:** Jak II's slums after the war.
  - `ctysluma` keeps Jak II's plaster houses, with craters blended into the ground
    (`city-slum-crater-blend-01`).
  - `ctyslumb` and `ctyslumc` were rebuilt: cleaner houses with purple and pink plaster, purple
    columns, awnings, railings, flower beds and trees.
- **Materials:**
  - Old houses: as Jak II's slums (`slum-ground-01`, `city-slumwall-07`, `city-slumbase-wall`,
    `city-dirtywood`, `city-slum-stonewall-bricks`).
  - Rebuilt houses: `ctyslumc-wall`, `ctyslumc-wall-trim`, `cityslumc-purple-plain`,
    `cityslumc-pinkish-purple`, `cityslumc-purple-column`, `ctyslumc-overhang-01`,
    `cityslumc-awning`, `ctyslumc-railing-trim`, `ctyslumc-floor-base`,
    `ctyslumc-flowerbed-flowers-a`.
- **The game's prefabs:** the slum houses of `library/prefabricated/game/` (`slum-house`,
  `slum-house-36-m`, `slums-building-1` to `11`, `slums-round-tower`...).

### Industrial zone: `ctyinda`, `ctyindb`

- **What:** Jak II's industrial section, crossed by the palace's cable section.
- **Shape:** 41 m tall walls with bulging metal and their pillars; houses of 1 to 3 floors
  (51 m wide). See [skill-modular-snapping](../skill-modular-snapping/SKILL.md) for the kit's
  sizes.
- **Materials:** `city-ind-band-dark-01`, `city-ind-metal-green-main-side`,
  `city-ind-wall-noisy-03`, `city-ind-wall-noisy-05`, `city-bigpipe-main-02`,
  `city-ind-border-stripe-dark-01`, `city-ind-palace-cable-section`, `city-green-pipe01`.

### Port: `ctyport`

- **What:** Jak II's port, defended: sea walls, bridges, big pipes, board vents.
- **Materials:** `city-port-bridge-brace01`, `city-port-pavmnt-01`, `city-port-seawalll-front`,
  `city-port-metal-block-01`, `city-port-grnd-cobl-01`, `city-port-bigpipe-siding`,
  `city-port-metal-green-main-side`.

### Streets, farms and HQ

- **Levels:** `ctygenb`, `ctyfarma`, `ctyfarmb`, the Freedom League's HQ `freehq`.
- **Streets and farms:** Jak II's streets and gardens. Ruined palace towers in the streets
  (`rub-palace-tower-side`). The farms are overgrown (`city-farm-treetop`).
- **HQ:** an interior of tiled floors, metal panels, pipes and monitors.
- **Materials:** `freehq-ground-tile-set1-m`, `freehq-panel-01-envmap`, `freehq-wal-plate03`,
  `freehq-pipe02`, `freehq-monitor01`.

### Ruins and stadium: `rubblea`, `rubblea2`, `rubbleb`, `rubblec`, `stadium`, `stadiuma`, `stadiumb`

- **What:** the destroyed sector around the palace's ruins and the old stadium, the way to the
  catacombs.
- **Shape:** broken walls with exposed beams and panels, rubble heaps, cement pillars, cracked
  marble, grass and streams taking over.
- **Materials:** `rub-rubble-01`, `rub-rubble-ground`, `rub-beam-gen`, `rub-wall-side-beam-02`,
  `rub-panels-01`, `rub-wall-trim`, `rub-wall-gen-01` to `04`, `rub-cement-a`,
  `rub-cement-pillars`, `rub-city-wall-inside-damaged`, `rub-stad-brick`,
  `rub-marble-floor-01-hitweak`, `rub-grass`, `rub-dirt-a`.
- **Damage:** this is the game's own vocabulary for a ruined building of Haven.

### Metal Head city and tower: `mhcitya`, `mhcityb`, `towera`, `towerb`

- **What:** the district taken by the Metal Heads: buildings grown over by tentacles, veins,
  egg clusters, skin on the ground. The tower is a stack of egg pods on bone.
- **Materials:**
  - City: `mhcity-wall-tentacle-01`, `mhcity-wall-tentacle-02`, `mhcity-building-base-01`,
    `mhcity-vein-01`, `mhcity-grunt-egg-03`, `mhcity-skin-ground-01`, `mhcity-baserock`.
  - Tower: `tow-wall-supports`, `tow-eggpod-01`, `tow-baserock`, `tow-basebone-01-envmap`.

### Sewers: `sewa` to `sewo`

- **What:** Jak II's sewers, larger: brick roofs, scaffolds, giant nuts and braces, arches.
- **Materials:** `sewer-pipe-02`, `sewer-pipe-rim-05b`, `sewer-plate-05`, `sewer-block-02`,
  `sewer-nut-01`, `sewer-scaffold-01`, `sewer-brick-block-10`, `sewer-brick-roof-02`,
  `sewer-stone-newarch-01`, `sewer-metal-block-04`.

### Haven seen from afar: `ctywide`, `lcitysml`, `intpfall`, and the palace's roof `intpalrf`

- **What:** the city drawn far away, in low-detail versions (`palcab-lowres-*`, `city-lowres-*`).
  `intpfall` and `intpalrf` are the opening: the palace's roof and its fall.
- **For a model:** never detail one for a distant view. A model for the palace's roof follows
  Jak II's Palace ([jak2.md](jak2.md)): `tpal-beam01`, `tpal-beam-redstripe01`,
  `tpal-panl_piller01`.

### Interiors: Naughty Ottsel `hiphog`, Vin's room `vinroom`, gun course `gungame`

The same as in Jak II (see [jak2.md](jak2.md), Interiors). `powergd` is the power grid: green
tiles.

### War Factory: `factorya` to `factoryd`

- **What:** the robots' factory: towers of panels and big pipes, red metal, arches. The Dark
  Makers' metal deep inside (`factoryd`).
- **Materials:** `fac-tower-pipe-01`, `fac-tower-base-03`, `fac-tower-panel-01`,
  `facb-big-metal-panl01`, `facb-bigpipe-01`, `facb_redmetal-02`, `facc-metal-panel-11`,
  `facc-bigpipe-01`, `facc-arches-01`, `facd-darkmaker-metal-01`.

## Under the city and beyond

### Haven Forest: `foresta`, `forestb`, `forestx`

- **What:** Jak II's forest, denser, with the city wall's frame and Metal Head spawn roots.
- **Materials:** `fora-foliage`, `fora-bark`, `fora-rock`, `fora-grass`, `fora-stream-rocks`,
  `fora-citywall-frame`, `fora-statue-stone-sides`, `fora-spawn-root`.

### Catacombs and subrails: `raila` to `railx`, `comba` to `combx`, the nest `nsta`, `nstb`

- **What:** Precursor tunnels under Haven.
  - **Subrails:** rails, pipes and cords with blue and yellow lights, on rock.
  - **Comb chambers:** circuit plates and rings.
  - **Nest:** carved caves with teeth and the spiders' eggs.
- **Materials:**
  - Subrails: `rail-edge-01-envmap`, `rail-gray-metal-01-envmap`, `rail-pipe-01-envmap`,
    `rail-cord-01-envmap`, `rail-rock-01`, `rail-light-blue-envmap`.
  - Comb chambers: `comb-plate-02-envmap`, `comb-crct-medium-envmap`, `comb-pipe1-envmap`,
    `comb-comb-tile`.
  - Nest: `nsta-wall`, `nsta-cave-floor-01`, `nsta-cave-carved-surface`, `nsta-cave-teeth`,
    `nsta-finger-pipe`.

### Precursor core: `precura` to `precurd`

- **What:** the Precursors' ship: honeycomb plates, segmented tubes, joints and nails, large
  floors, the terraformer's metal.
- **Materials:** `precur-tubes-small-01`, `precur-tubes-segment-01`, `precur-tube-joint-01`,
  `precur-plate-honey-01`, `precur-plate-large-01-envmap`, `precur-floor-large-01-lotweak`,
  `precur-frame-small-01`, `precur-wall-brace-01`, `precur-nail-01`.
