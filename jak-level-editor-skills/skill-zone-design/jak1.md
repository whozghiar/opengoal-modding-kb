# Jak 1: its zones

A bright, tropical world of hand-built villages on top of the Precursors' ancient machines.

- **Villages:** wood, rope, plaster and thatch.
- **Precursor:** polished metal, rings, "stonehenge" tops and blue glow.
- **Lurkers:** crude riveted scrap, balloons and chains.
- **Colors:** saturated, warm in the villages and cold in the Precursor halls.

Jak 1 is not in the editor's library. Its textures are the decompiler's, in the jak-project
checkout: `decompiler_out/jak1/textures/<level>-vis-tfrag/*.png` (also `-vis-shrub`,
`-vis-alpha`, `-vis-pris`). Its models are in `decompiler_out/jak1/levels/<level>/*.glb`. Shared
textures start with `cmn-` (Precursor, eco vents) and `gen-`.

## Sandover Village: `village1` (also `village1` and `sagehut` in Jak II)

- **What:** Jak's fishing village on a tropical cove. Huts on rocks and stilts over the water,
  Samos's hut under a giant tree, a Precursor silo.
- **Shape:**
  - **Huts:** round or rounded, plastered walls between thick wooden posts.
  - **Roofs:** domed or curved, of overlapping tiles with a rounded top cap
    (`vil-huttoprim`).
  - **Floors:** wooden, over stilts.
  - **Mood:** everything soft and organic, with no straight long lines. Windmill spinners, piers,
    fishing poles.
- **Materials:** `vil-wallplaster`, `vil-hut-wood-01` (and `-endcap`), `vil-hut-roof-tile-01`,
  `vil-under-roof`, `vil-huttoprim`, `vil-plankwood`, `vil-plankwood-onwalls`, `vil-woodfloor`,
  `vil-polishedwood`, `vil-metalband-01`, `vil-rope-01`, `vil-beachrock`, `vil-towerpole-01`,
  `vil1-palm-leaf-colorful`.
- **Details:**
  - Metal bands around posts, rope lashings, lamps on wooden posts (`vil-lamp-wood`).
  - Benches, beds inside the huts, rice fields with hay bales, fishing nets.
  - Palms everywhere.

## Geyser Rock: `training`

- **What:** the training island. Grass and beach rock around Precursor platforms and their silo.
- **Materials:** `tra-beachrock`, `tra-grass`, `pbot-precursor-metal-plain-01`,
  `pbot-stonehengetop`, `pbot-stonehenge-rim`, `vil-hut-wood-01`, `bch-plankwood`.

## Sentinel Beach: `beach`

- **What:** a beach of sand and rocks with the Precursor sentinels, a lurker cannon, the ruined
  outpost and the sages' stone walls.
- **Shape:** round huts like Sandover's, a stone wall with a straw roof, stone idols built of
  bricks, wooden piers.
- **Materials:** `bch-beachrock`, `bch-beachgrass`, `bch-hut-wood-01`, `bch-hut-roof-tile-01`,
  `bch-wallplaster`, `bch-plankwood`, `bch-sages-stonewall-01`, `bch-sages-strawroof-01`,
  `bch-idol-brick-01`, `bch-statue-rock`, `bch-outpostwall`, `bch-precursor-metal-plain-01`,
  `bch-stonehengetop`.

## Forbidden Jungle: `jungle`, `jungleb`

- **What:** a jungle of giant trees crossed by rope bridges, around a Precursor temple and its
  mirror towers.
- **Shape:**
  - **Rope bridges:** plank decks hung on ropes between anchors.
  - **Trees:** big trunks with vines.
  - **Lurker parts:** riveted panels with big bolts.
  - **Temple:** Precursor metal with circuit patterns and a blue glow (`jungleb`).
- **Materials:**
  - Jungle: `jng-rockwall-01`, `jng-treebark-01-4x4`, `jng-leafyground`, `jng-hut-wood-01`,
    `jng-logwood`, `jng-rope-01`, `jng-ropebridge-plank-01`, `jng-ropebridge-anchor-01`.
  - Lurker parts: `jng-lrkr-metal-02` to `08`, `jng-lrkr-bigmetal-01`.
  - Temple: `jng-precursor-metal-plain-01`, `jngb-precursor-circuitpattern-andwall-01`,
    `jngb-stonehengetop`.

## Misty Island: `misty` (and `intro`)

- **What:** a foggy island where the Lurkers gather: an arena, a wrecked ship, quicksand, bones.
- **Shape:**
  - **Lurker structures:** dark wood planks and decks, iron-banded wheels.
  - **Metal:** shingles, riveted plates, cannons.
  - **Ground:** rock walls with trims.
- **Materials:** `mis-wood-plank`, `mis-wood-plankB`, `mis-wooddeck`, `mis-wood-metal-trim`,
  `mis-boatwall`, `mis-boatdoor`, `mis-metalplating`, `mis-metalrivets`, `mis-metalshingle`,
  `mis-lrkr-metal-02`, `mis-rockwall`, `mis-ground`, `mis-plainrope`.
- **Mood:** muted greys, browns and greens, never bright.

## Fire Canyon: `firecanyon`

- **What:** a scorched canyon crossed on the zoomer, with lurker balloon mines.
- **Materials:** `fcn-crackedfloor`, `fcn-grass-burned`, `fcn-rockwall`, `fcn-rockwall-cliffside`,
  `fcn-spikerock`, `fcn-metal-pitted-01`, `fcn-precursor-metal-plain-01`, `fcn-stonehengetop`.

## Rock Village: `village2`

- **What:** the Blue Sage's village on the cliffs, shelled by the boulders of Klaww.
- **Shape:** huts like Sandover's, many broken. Canopies burnt at their edges, hut debris,
  cracked and angled wood, metal-cornered crates, bonsai trees on rock, burned grass.
- **Materials:** `vil2-hut-canopy-01`, `vil2-hut-canopy-01-burnt`, `vil2-hut-debris-01`,
  `vil2-crackedwood`, `vil2-anglewood-01`, `vil2-floor-01`, `vil2-crate-metal-corner`,
  `v2-grass-burned`, `v2-coral-rockbase`, `vil2-bonzailarge-02`, `vil2-lurkermetal-01`,
  `vil2-rope-01`.
- **Damage:** burnt canopy edges, splintered planks (`v2-kart-plank-splintered-01`). This is the
  game's own vocabulary of a damaged village.

## Lost Precursor City: `sunken`, `sunkenb`

- **What:** a drowned Precursor city: rooms of polished metal, helix stairs, hot pipes, coral and
  sea mushrooms.
- **Materials:** `sun-precursor-metal-plain-01`, `sun-precursor-wallplate-01`,
  `sun-precursor-circuitpattern-andwall-01`, `sun-floor-01`, `sun-floor-01-glow`,
  `sun-helix-floor-01`, `sun-walls-outside-01`, `sun-darker-metal`, `sun-hot-pipe-01`,
  `sun-coral-sponge-01`.

## Boggy Swamp: `swamp`

- **What:** mud, tar and twisted trees. Plank huts on stilts with tile roofs, Precursor tethers.
- **Materials:** `swp-mud-floor`, `swp-mud-floor-tar`, `swp-bark`, `swp-bark-roots`,
  `swp-hut-wood-01`, `swp-plankwood`, `swp-plankwood-rivets-float`, `swp-roof-tile-01`,
  `swp-stonewall`, `swp-metalstrip`, `swp-rope-01`, `swp-wood-spike-end`, `swp-moss-hang`.

## Precursor Basin: `rolling`

- **What:** rolling green hills cut by Precursor structures, raced on the zoomer.
- **Materials:** `rol-grass-ground-hirez`, `rol-rockwall`, `rol-precursor-metal-plain-01`,
  `rol-precursor-circuitpattern-andwall-02`, `rol-stonehengetop`.

## Mountain Pass: `ogre`

- **What:** a pass of rock walls and caves where Klaww throws boulders. Lurker scaffolds.
- **Materials:** `ogr-rockwall`, `ogr-rock-floor`, `ogr-crackedfloor`, `ogr-cavefloor`,
  `ogr-wood-01`, `ogr-rope-01`, `ogr-scaffold-brace`, `org-pinetree-big-bark`.

## Volcanic Crater: `village3`

- **What:** the Red Sage's mining village inside a crater. Huts of metal plates, gondolas, mine
  tracks, sulfur pools, lava.
- **Shape:** huts with metal walls and rusted roofs on cave rock, gondola supports, rimmed lurker
  metal, damaged plates riveted on.
- **Materials:** `vil3-hutwall`, `vil3-hutroof`, `vil3-hutroof-edge-01`, `vil3-hutfloor`,
  `vil3-hut-metalplain`, `vil3-gondola-roof-rusted`, `vil3-gondola-wallsupport`,
  `vil3-dmg-plate-01`, `vil3-lurkermetal-rim-01`, `vil3-cavewall-02`, `vil3-ground-01`,
  `vil3-crackedfloor`, `vil3-lava-floor`.

## Spider Cave: `maincave`, `darkcave`, `robocave`

- **What:** caves of dark crystals, giant mushrooms and webs. Lurker wooden scaffolds around a
  Precursor robot (`robocave`).
- **Materials:** `cv-wall-02-2x2`, `cv-wall-bumps-01`, `cv-cavefloor-01-2x2`, `cv-planks-all`,
  `cv-supportpole-end-2x2`, `cv-supportrope-01`, `cv-villagepole-new-wood`, `cv-roofweb`,
  `cave-darkcrystal`, `cv-rustymetal-01`, `cv-robo-precursor-metal-plain-01dk`.

## Snowy Mountain: `snow`

- **What:** snow, ice walls and a Lurker fort of wood and metal in the peaks.
- **Shape:** the fort's walls have stone bases and caps. Gates of wood bound with bolted metal,
  metal roofs, torches, horn bones.
- **Materials:** `snow`, `snow-path`, `snow-icewall-01`, `snow-ice`, `snow-crystals`,
  `snow-fortwallbase`, `snow-fortwallcap`, `snow-fortmetaltrim`, `snow-gate-wood`,
  `snow-gate-metal`, `snow-gate-bolts`, `snow-metalroof-01`, `snow-metalstrap-bolts`,
  `snow-forttorch-01`.

## Lava Tube: `lavatube`

- **What:** a tunnel of lava crossed on the zoomer. Lurker machinery, energy rails, balloons.
- **Materials:** `lava-ground`, `lava-floor`, `lava-cracked-floor-top`, `lava-flow-01`,
  `lava-coping`, `lava-lrkr-metal-01`, `lava-lrkrm-plank`, `lava-metal-pipes`,
  `lava-energyrail-top`, `lava-energycoil`.

## Gol and Maia's Citadel: `citadel`, and the summit `finalboss`

- **What:** the Precursor citadel taken over by Gol and Maia. Colored eco rooms, pink beams,
  gantries, the sages' cages, the silo and its robot.
- **Materials:**
  - Citadel: `cit-precursor-largewall`, `cit-precursor-metal-plain-01`,
    `cit-precursor-circuitpattern-andwall-01`, `cit-precursor-metal-floor`,
    `cit-blueroom-metal-01`, `cit-pinkbeam`, `cit-metalrivet`, `cit-wood-plank`, `citb-metal-blue`.
  - Summit: `fin-precursor-metal-plain-01`, `fin-largerock`, `fin-leafyground`.
