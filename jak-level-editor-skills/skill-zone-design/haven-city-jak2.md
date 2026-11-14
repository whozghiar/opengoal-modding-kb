# Haven City in Jak II: the measured charter

Read this before modeling any structure or object in Blender for Haven City or in Jak II's style:
its numbers are the budget and the look to match. [jak2.md](jak2.md) describes each zone in words
and lists its textures; this file gives what the game's own decor measures, district by district.

The numbers come from the decor of the 34 levels of Haven City (`library/jak2/<level>/decor/`),
measured by `tools/zone_charter.py`. Run it again for a level, or for another game:

```text
python tools/zone_charter.py --game jak2 ctymarka ctyport      # a table per level, in markdown
python tools/zone_charter.py --game jak2 --json out.json ctygena
```

Colors are given as the game shows them: the images are stored at half brightness (128 is white,
the PS2's scale) and the tool doubles them. "Light" is the vertex light the game baked, 1 being a
texture at its own brightness.

## What every district shares

### 1. A cheap shell, and a kit of small pieces copied thousands of times

A district is two layers:

- **The shell** (the game's tfrag): the ground, the big walls and masses, in very large triangles.
  It holds 500 to 16,000 triangles per district, each triangle covering 8 to 28 m².
- **The kit** (tie and shrub): 50 to 120 different pieces, copied 5,000 to 16,000 times. It holds
  almost all the triangles of a district (200,000 to 480,000). A piece is small and cheap: 36 to
  64 triangles at the median, 130 to 360 for the largest tenth.

So a new building is a simple massing in few large faces, dressed with the district's own small
pieces: beams, pipes, windows, trims, roof slats, wear decals. Detail never comes from a dense
mesh.

### 2. Triangle budgets by kind of piece

Medians measured across the districts (triangles of one piece, its largest side):

| Piece | Triangles | Size | Notes |
|---|---|---|---|
| Wear decal (stain, blotch, streaks) | 2 | 2.5 to 7 m | a quad laid on a wall |
| Beam, post, strut | 12 to 48 | 3.5 to 8 m | slums 20, Main Town 48 |
| Window | 8 to 48 | 3.5 to 7 m | 8 when it is a dark quad in a frame |
| Door | 36 to 96 | 5 to 13 m | with its frame |
| Pipe part (ring, straight, 45, 90 elbow) | 24 to 72 | 2 to 6 m | 6 to 8 sides |
| Roof part (slats, plates, tiles) | 12 to 64 | 6 to 16 m | the slums' slats 60 |
| Trim, bevel, cap, cornice | 28 to 128 | 12 to 30 m | Main Town's bevels 48 |
| Wall module | 32 to 128 | 12 to 45 m | the industrial walls 128 |
| Railing segment | 24 to 68 | 4 to 13 m | |
| Lamp | 18 to 58 | 1 to 8 m | |
| Prop (crate, stall, barrel) | 36 to 150 | 1 to 14 m | |
| Plant (grass tuft, vine, canopy) | 16 to 96 | 2 to 18 m | crossed quads |
| Ground or canal part | 40 to 134 | 8 to 16 m | |

A whole building of the game is 2,500 to 9,400 triangles, 0.2 to 2 triangles per m² of facade
(0.6 to 1 in most districts).

### 3. Wear everywhere, as decals

In 9 of the 12 outdoor districts the most copied piece is a 2-triangle wear decal: 800 to 3,250
copies a district (`city-slum-blotch-withstreaks-01`, `city-wall-window-stain-01`,
`city-ind-wall-blotch-withstreaks-01`...). Every wall gets streaks under its windows and beams,
stains at its foot, plaster or paint broken off at its corners. A wall with none reads as a model
from elsewhere.

### 4. Textures

- **Size:** 128 x 128 px nearly everywhere; 64 px for small or detail images (wood planks, trims,
  roof slats); 256 px only for a few big surfaces (the palace's metal walls, foliage).
- **Repeat, in meters per repeat of the image:**

  | Surface | Meters |
  |---|---|
  | Wood planks, beams | 1.8 to 3.5 |
  | Pipes | 1.7 to 3 |
  | Roofs | 3 to 7 |
  | Walls (plaster, stone, metal panels) | 3 to 6, up to 8.5 for big fillers |
  | Ground, cobbles, floors | 4 to 8.5 |
  | Huge walls (the city wall, the palace) | 9 to 16 |
  | Interiors (the hideout, the saloon) | 0.5 to 2 |

- **Laid once:** doors, windows, signs, posters and stains are images laid once on their face,
  never repeated.
- **A district wears its own textures:** 50 to 90 images, each district's own prefix (`city-slum-`,
  `city-mark-`, `city-port-`, `city-ind-`, `ctyn-`, `t-citypal-`, `city-farm-`). Shared ones are few
  (`city-dirtywood`, `city-metal-wall`, `city-stain-wall-01`, the city wall's `citywide-wall-*`).

### 5. Color and light

- **Muted:** colors are greyed, low in saturation, rarely pure. Each district has one family:
  slums cream, rust-brown and blue-grey; bazaar pale stone and blue roofs; port green-grey metal;
  industrial cold grey; Main Town red-brown and silver; gardens yellow-green.
- **Accents:** strong color only on small things laid once: the bazaar's striped awnings (red and
  white, blue, green), lit windows (`city-slum-litwindow`, orange), neon signs, the Krimzon red of
  the Guard's banners and the palace's red paint, the stadium's orange sails.
- **Light:** the baked light is neutral grey, at 0.6 to 0.8 of the textures' brightness outdoors
  (slums 0.65 to 0.68, bazaar 0.70 to 0.76, industrial 0.66 to 0.70, Main Town 0.61 to 0.71, gardens
  0.80), 0.15 to 0.3 indoors. No colored light outdoors. In Blender, bake nothing: the editor
  lights the model from the level around it.

### 6. Shape and scale

- **Heavy and soft:** thick walls, rounded or bevelled edges, overhanging tops, posts wider at the
  foot. Bevels are a few flat faces, never a smooth subdivided curve.
- **Not straight:** in the slums and the bazaar, walls lean, beams cross at angles and roof slats
  splay out. The Guard's and Main Town's metal is upright and ordered, but its corners are rounded
  and its tops bevelled.
- **The game's scale:** a storey is 12 to 15 m, a door 4.5 x 7.5 m in the slums and up to 13 m in
  Main Town, a window 3.5 to 4.5 m. Streets are 15 to 30 m wide, canals 10 to 15 m. Buildings are
  20 to 45 m high, the industrial blocks 30 to 45 m.

## The districts

Each card: what it looks like, its numbers, how it is built, its palette (the textures covering the
most surface, their color and repeat), the pieces it copies most.

### Slums: `ctysluma`, `ctyslumb`

- **Look:** two- to four-storey houses packed along dirt streets, wires overhead. Grey-blue
  plaster ground floors with dark streaks, brown upper storeys, roofs of wooden slats splayed out
  on all sides like petals, everything leaning.
- **Numbers:** shell 15,000 to 16,500 triangles, 7 to 8 m² each; kit 210,000 to 250,000 triangles,
  15,000 copies of 87 pieces, a piece 44 triangles median; 245,000 to 280,000 triangles of grass
  and small props (the most of any district). Light 0.65 to 0.68.
- **Built:** plaster walls on stone bases, posts and beams (`city-woodbeam-small-01`, 20
  triangles, 1,890 copies), slat roofs (`city-slum-roof-woodslats-01`, 60 triangles, 840 copies),
  dark recessed windows with H shutters (`city-slum-window-01`, 32 triangles, 600 copies),
  ditches with stone edges.
- **Palette:** `city-dirtywood` 15% (grey-green wood, 1.9 m), `city-slum-roof-1` 10% (cream, 3.4 m),
  `city-slum-roof` 9% (pale green-grey, 4.2 m), `slum-ground-01` 8% (sand, 8.4 m),
  `city-slumwall-06` and `01` 13% (pale plaster, 5 to 6 m), `city-metal-wall` 5% (tan, 3.7 m),
  `slum-stone-03` 4% (ochre stone, 5 m). By surface: orange-brown 30%, green 20%, grey 19%.
- **Copies most:** beams 4,200, wear decals 2,600, grass tufts 2,500 (`city-slum-cattail-grass`,
  60 triangles), roof slats 1,650, pipes 1,100, windows 900.

### Slums over the water: `ctyslumc`

- **Look:** round shacks on stilts over the sea, joined by plank catwalks with rope-wrapped posts.
  Their roofs are cones of slats splayed out.
- **Numbers:** shell 4,250 triangles, 15 m² each; kit 285,000 triangles, 13,700 copies of 101
  pieces, a piece 36 triangles median. Light 0.58.
- **Built:** catwalk pylons (`city-slums-catwalk-pilon-01`, 48 triangles, 590 copies) and
  underbeams, small beams (`city-slum-woodbeam-small-01`, 12 triangles, 2,860 copies), metal panel
  patches (`city-slums-overwater-metalpanel-03`).
- **Palette:** `city-slum-wood-plain` 26% (cream planks, 2 m), `city-dirtywood` 12% (2.1 m),
  `city-slum-roof` 9%, `city-slumwall-metalsiding-03` 5% (blue-grey sheet, 3.3 m),
  `city-slums-overwater-wall-01` 3% (orange-red, 8.5 m).

### Bazaar: `ctymarka`, `ctymarkb`

- **Look:** a sand square ringed by one- and two-storey stone houses with heavy blue-grey tiled
  roofs, round dormers and a green cornice. Arcades of stocky pillars on stone feet, shelves of
  pots and bottles behind, striped cloth awnings (red and white, blue, green) and purple banners.
- **Numbers:** shell 6,800 to 7,600 triangles, 13 to 16 m² each; kit 316,000 to 338,000 triangles,
  7,600 to 8,900 copies of 64 to 88 pieces, a piece 40 to 44 triangles median. A building 6,600 to
  9,400 triangles, 1 to 2 per m² of facade. Light 0.70 to 0.76 (the brightest town district).
- **Built:** stone walls in big blocks, roof tiles in long rows, border rocks along the ground
  (`city-mark-border-rocks-02`, 84 triangles, 420 copies), shelves and bottles
  (`city-mark-shelf-5bottles-01`, 150 triangles), awning poles and brackets.
- **Palette:** `city-mark-floor` 13 to 20% (pale sand, 8 m), `city-mark-wall-stone-06` 14 to 16%
  (green-grey stone, 3.1 m), `city-mark-roof-tiles` 12% (blue-grey, 7 m),
  `city-mark-wall-stone-05` and `04` 15 to 17% (pale stone, 2.9 to 3.8 m). By surface: yellow 29 to
  38%, green 25 to 28%, blue 12%.

### Port: `ctyport`, the sea wall `portwall`

- **Look:** quays of cobbles on massive sea walls, warehouses of green-grey riveted metal on dark
  stone bases, towers with balconies, bridges on braces, copper conduits and small pipes everywhere.
- **Numbers:** shell 4,900 triangles, 13 m² each; kit 267,000 triangles, 8,160 copies of 102
  pieces, a piece 48 triangles median, 260 for the largest tenth. A building about 2,600 triangles,
  0.2 per m² of facade: the plainest walls of the city. The sea wall `portwall`: 4 pieces, 96 m
  modules of 50 triangles. Light 0.73.
- **Built:** small pipe rings (`city-port-smallpipe-ring-01`, 36 triangles, 990 copies), large
  beams (`city-port-beam-large-02`, 20 triangles, 23 m), two-bolt plates, metal blocks.
- **Palette:** `city-port-roofmetal` 20% (grey-green, 6.3 m), `city-port-metal-block-01` 7%
  (9 m), `city-port-copper-lines` 6%, `city-port-grnd-cobl-01` 5% (warm cobbles, 4.8 m),
  `city-port-dark-marble` 4% (dark green-grey, 8 m). The city wall: `citywide-wall-mainmetal`,
  `-frame`, `-orange-plain`, 9 to 16 m. By surface: grey 46%, green 27%.

### Industrial section: `ctyinda`, `ctyindb`

- **Look:** the biggest blocks of the city, 30 to 45 m high, flat metal walls with rounded
  corners and dark horizontal bands, bulging yellow-tan panel sides, red-lit bays, catwalks,
  pipes in bundles, vents with blue strip lights.
- **Numbers:** shell almost none (500 triangles): the walls themselves are pieces; kit 457,000 to
  480,000 triangles (the most of any district), 12,000 copies of 77 to 80 pieces, a piece 48
  triangles median, 360 for the largest tenth (the wall modules, 128 triangles, 32 to 43 m). Light
  0.66 to 0.70.
- **Built:** small pipe rings (`city-ind-smallpipe-ring-01`, 36 triangles, 2,720 copies), 45 and
  90 elbows, big pipe rings (72 triangles), wall modules, catwalk panels.
- **Palette:** `city-ind-wall-noisy-03`, `05`, `01` 22% (cold grey, 4 to 5 m),
  `city-ind-grnd-cobl-01` and `02` 20% (blue-grey paving, 8 m), `city-ind-dark-marble` 6 to 7%,
  `city-ind-wall-base-03` 4%, `city-ind-band-dark-01` (a 32 px band, 2.6 m). By surface: grey 83%:
  color comes only from the lit bays and the tan panels.

### Main Town: `ctygena`, `ctygenb`, `ctygenc`

- **Look:** tall rounded buildings of red-brown and silver panels in a frame of thick beams,
  overhanging bevelled tops, half-cylinder bays, archways, dark lower bands. Canals with railings
  and bridges. `ctygenb` is the Guard's outpost quarter: green-grey `city-outpostwall`, many
  pipes and lamps.
- **Numbers:** shell 1,000 to 11,500 triangles, 13 to 18 m² each; kit 232,000 to 269,000
  triangles, 6,400 to 10,400 copies of 111 to 120 pieces (the most varied kit), a piece 36 to 48
  triangles median. A building about 5,800 triangles, 0.7 per m² of facade. Light 0.61 to 0.71.
- **Built:** trims with bevels (`ctyn-trim-12-thick-w-bevel-a`, 48 triangles), pillars, archway
  blocks (60 triangles, 224 copies), wall gap fillers, canal rails and trapezoid blocks, bridge
  beams, wires across the streets (`city-mark-wire-01`, 6 triangles, 300 copies).
- **Palette:** `ctyn-brown-red` 13 to 17% (muted red-brown, 4.4 to 5.4 m), `ctyn-beams` 9 to 11%
  (silver-grey, 4.8 m), `ctyn-top-bevel-small-bottom` 8 to 11% (brown, 3.7 to 4 m), `ctyn-ground`
  9 to 10% (green-grey paving, 4.2 to 6.2 m), `ctyn-blue-trim-fancy-01`, `ctyn-black-wall-lower-01`
  (8 m). `ctygenb`: `city-outpostwall` 18% (green-grey, 8 m), `city-roofmetal`, `city-step`,
  `city-railing`. By surface: grey 45 to 57%, orange-brown 23 to 42%.

### Palace district: `ctypal`

- **Look:** a plaza of hexagonal paving around the palace's foot, ring walls with carved panels
  and gold-edged copings, statues of the Baron on pedestals, big planters with flower beds, trees
  with blocky canopies, Main Town's red-brown buildings around.
- **Numbers:** shell 14,200 triangles, 17 m² each; kit 222,000 triangles, 4,730 copies of 99
  pieces. Light 0.77.
- **Palette:** `city-pal-ground-01` 15% (4 m), `t-citypal-tree-top` 10% (yellow-green),
  `t-citypal-met-strp01` 8% (striped metal, 3.4 m), `ctypal-wall-tile-01` 7%,
  `t-citypal-met-wall-01` 5% (256 px, 13.7 m), with Main Town's `ctyn-*`.

### Gardens: `ctyfarma`, `ctyfarmb`

- **Look:** fields of yellow-green grass and dug plots in a ring of stone walls with capped tops,
  orchards of round fruit trees, dirt roads, sprinklers.
- **Numbers:** shell 5,500 to 6,200 triangles, 22 to 28 m² each (the largest triangles); kit
  342,000 to 392,000 triangles, 12,000 to 13,900 copies of 52 pieces, a piece 56 to 64 triangles;
  276,000 to 349,000 triangles of grass and pebbles. Light 0.80.
- **Built:** leaves (`city-farm-leaf-01`, 64 triangles, 1,600 copies), cattail grass (90
  triangles, 1,570), vegetable tree foliage, pebbles (84 triangles, 920), fruit.
- **Palette:** `farm-grass-ground-02` and `01` 25% (yellow-green, 8 m), tree tops 18%
  (`city-farm-treetop`, 2.5 m), `for-foliage` 5 to 7%, `city-farm-stonewall-bricks` and
  `stone-wall-01` 10% (pale stone, 4.2 to 5.6 m). By surface: green 47 to 50%, yellow 32 to 34%.

### The palace: `palent`, `palroof`, `palshaft`, `throne`

- **Look:** a tower of red-painted steel beams, rings, glass walls and big grey metal panels,
  pillar caps, horn-like spikes; the roof a plateau of grey slabs ringed by railings.
- **Numbers:** kit 230,000 to 271,000 triangles, 6,000 to 7,650 copies; the roof made of 74 roof
  pieces (24 triangles, 6,800 copies). Light 0.62 (`palent`) to 0.82 (`palroof`).
- **Palette:** `tent-red-paint-02` 18% (dull red, 5 m), `tent-red-pil-01` and `02` 27% (pinkish
  red, 6 to 7 m), `tent-wind-glass-02` 12% (pale glass, 27 m), `tent-beam01` 10% (1.9 m);
  the roof `tpal-panl_piller01` 19%, `tpal-beam01` 16%, `tpal-big-metal-panl01` 16% (12 m).

### Stadium and the JET-board park: `stadium`, `stadiumb`, `skatea`

- **Stadium:** marble and cobble plazas, white stone walls, stairs, statues, trees, orange sails
  over the stands (`citywide-sail-01`, 35 m). A piece 96 triangles median (the bulkiest kit).
- **JET-board park:** metal floors, grind rails (34 railing pieces, 1,345 copies, 48 triangles),
  stairs (1,150 copies), green pipes, stone edges. Light 0.83.

### Interiors

`hiphog` (the saloon), `hideout`, `kiosk`, `onintent`, `vinroom`, `oracle`, `gungame`, `garage`:

- Lit at 0.15 to 0.3 (warm under lamps, blue in Vin's room); their textures repeat every 0.5 to
  2 m (much denser than outside); a piece 40 to 190 triangles median (furniture and machines are
  the detail).
- The saloon: dark red roof metal, red checkered floor, yellow walls. The hideout: plaster, wood
  beams, plain metal, crates. Onin's tent: cloth, rugs, candles.

### The city from afar: `ctywide`, `palcab`, `palout`

Low-detail stand-ins seen across the city: never model detail for them.

## Before exporting a model made in Blender

- [ ] Its shell is a few large faces; its detail is the district's own pieces or small parts
      within the budgets of section 2.
- [ ] Triangles per m² of facade between 0.5 and 2 (`game_buildings.py --compare` says).
- [ ] Every face wears one of the district's textures, 64 or 128 px, repeated at the meters of
      section 4; doors, windows and signs laid once.
- [ ] Wear decals on every wall: streaks under openings and beams, stains at the foot.
- [ ] Colors only from those textures: muted; strong color only on small accents.
- [ ] No light baked, no vertex colors painted: the editor lights it.
- [ ] Edges bevelled in a few flat faces; in the slums and the bazaar, nothing exactly straight.
- [ ] Sizes at the game's scale: storeys 12 to 15 m, doors 4.5 x 7.5 m and up.
