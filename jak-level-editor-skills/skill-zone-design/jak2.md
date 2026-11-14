# Jak II: its zones

Haven City is a fortified city under the Baron's rule: the Krimzon Guard's heavy, rounded metal
over an older town of plaster, stone and wood. Every district has the same base (cobbles,
canals, pipes, guard turrets, cables overhead) and a look of its own. Outside the walls, Precursor
ruins and machines in the wild.

Textures are named as in `library/jak2/<level>/decor/*.glb` (their images are there). Sizes were
measured on the game's pieces, in meters. What Haven City's decor measures district by district
(triangle budgets, textures' repeat and color, light, the pieces copied most) is in
[haven-city-jak2.md](haven-city-jak2.md): read it before modeling for the city.

## Haven City

### Slums: `ctysluma`, `ctyslumb`, `ctyslumc`

- **What:** the poorest district, under the fortress. Two- to four-storey houses packed along
  dirt streets and ditches. `ctyslumc` is built over the water: shacks on stilts joined by plank
  catwalks.
- **Shape:**
  - **Ground floor:** thick plastered walls on a rubble-stone base. Heavy wooden posts every 6
    to 10 m, wider at the foot than at the top, often leaning, sometimes bound with metal straps.
  - **Between storeys:** a thick band of beams and stone blocks. The posts end in block caps
    under it.
  - **Upper storeys:** in planks or corrugated sheet, or in brown riveted panels
    (`city-side-support`) inside a blue-grey frame (`city-slum-greenmetal-tube`). That last one
    is the warm color of the streets, seen beside the game's houses in `ctysluma`. They jut out
    over the street on beams and diagonal struts.
  - **Color:** a ground floor of grey-blue plaster with dark streaks under its beams, an upper
    storey of brown panels. Pale plaster everywhere reads as untextured next to the game's
    houses.
  - **Roofs:** overhanging, flat or barely sloped, of wooden slats and damaged metal plates on
    thin rafters.
  - **Openings:**
    - Doors are small for the facade and recessed in a thick frame with rounded corners: a
      brown wooden frame, or one painted blue-grey.
    - Windows are dark recessed holes with a wooden H-shaped shutter frame.
  - **Mood:** everything is a little crooked.
- **Materials:**
  - Plaster: `city-slumwall-01`, `05`, `06`, `07` (cream, 5.5 to 7.3 m per repeat),
    `city-slum-lightwall`, and `city-slum-blotch-withstreaks-01` (stains).
  - Bases: `city-slumbase-wall` and `city-slumbase-wall-3` (5 m), `slum-stone-03` and
    `slum-stone-bottom` (7 m).
  - Broken plaster: `city-slumbase-wall-broken-to-bricks` and `-2` (2.3 m),
    `city-slum-brick-showing-through`, `city-slum-stonewall-bricks`.
  - Boarded openings: `city-slumbase-wall-boarded`.
  - Wood: `city-dirtywood` (3.3 m), `city-slum-wood-plain` (4.4 m),
    `city-slum-building-frame`.
  - Sheet metal: `city-slumwall-metalsiding-01`, `city-metal-wall`.
  - Roofs: `city-slum-roof` and `city-slum-roof-1` (3 to 3.7 m), `city-slum-roof-side`,
    `city-roofmetal-rim`, `city-slum-awning-rustedmetal`.
  - Openings: `city-slum-door-01`, `city-slum-litwindow` (a lit window),
    `city-slum-stain-window-01`.
  - Pipes: `city-slum-medpipe-01` and `02`, `city-slum-bigpipe-02` to `04`,
    `city-slum-greenmetal-tube`.
  - Ground: `slum-ground-01` (8.4 m), `city-slum-ground-2-ditch-03`, `slum-ditch-walll-01`.
  - Extras: `city-slum-vine`, `city-slum-wire`, `city-slum-clothesline-01`,
    `city-slum-hangsign-01` to `03`.
  - Over the water: `city-slums-overwater-wall-01`, `city-slumwall-wood-02`, `city-slum-rope-01`,
    `city-slums-nail`.
- **Inside a house:** the slums have no interiors; take them from Haven's, by kind of part
  (`tools/game_textures.py --piece <kind> --game jak2`):

  | Part | Texture | From |
  |---|---|---|
  | Beams, joists, posts inside, furniture legs | `hide-wood-beam` | the Underground's hideout |
  | Floors | `hide-floor-01` | the hideout |
  | Planks, stair treads, boards | `consite-lurker-wood-01` | the Construction Site |
  | Railings | `ctyslumc-railing-trim` | Jak 3's slums |
  | Tables, shelves, crates | `city-mark-wood-plain` | the Bazaar |
  | Stools, benches | `kiosk-stool-seat-02`, `kiosk-stool-leg-01` | the kiosk |
  | Barrels | `barrel01`, rings `kiosk-barrel-metal-ring-01` | the city |
  | Stove | `hide-furnace`, `hide-furnace-piece-metal` | the hideout |
  | Bed | `hide-bed-matress`, `hide-bed-top`, `hide-bed-pillow` | the hideout |
  | Rug | `onin-rug-long` | Onin's tent |
  | Lamps | `city-slum-bigpipe-04`, light `city-bluelight` | the slums' street lamps |
- **The game's kit** (piece: size x, y, z):
  - Door: `city-slum-door-01` 4.5 x 7.5 x 0.2, frame `city-slum-doorframe-01` 5.5 x 8.1 x 0.5.
  - Window: `city-slum-window-01` 3.8 x 5.0.
  - Facades: storeys `city-slum-facade-mid-*w-12h` 12.6 to 15.3 high and 7 to 10 deep
    (12w: 19.4 wide, 48w: 56.4); `city-slum-facade-divider-12h-02` 1.2 x 10.7.
  - Beams: `city-woodbeam-small-01` 0.36 x 0.67 x 8.1, `city-slum-stonebeam-med-01` (a post)
    1.4 x 9.5 x 1.3.
  - Roofs: `city-slum-roof-woodslats-01` 10.6 x 2.0 x 9.2 (60 triangles),
    `city-slum-roof-metalplates-damaged-01` 18.3 x 2.6 x 8.8.
  - Small parts: awning `city-slum-awning-rustymetal-01` 11.5 x 3.2, chimney
    `city-slum-chimney-01` 0.9 x 5.8, railing `city-slum-railing-slats-01` 2.5 x 2.3, lamp
    `city-slum-lamp-01` 5.8 x 3.2 x 2.4.
- **Details:**
  - Pipes run down the walls, held by ring brackets, with elbows and joints.
  - Chimneys, transformers, hanging vines and wall vines.
  - Clotheslines and wires between the houses.
  - Stick-out signs and lamps on arms.
  - Plaster broken off at the corners, stains under the windows.
  - Dry grass tufts and pebbles at the foot of the walls, puddles of dirt.
  - Burning cans (`city-slum-burning-can`), fruit stands, crates.
  - Over the water: catwalk posts with rope wraps, plank walkways with broken railings,
    cisterns.
- **Avoid:**
  - Straight, clean walls.
  - Glass windows.
  - Bright colors: the slums are cream, brown, grey-blue and rust.
  - Pitched tile roofs.
  - The Krimzon red, which belongs to the fortress above.

### Bazaar: `ctymarka`, `ctymarkb`, with Onin's tent `onintent`

- **What:** the market. A crowded square of stalls between stone houses, cloth over the street.
- **Shape:** stone walls of larger blocks than the slums. Wooden stalls with counters and cloth
  awnings. Baskets, sacks and crates piled high.
- **Materials:**
  - Stone and floor: `city-mark-wall-stone-01` to `06`, `city-mark-floor`.
  - Stalls: `city-mark-wood-plain`, `city-mark-rope-01`.
  - Goods: `city-mark-bucket`, `city-mark-bottle-cork-01`, `city-mark-garlic-01`.
  - Onin's tent: `onin-tent`, `onin-tent-wood-posts`, `onin-rug-long`, `onin-candle-holder`,
    `onin-skull`.
- **Details:** hanging garlic, buckets, ropes, bottles, fruit stands (`cty-fruit-stand`), market
  crates and baskets. Onin's tent is all cloth, rugs, candles and skulls.

### Port: `ctyport`, the sea wall `portwall`, the Hip Hog saloon `hiphog`

- **What:** the docks: quays, sea walls, warehouses, bridges, ships. Krew's saloon.
- **Shape:** massive sea walls with a sloped front, warehouses in riveted metal on stone bases,
  metal bridges on braces. The city's outer wall is a huge frame of metal panels.
- **Materials:**
  - Port: `city-port-seawalll-front`, `city-port-metal-block-01`,
    `city-port-metal-green-main-side`, `city-port-grnd-cobl-01`, `city-port-bridge-brace01`,
    `city-port-copper-lines`, `city-port-roofmetal`, `city-port-wallbase`.
  - Outer wall: `citywide-wall-frame`, `citywide-wall-mainmetal`, `citywide-wall-orange-plain`,
    `citywide-wall-greybolts`.
  - Saloon: `hip-tred-check01` (checkered), `hip-tredmetal09`, `hip-tyellwall01`,
    `hip-tlogorag01`.
- **Details:** bollards, ropes, copper conduits, security walls. Inside the saloon: trophies,
  mirrors, curtains, checkered floors.

### Industrial section: `ctyinda`, `ctyindb`

- **What:** factories and warehouses, the largest buildings of the city.
- **Shape:** blocky masses with rounded corners, 30 to 40 m tall. Dark horizontal bands, bulging
  green metal sides, huge pipes, vents.
- **Materials:** `city-ind-band-dark-01`, `city-ind-wall-noisy-03`, `city-ind-wall-noisy-05`,
  `city-ind-metal-green-main-side`, `city-ind-border-stripe-dark-01`, `city-ind-dark-marble`,
  `city-ind-wall-base-07`.
- **Details:**
  - Hazard stripes, big pipes on brackets, guard turrets.
  - Parking spots for zoomers.
  - The door to Vin's (`vin-door-ctyinda`).

### Main Town: `ctygena`, `ctygenb`, `ctygenc`

- **What:** the central streets and canals of the middle class.
- **Shape:** tall rounded buildings in red-brown and silver panels. Overhanging metal awnings,
  bevelled tops. Canals with railings, steps and bridges.
- **Materials:**
  - Panels: `ctyn-brown-red`, `ctyn-silver-wall-lower-01`, `ctyn-hangmetal`, `ctyn-beams`,
    `ctyn-trim`, `ctyn-top-bevel-small-bottom`, `ctyn-base-vent-01`.
  - Ground and canals: `ctyn-ground`, `city-canal`.
  - Street parts: `city-outpostwall`, `city-railing`, `city-step`, `city-sideframe`.
- **Details:** street lamps (`ctyn-lamp`), neon signs of the Baron (`city-neon-praxis`), guard
  turrets, lurker pipe lids, parking spots.

### Palace district and the Palace

- **Levels:** `ctypal`, the cable car `palcab`, the roof `palroof`, the entrance `palent`, the
  shaft `palshaft`, the throne room `throne`.
- **What:** the gardens, fountains and statue around the Baron's spire, and the spire itself.
- **Shape:**
  - **District:** stone plazas, water, trees, statues on metal-striped pillars.
  - **Palace:** a vertical tower of red-painted steel beams, rings and cables. Platforms with
    pillar caps, perforated grey plates.
  - **Throne room:** dark stone and metal window frames.
- **Materials:**
  - District: `city-pal-ground-01`, `t-citypal-met-strp01`, `t-citypal-met-pill-01`,
    `t-citypal-statue-stone-01`, `t-citypal-tree-01`.
  - Palace: `tcab-beam01`, `tcab-ring-01`, `tpal-beam01`, `tpal-beam-redstripe01`,
    `palroof-grey-holes`, `tent-red-paint-02`, `shaft-cable01`.
  - Throne room: `thrm-darkstone-01`, `thrm-metal-windowframe-03`.

### Gardens: `ctyfarma`, `ctyfarmb`

- **What:** the farms inside the walls: fields, orchards, sprinklers.
- **Shape:** low stone walls with capped tops, metal brackets, vegetable trees.
- **Materials:** `farm-grass-ground-01`, `farm-grass-ground-02`, `city-farm-dirtymetal-01`,
  `city-farm-metal-bracket-02`, `city-farm-vegtree-bark-01`, `city-farm-rock`,
  `city-farm-stone-wall-01`, `city-farm-wall-top`.
- **Details:** cabbages, marrows, beehive trees, sprinkler barrels, yakows.

### Stadium and race tracks: `stadium`, `stadiumb`, `stadiumc`, `stadiumd`, garage `garage`

- **What:** the class race stadium, its tracks and Keira's garage.
- **Shape:**
  - **Stadium:** marble and cobble plazas, stairs, statues.
  - **Tracks:** wide tech walls, glass walls, metal slopes, track borders.
  - **Garage:** girders, ring lights, yellow metal.
- **Materials:**
  - Stadium: `stdm-cobble-floor-01`, `stdm-marble-floor-01-hitweak`, `stdm-stairs-01`,
    `stdm-wall-04`, `stdm-statue-stone-01`.
  - Tracks: `stdmb-track-01`, `stdmb-track-side-01`, `stdmb-tech-walls`, `stdmb-glass-wall-03`.
  - Garage: `stdm-gar-ring-light`, `stdm-gar-metal-01`, `stdm-gar-yellow-metal-02`.

### Interiors: hideout, Vin's room, kiosk, Oracle, gun course

- **Levels:** the Underground's hideout `hideout`, Vin's room `vinroom`, the kiosk `kiosk`, the
  Oracle `oracle`, the gun course `gungame`.
- **Shape:**
  - **Hideout:** brick vaults with wooden beams, pipes, bunks.
  - **Vin's room:** a control room of monitors and blue lights.
  - **Kiosk:** a stall of barrels, ropes and a grill.
  - **Oracle:** a wooden chamber around a Precursor statue with glass eyes.
  - **Gun course:** a hall of pavements, light walls and big pipes.
- **Materials:**
  - Hideout: `hide-wallbrick-01`, `hide-floor-01`, `hide-wood-beam`, `hide-plain-metal`.
  - Vin's room: `vin-wall-01-envmap`, `vin-monitor-rim`, `vin-pipe-01`.
  - Kiosk: `kiosk-barrel-wood-01`, `kiosk-rope-01`.
  - Oracle: `oracle-wood-beam-01`, `oracle-wood-floor-01`, `precursor-metal-plain-01-envmap`.
  - Gun course: `gun-lightwall-01`, `gun-pavement-01`, `gun-building-wall-gray-01`.
- **Scale:** these interiors are 17 to 36 m high.

### The JET-board park: `skatea`

- **What:** a training park for the JET-board: grind rails, ramps, metal floors, stone edges.
- **Materials:** `skatea-grind-rail-01`, `skatea-rail-04`, `skatea-metalfloor-01`,
  `skatea-gray-metal-01`, `skatea-wall-edge-01`, `skatea-pipe-03`, `skatea-blnk-stone-01`,
  `skatea-track-01`.

### Haven seen from afar

`ctywide`, `lcitylow`, `palout`, `cascity`: low-detail versions of the city
(`palcab-lowres-*`). They are drawn far away: never model detail for them.

## Krimzon Guard and the Baron

### Fortress and prison: `fordumpa` to `fordumpd`, `forexita`, `forexitb`, `forresca`, `forrescb`, `prison`

- **What:** the Baron's fortress: ammunition dumps, conveyors, cells.
- **Shape:** dark industrial halls of grey metal and asphalt. Pipes everywhere, rubber-rimmed
  openings, grates, chains, blue panels. The prison has grey walls, cell doors and chairs with
  shackles.
- **Materials:**
  - Fortress: `fort-wall-trim`, `fort-wall-trim-plain`, `fort-pipe-01`, `fort-pipe-small-01`,
    `fort-tasphlt01`, `fort-grate-01`, `fort-metal-plain`, `fort-big-chain`, `fort-tblupanl01`,
    `fort-supportmetall`, `sewer-rubber-rim-01`.
  - Prison: `prison-grey-wall`, `prison-door-frame-01`.
- **Details:** electric gates, turrets, conveyors, floor spikes, lifts.

### Weapons Lab: `castle`, `casboss`, `caspad`, `casext`

- **What:** the lab where the Heart of Mar is kept, and the cliff where the air train lands
  (`caspad`).
- **Shape:** sheet-metal halls with bronze trims, stone arches, rings and pipes. The landing pad
  is grass, rock, cave stalactites and palms.
- **Materials:**
  - Lab: `tcas-mtl-shet-01`, `tcas-mtl-shet-02`, `tcas-pipe-01`, `tcas-brnz-trim-01`,
    `tcas-stone-arch-01`, `t-c-boss-beam-01`.
  - Landing pad: `cpad-grass`, `cpad-rock`.

### Construction Site: `consite`, `consiteb`

- **What:** the Baron's work site in the rock: silos, girders, elevators.
- **Materials:** `consite-edge-01`, `consite-fort-big-chain`, `consite-rockrubbel-01`,
  `consite-girder-01`, `consite-dirt-ground-01`, `consiteb-metal-plain`,
  `consiteb-const-dia-plate`.

### Sewers: `sewer`, `sewerb`, `sewesc`, `sewescb`

- **What:** the tunnels under the city.
- **Shape:** brick and concrete channels, metal blocks, large pipes with rims, grates.
- **Materials:** `sewer-metal-04`, `sewer-metal-block-04`, `sewer-pipe-01`, `sewer-pipe-02`,
  `sewer-pipe-rim-10`, `sewer-grate-01`, `sewer-plate-02`, `sewer-brick-block-01`,
  `sewer-concrete-block-02`.
- **Details:** mines, rotating blades, water.

### Under Port: `under`, `underb`

- **What:** the drowned old port under the city.
- **Shape:** carved stone faces and pillars, metal fittings, water stones, seaweed.
- **Materials:** `under-t-stone-face-01`, `under-t-pil-stone-01`, `under-t-crod-stone-01`,
  `under-t-metal-01-envmap`, `under-ground-01`, `under-t-water-stone-03`.

## Outside the walls

### Haven Forest: `forest`, `forestb`

- **What:** a forest of big trees and streams around Precursor relics. `forestb` is the great
  tree, and the shrine where young Samos waits.
- **Materials:**
  - Forest: `for-grass`, `for-grass-fringe`, `for-rock`, `for-bark`, `for-stream-rocks`,
    `for-vineleaf`.
  - Great tree and shrine: `for-mother-bark`, `for-zenwall-wood-plain`,
    `for-zenwall-tarnished-metal`, `for-zenwall-emblem`.

### Dig: `dig1`, `dig3a`, `dig3b`

- **What:** the excavation where the Baron digs for the Precursor stone.
- **Shape:** caves with stalactites, metal floors, conveyors, clasps, ropes and wires.
  `dig3b` is burnt, with lava.
- **Materials:** `dig-cave-floor-01`, `dig-cave-stalags-04`, `dig-wall`, `dig-metal-floor`,
  `dig-rope`, `dig-wire`, `dig-cave-floor-burnt`, `dig-lava-transition-01`.

### Strip Mine: `strip`

- **What:** an open-pit mine with the city's machinery.
- **Materials:** `strip-ground-01`, `strip-rockwall-01`, `strip-metal-02`, `strip-pipe-03`,
  `strip-rockrubbel-01`.

### Mountain Temple: `mountain`, `mtnext`

- **What:** cliffs and pines around a Precursor temple.
- **Shape:** Precursor platforms and dice in polished metal between rock walls, bonsai trees.
- **Materials:** `mtn-rockwall`, `mtn-rockwall-lowgrass`, `mtn-precursor-metal-plain-01-envmap`,
  `mtn-precursor-metal-edge-01-envmap`, `mtn-bonzai-trunk`, `mtn-pinetree-big-bark`,
  `mtn-grass-ground-hirez`.

### Dead Town: `ruins`

- **What:** the ruins of old Haven.
- **Shape:** broken stone walls with ragged tops, canals, buttresses, fallen beams.
- **Materials:** `ruins-stone-03-edges`, `ruins-stone-05`, `ruins-stone-alternate`,
  `ruins-canal-top`, `ruins-base-wall-broken`, `ruins-base-wall-broken2`,
  `ruins-citywall-frame`, `ruins-endblocks`.

### Pumping Station: `atoll`, `atollext`

- **What:** water pumps and windmills on an atoll at sea.
- **Shape:** riveted metal rings, rails and pistons on sand and cliffs. Old city-wall frames.
- **Materials:** `atoll-r-endblocks`, `atoll-beach-01`, `atoll-tbolt`, `atoll-tclifface01`,
  `atoll-tmetarail-01`, `atoll-tringrust02`, `atoll-r-citywall-frame`,
  `ruins-butress-metal-01`.

### Drill Platform: `drill`, `drillb`, `drillmid`, `drillmtn`

- **What:** a drilling rig full of Metal Head eggs, in front of snowy mountains.
- **Materials:** `drill-electrode-blue`, `drill-floor-02`, `drill-brick-block-01`,
  `drill-beam01`, `drill-big-metal-panl01`, `drill-pipe-02`, `drill-door-frame-02`; the
  mountains `drill-mountain-01`, `drill-snow-01`, `drill-metal-ring-01`.

### Mar's Tomb: `tomba` to `tombe`, `tombboss`, `tombext`

- **What:** the tomb of Haven's founder.
- **Shape:** monumental stairs, pillars and tunnels of light and dark stone. Panels painted
  blue, yellow, red and green. Winged doors, a sphinx.
- **Materials:** `tomb-stair_stone01`, `tomb-stair_stone_dark01`, `tomb-brick-02`,
  `tomb-piller01`, `tomb-blue-01`, `tomb-yellow-01`, `tomb-red-01`, `tomb-green-01`,
  `tomb-tunel_trim01`, `tomb-cros_trim01`.

### Metal Head Nest: `nest`, `nestb`

- **What:** Kor's lair.
- **Shape:** grey rock and sand pierced by organic pipes and finger-like growths, egg skins, a
  mechanical brain, dark eco pools.
- **Materials:** `nest-greydirt`, `nest-sand-01`, `nest-backrock`, `nest-greywalls`,
  `nest-red-strip`, `nest-orgpipe`, `nest-finger-pipe`, `nestb-eggskin`, `nestb-mekbrain`.

### The canyon of the lighthouse: `mincan`

- **What:** rock walls, ferns and a lighthouse with its cogs and lens.
- **Materials:** `mincan-rockwall`, `mincan-barkv1`, `mincan-concv1`, `mincan-fern`,
  `mincan-stone-trim01`.

### Sandover in Jak II: `village1`, `sagehut`

The village of Jak 1, seen again (see [jak1.md](jak1.md), Sandover).
