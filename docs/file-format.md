# The Iskra project file

An Iskra project (`.iskra`) is one [SQLite](https://sqlite.org) database, so its contents can be
read with any SQLite tool even without Iskra. A library template (`.iskratpl`) is such a file
too: see [Library templates](#library-templates).

This document describes **format version 53**. Files of versions 1 to 52 are still opened: see
[Earlier versions](#earlier-versions).

```
MyFilm.iskra         the project: structure and every drawing
MyFilm.backup.iskra  a copy of the project as it was last saved (same format)
```

Pictures and sound brought into the project are not in the file: it refers to their files (see
[Media](#media)).

## Getting drawings out without the app

`iskra-extract MyFilm.iskra` writes every drawing as a PNG, plus `timing.json`, `palette.json`
and `styles.json`, into `MyFilm extracted/`. It also works on backups and on damaged files: it
reads what is still readable and lists what it had to skip. It never writes to the project file.

## Identification

| Pragma | Value | Meaning |
|---|---|---|
| `application_id` | `0x49534B52` ("ISKR") | The file is an Iskra project |
| `user_version` | `53` | The format version. Iskra refuses files with a higher version than it knows |

## Tables

### `document`

| Column | Type | |
|---|---|---|
| `name` | TEXT, primary key | `project`; a library template also has a row `template`, and a shot that was sent out a row `handOff` |
| `json` | TEXT | The project's structure as JSON |

The JSON holds everything except pixels: settings, the palette, the bins, the media, the
storyboards, the renders, and each shot with its nodes, each layer's drawing pool and exposure
track. Names are in camelCase. A value that is left out is at its default, which is given
below where it is not nothing, false or zero.

#### The project

- `settings`: `width` and `height` of a shot in pixels, `frameRate` as `{ "num", "den" }`
  frames a second, `drawingDensity` (the `density` new layers get), `workingSpace` (`srgb` or
  `linear`: how layers are blended and effects run) and `volume`, the master volume, 1 if left
  out.
- Ids of storyboards, shots, nodes, drawings, bins, media and sound clips are unique in the
  project and never reused. `nextId` is the next one to be given out. A palette's swatches,
  styles and named palettes have ids of their own (see [The palette](#the-palette)).
- `shots` lists every shot of the project, whether or not a storyboard uses it. Names are
  unique among shots, among storyboards and among media.
- The project's `bins` order its items, as folders order files: each is `{ "id", "name",
  "parent" }`, the `parent` being the bin it is in, or `null` for the bin everything starts
  in, which is called Master and is not listed. A shot, a storyboard or a media item is in the
  bin its `bin` names; one with no `bin`, or with one the project lacks, is in Master. Names
  are unique among the bins of one bin.
- `look` is a number that is another whenever the palette or the settings change. With a
  shot's `revision` it says whether a render is still of the project as it is. Only whether
  it is the same matters, not its value.

#### The palette

`palette` is `{ "swatches", "nextSwatch", "styles", "sets", "skins" }`.

- A swatch is `{ "id", "name", "color", "set" }`. Its `color` is sRGB with straight alpha,
  `[r, g, b, a]`, 0–255. Drawings name swatches by id, from 1; 0 is no swatch. A swatch may
  have `drift`, `[hue, saturation, value]`: how far a vector stroke's colour drifts from it at
  most, its hue turned by so many degrees (-180 to 180) and its saturation and value moved by
  so many hundredths (-100 to 100), added to its colour in HSV.
- No new swatch gets an id below `nextSwatch`, so the id of a removed swatch, which drawings
  may still be painted with, is not given again.
- `sets` are the named palettes, in the order they are listed, each `{ "id", "name" }`; there is
  always at least one. A swatch or a style is of the named palette its `set` names, 0 (the
  one first called *Default*) if left out. Swatches of all of them are in the one list, in
  the order each has in its own.
- A style is `{ "id", "name", "colors", "set" }`: other colours for its named palette's
  swatches, as *Night* is to *Day*. `colors` maps a swatch id (as text) to a colour; a swatch
  it gives none has the palette's own.
- `skins` are the project's skins, each `{ "id", "name" }`, names unique among them: named sets
  of drawings shown in place of the day ones, as a style is colours in place of the swatches'
  own. A layer keeps its drawings in a skin as `variants`, and a shot says which skin it is
  shown in (see [Shots](#shots) and [Layers and drawings](#layers-and-drawings)). A Style node
  shows what goes through it in the skin of its style's name too, so the two are found by the
  same names. Left out, there are none.

#### Storyboards

- A storyboard's `clips` say where its shots sit, in film order: `{ "shot", "start", "in",
  "out" }` puts the shot's frames `in` up to (not including) `out` on the storyboard from frame
  `start`. Clips never overlap, and a shot has at most one clip in the whole project.
- A storyboard's `transitions` are `{ "shot", "edge", "kind", "length" }`: a transition of
  `length` frames on the `start` or `end` of that shot's clip. Where another clip meets that
  edge the transition lies across the cut, centred on it. `kind` is `crossDissolve`,
  `dipToBlack` or `dipToWhite`.
- A storyboard's `start` is the number its first frame has when its times are told as
  timecode: 0, or for a film that starts at one hour at 24 frames a second, 86400.
- `sound` lists its sound tracks, each `{ "name", "mute", "clips" }`. A sound clip is
  `{ "id", "media", "start", "offset", "length", "gain" }`: the sound item `media` plays from
  frame `start` of the film for `length` frames, starting `offset` frames into the sound, at
  `gain` times as loud as recorded. The clips of a track don't overlap and are in the order
  they play.
- `markers` are marks on frames, in frame order, one a frame: `{ "frame", "name", "colour" }`,
  the colour sRGB `[r, g, b]`.
- `range` is `[first, after]`: the stretch that is played and looped, from its first frame to
  the one after its last. Without one, all of it is. It is for playing only.

#### Shots

- A shot's `duration` is its whole length in frames, handles included.
- A shot's `status` says where it is in its life: `board`, `out`, `animation`, `post` or
  `final`. A shot that is `out` was sent to someone as a file of its own (see
  [Shots sent out](#shots-sent-out)); its `outTo` is who has it, if that was said.
- A shot's `scene` and `sequence` are names it is given to organise the shots by; a shot
  without one is in none. They hold nothing the shots of a scene or a sequence share.
- A shot's nodes are its `layers`, `pegs`, `composites`, `deformers`, `effects`, `groups` and
  `displays`, each list in the order they were made; that order means nothing to the picture.
  No two of a shot's nodes share a name.
- `revision` is a number that is another after every change to what the shot looks like (see
  [Renders](#renders)).
- `places` say where nodes have been put in the node view: a node's id (as text, since it is
  a JSON key) to `[x, y]`, y down. The main display's place is under the shot's own id. A node
  with no place is laid out by the view. `colours` are colours given to nodes there, by id, as
  sRGB `[r, g, b]`; a node with none has the colour of its kind. Neither changes the picture.
- `styles` are the palette styles the shot is shown and rendered in: at most one of each named
  palette. A named palette with none here shows its own colours.
- `skin` is the id of the skin the shot is shown and rendered in (see [The palette](#the-palette)):
  each layer's variant in that skin shows in place of its day drawing, and a drawing with no
  variant in it shows as it is. Left out, the shot shows its day drawings.
- `guides` are drawing guides, each `{ "id", "name", "kind", … }`: a `ruler` has `at`
  (`[x, y]`) and `angle` (degrees), a `perspective` has `points`, one to three vanishing
  points. Places are in the shot's pixels, from its middle.
- `panels` are the shot's storyboard panels after its first, one a frame, in frame order:
  `{ "frame", "action", "dialogue", "notes" }`. The first panel, on frame 0, is listed only
  once something is written of it.
- `markers` and `range` are as a storyboard's, in the shot's frames. A render is of the cut,
  whatever the range.
- What the timeline shows: `folded` lists the pegs whose rows are folded, `hidden` the layers,
  pegs and groups whose rows are hidden, and `unlocked` the board layers opened to be drawn on
  on the Animation page, where boards are otherwise locked. Ids of deleted nodes may stay in
  these lists and name nothing. A hidden layer is not in what the shot shows, nor is one that
  a hidden peg carries or that is in a hidden group.

#### Layers and drawings

- A layer's `role` is `board` or `animation`. Board layers are what the storyboard shows;
  renders use the animation layers.
- A layer's `density` is its drawing pixels per scene unit; a scene unit is one shot pixel.
- A layer's `drawings` is its pool, each `{ "id", "name", "drawing" }`; names are unique in the
  pool. Its `variants` are its drawings in skins, each `{ "skin", "of", "drawing" }`: in the
  skin with the id `skin`, the drawing `drawing` (a pool drawing of its own, `{ "id", "name",
  "drawing" }`, whose tiles are kept under its own id) shows in place of the pool's drawing
  `of`. A drawing has at most one variant in a skin. The exposure always names the day
  drawing, so a change of skin retimes nothing. A drawing's `artLayers` are listed bottom first: `colourArt`, `lineArt` over it, and
  `overart` over the lines. Each has a `kind` (`colourArt`, `lineArt`, `overart`) and a `content` mode (`indexed`, `rgba` or `vector`) that says how its tiles are
  to be read, or for `vector` that its shapes are in the table [`vector`](#vector).
- A layer's `exposure` is a list of runs `{ "start", "len", "drawing" }`: the drawing with
  that id shows for `len` frames from frame `start` (frames count from 0). Frames no run covers
  are empty.
- `freeColour` lists the kinds of art layer that are painted in free colour (`rgba`) in the
  layer's drawings, and not with the palette; a new drawing is made so. A bitmap layer made
  since 2026-10-27 lists all three; one left out is swatch-linked pixels. `vector` lists the kinds
  that are vector art, and a new drawing is made so too. `artOff` lists the
  kinds that are switched off: not seen, in the window or in a render. `artPorts`, when `true`,
  has the layer's node show an output for each art layer in the node view (outputs that carry a
  wire are shown whatever it says).
- A layer made by placing pictures that were brought in names that media item as its
  `media`. The pictures are the layer's own, in tiles like any drawing; the item is kept to
  read them again when its file changes. `opaque` leaves their alpha out, and `alpha` says
  how their files keep their colours with it: `straight` (if left out),
  `premultipliedBlack` or `premultipliedWhite`.
- A layer's `deformer` is the deformer of the shot that bends its drawings, if one does.

#### Pegs

- A shot's `pegs` are transforms that carry what is parented to them. A layer's, a peg's or
  the camera's `parent` is the id of a peg of the same shot, or `null`. A peg's `transform` has
  `position` (`[x, y]`, how far it is moved), `rotation` (degrees, clockwise on screen),
  `scale` (`[x, y]`, 1 is the drawn size), `skew` (degrees) and `pivot` (`[x, y]`). A point `p`
  of what is under the peg is put at `position + pivot + R·K·S·(p − pivot)`: scaled, then
  skewed (`x` grows by `y·tan(skew)`), then rotated, all about the pivot. Lengths are in scene
  units, y down. A node under several pegs gets the transform of each, the nearest first.
- A peg's `transform` also has a `depth`: how far it moves what is under it towards the
  viewer, in scene units, below zero for away. The depths of the pegs above a layer add up.
  Layers are drawn in the order of their depth, the farthest first, unless a composite stacks
  them otherwise (see [Composites](#composites-displays-and-effects)); layers at one depth are
  drawn as the composites stack them.
- `tilt` (`[x, y]`, degrees, about the pivot) is how far the top of what the peg carries leans
  away from the viewer and how far its right side swings away. Not zero, the camera sees it
  in perspective.
- A peg's `animation` has a list of keyframes for each of `position`, `rotation`, `scale`,
  `skew`, `depth` and `tilt`; the pivot is not animated. A key is `{ "frame", "value",
  "into", "out" }`, the keys of a list are in frame order, and `into` and `out` say how the
  value arrives at the key and leaves it: `linear`, `ease` (coming to rest at the key), for
  `out` also `hold` (it stays until the next key, then jumps), or `{ "curve": { "reach",
  "speed" } }`. A curve is one handle of the Bézier that the value follows over time between
  two keys. With the time and the value between the two keys both measured from 0 to 1, the
  handle of the first key's `out` is at (`reach`, `speed`·`reach`) and that of the second
  key's `into` at (1 − `reach`, 1 − `speed`·`reach`). So `reach` is how far the handle reaches
  towards the other key and `speed` how fast the value moves at the key, 1 being the steady
  speed. `linear` is a `reach` of ⅓ at `speed` 1 and `ease` a `reach` of ⅓ at `speed` 0. For
  a pair of numbers each follows the same curve; for the position it is the distance along
  the path.
- A key may have a `chart`, the timing chart of the way to the next key: a list of points
  `[time, along]`, both from 0 to 1, in order, `time` the part of the time between the two keys
  and `along` how far the value is on the way then, never going back. With a chart, the `out`
  of the key and the `into` of the next are not used: between the keys the timing is a smooth
  curve through `[0, 0]`, the points and `[1, 1]` that never goes back (a monotone cubic, after
  Fritsch and Carlson). The value on a frame is the value the way would have, with both sides
  `linear`, at the time when it is as far along as the curve says there; that time can fall
  between frames. So for the position the path stays as its keys and bends make it, and only
  the speed along it changes. A chart's points are usually one for each frame between the
  keys, but need not be: a key moved keeps its chart, stretched to the new time. Out of a
  `hold` the chart is not used.
- A value with keys is what they make it on each frame: the first key's value before it, the
  last key's after it, and that value of `transform` is then not used. A value with no keys
  is the one in `transform` on every frame. Every other keyed value in the file (a lens, a
  pose, a parameter) has keys of this form and reads the same way.
- The position moves along a curve through its keys, not from key to key in straight lines:
  between two keys the curve is a cubic Bézier whose control points are the first key's value
  plus its `out` handle and the second key's value plus its `into` handle. A position key with
  a `bend` (`{ "into": [x, y], "out": [x, y] }`) has those handles; one without passes
  smoothly, its handles lying along the line between the keys before and after it, each a
  third as long as the way to that neighbour. The speed along the curve is what the eases make
  it.
- A peg that `follows` a point of a deformer, `{ "deformer", "point", "bends" }`, goes with that
  point as the deformer is posed, under its parent and before its own transform. `point` is the
  joint of a chain or the point of a curve, 0 where it starts; left out, it is the end. With
  `bends` true, the drawings right under the peg are bent by the deformer as its own layers'
  are, taken into its space at rest, and are not carried by the point; the pegs under it are.
  Left out, it is false: nothing the peg carries is bent.
- `inherit` (`{ "position", "rotation", "scale" }`, each true if left out) says what the peg
  takes from its parent and from the point it follows. Without the position its pivot stays
  where its own values put it under the parent's parent; without the rotation or the scale it
  is not turned, or not scaled, by the parent, though it still goes where that puts it. The
  pegs above the parent always count.
- `matches`, on the peg at the top of a rig, maps names of library templates' nodes to the
  names of the rig's that they were matched to by hand.

#### Deformers

A deformer bends the drawings of the layers given to it. It is `{ "id", "name", "origin",
"offset", "offsets", "bones", "stiffness", "spine", "closed" }`.

- A chain of bones starts at `origin` and is moved from there by `offset` (`offsets` are its
  keys). Each bone starts where the one before it ends.
- A bone has `restAngle` (degrees from the bone before, or for the first from the x axis) and
  `length`; `turn` (degrees from its rest, keys `turns`) and `stretch` (a part of its length,
  1 if left out, keys `stretches`) pose it. `hold` (1 if left out) is how strongly it holds the
  drawing against the other bones. The joint it starts at is round within `round` scene units
  and fades out at `fade`; `pull` (−1 to 1) draws what is in the joint towards the middle of
  the bend, and `crease` (0 to 1, 1 if left out) how sharp the inside of the bend is.
- `stiffness` is how sharply the bones' hold falls off with distance.
- A deformer with a `spine` is a curve and has no bones; with `closed` its line closes on
  itself as an envelope and holds what is within it. A point of the spine (a knot) lies `at`
  `[x, y]` at rest and is `moved` from there (keys `moves`). The drawing within `reach` of the
  line follows it wholly, less beyond, and not at all from `fade`; both at 0 have no limit. A
  knot's `bend` gives the line's handles through it at rest, as a position key's does, and
  `bent` moves them (keys `bendsInto` and `bendsOut`, on the same frames).

#### Composites, displays and effects

A node that makes a picture is named, where it is wired in, as `{ "layer": id }`,
`{ "composite": id }` or `{ "effect": id }`. A layer's wire leaves from its flat output, all its
drawings' art layers as one picture, or as `{ "art": [id, kind] }` from the output of one art
layer (`colourArt`, `lineArt`, `overart`), which gives that art layer alone.

- A composite stacks its `inputs`, each at most once. Its `stacking` is `depth` (if left out:
  what it stacks is ordered by depth among all else the shot shows), `depthFlat` (by depth
  among themselves, as one flat card) or `ports` (by the order of its inputs, as one flat
  card). The first input is on top, or with `rightFirst` the last.
- A shot's `display` is what its main display shows, or `null`; `displayName` is that
  display's name, `Display` if left out. Its `displays` are other displays, each
  `{ "id", "name", "shows" }`, for the viewer alone. A display with `"pass": true` is a pass:
  what it shows is rendered apart on the Export page, under its name. What a shot shows is its
  main display's node, with every composite and effect opened up into what it takes. A node
  that is connected to nothing the main display shows is in the file and not in the picture.
- An effect is `{ "id", "name", "kind", "inputs", "params", "enabled", "parent" }`. `kind` names
  what it does: `blend`, `cutter`, `glow`, `dropShadow`, `tone`, `highlight`, `blur`,
  `directionalBlur`, `radialBlur`, `colourCorrection`, `transparency`, `visible`, and the
  structure kinds `portIn`, `portOut`, `send` and `receive`. A kind a build doesn't know is
  kept and lets its picture through. `inputs` maps the names of its ports (`picture`, `matte`)
  to what is wired there. `params` maps parameter ids to `{ "value", "keys" }`, the value one
  of `{ "number": n }`, `{ "choice": "…" }`, `{ "switch": b }`, `{ "colour": [r, g, b, a] }`
  (0–1), `{ "text": "…" }` or `{ "curves": { "master", "red", "green", "blue" } }` (each a
  list of `[in, out]` points from 0 to 1); a number or a switch (1 on, 0 off) can have keys.
  A parameter with no entry is as its kind starts it. Switched off (`"enabled": false`) an
  effect lets its picture through unchanged. A port or a Send whose `parent` names a peg
  carries that peg's transform: what is parented to the effect's id is carried by that peg.
- A group is `{ "id", "name", "members", "folded", "board" }`: `members` are ids of the shot's
  nodes, each node in one group at most; deleted ones may stay. Folded, it is shown as one box
  and one row. A `board` group is the storyboard's: everything in it is a board, whatever its
  layers say.
- A layer, peg, composite, effect or group that is a copy from a library template has the
  template's name as its `template`. It is a note for the user and nothing follows from it.

#### The camera

A shot's `camera` is what the shot is seen through: `{ "id", "parent", "distance", "zoom",
"animation" }`. Its `parent` is the peg that moves it, or `null` for a camera at home. A camera
at home looks straight at the middle of the scene from `distance` scene units in front of the
plane at depth 0, and sees that plane as it is drawn: a scene unit there is a pixel of the
shot. What is at depth `d` is seen `zoom · distance / (distance + c − d)` times its drawn size,
about the point the camera looks at, where `c` is the depth of the camera's pegs, added up;
what is not in front of the camera is not seen. The camera's pegs move it as they move a
drawing: the picture is what a drawing is, seen from the camera's place. So `distance` is the
lens: the field of view, top to bottom, is `2 · atan(height / (2 · distance))`. Its
`animation` has keys for `zoom` and `distance`.

#### Media

`media` lists the pictures and sound brought in, each `{ "id", "name", "bin", "path", "kind" }`.
`path` is the file it refers to; a file in the project's folder is named from that folder, so
the folder can be moved with its media. `kind` is one of:

- `{ "image": { "width", "height" } }`;
- `{ "imageSequence": { "width", "height", "frames" } }`: numbered pictures, one a frame, the
  path naming the first; a number with no file is a frame that shows nothing;
- `{ "sound": { "sampleRate", "channels", "frames" } }`, `frames` being samples a channel.

A sound can have a `transcript`, what is spoken in it in order: `{ "text", "start", "end" }`,
in thousandths of a second from the start of the sound.

#### Renders

`renders` records the renders that were made, each `{ "what", "at", "settings", "mark",
"when", "pass" }`. `what` is `{ "shot": id }` or `{ "film": storyboard id }`; `at` is the movie
file or the sequence's folder; `settings` are the render settings as the export wrote them;
`when` is in seconds since the start of 1970, UTC. `pass` is the pass it was, by its name, or
`Final` for the whole picture beside them; left out, the shot was rendered as itself. `mark`
says how what it shows was: `picture` (the shot's `revision`, or for a film a number made of
its shots' and where they lie), `look` (the project's), `frames` (`[first, after]`) and
`sound`. A render whose mark the project no longer gives is out of date.

### `tile`

Drawings are sparse planes of 256 × 256 pixel tiles. Only tiles with something painted on them
have a row.

| Column | Type | |
|---|---|---|
| `drawing` | INTEGER | The drawing's id |
| `layer` | INTEGER | Index into the drawing's `artLayers` (0 is the bottom layer) |
| `tx`, `ty` | INTEGER | The tile's position: it covers pixels `tx*256 … tx*256+255` across and `ty*256 … ty*256+255` down. Both can be negative |
| `data` | BLOB | The tile's pixels, compressed |

The primary key is (`drawing`, `layer`, `tx`, `ty`), so the tiles of one drawing, and with the
structure's list of a shot's drawings those of one shot, can be read without reading the rest.
Iskra itself reads a shot's tiles only when the shot is opened.

`data` is a 4-byte little-endian length (always 262144) followed by an
[LZ4 block](https://github.com/lz4/lz4/blob/dev/doc/lz4_Block_format.md). Decompressed, it is
256 rows of 256 pixels, top row first, 4 bytes a pixel:

| Layer mode | Pixel bytes |
|---|---|
| `indexed` | Swatch id as a little-endian `u16`, then coverage as a little-endian `u16` (0 = unpainted, 65535 = fully covered). The pixel's colour is the swatch's colour, in the style the shot shows its named palette in, with its alpha scaled by the coverage. An unpainted pixel is all zero. An id from 32768 to 65534 is a swatch's colour drifted: the swatch is its lowest 11 bits, and the next 4 how many fifteenths of its `drift` the colour has moved |
| `rgba` | Red, green, blue, alpha, one byte each, sRGB with straight alpha |

### `vector`

An art layer of vector art (`content` `vector`) has no tiles: its shapes have one row here.

| Column | Type | |
|---|---|---|
| `drawing` | INTEGER | The drawing's id |
| `layer` | INTEGER | Index into the drawing's `artLayers` |
| `data` | BLOB | The shapes, compressed |

The primary key is (`drawing`, `layer`). `data` is a 4-byte little-endian length followed by an
LZ4 block, as a tile's is; decompressed, it is UTF-8 JSON: a list of shapes, bottom first, each
painted over those before it. Every shape has an `id` (unique in its art layer), a `swatch`
whose colour it is painted in, and a `shape`:

- `stroke`: a line drawn with a brush. `path` is its middle line as
  [SVG path data](https://www.w3.org/TR/SVG/paths.html#PathData), one open path, in the
  drawing's pixels (as tiles count them, y down). `widths` has the stroke's width at each point
  of the path in order, one for each command that ends at a point; between two points the width
  goes evenly from one to the other. The stroke covers what a round nib of that width covers
  going along the path, its ends round.
- `fill`: an area. `outline` is its outline as SVG path data, filled by the non-zero rule. A
  fill the paint bucket made reaches the middle of the strokes round it. `inside`, if there, is
  the point it was filled at, `boundBy` the strokes that bound it, each `{ "art", "shape" }`:
  the kind of the drawing's art layer it is on and its `id` there; `gap` the width of the gaps
  in those lines that were closed. When one of those strokes changes, Iskra finds the area round
  `inside` again and gives the fill its outline; `open` marks a fill whose lines no longer close
  round it, which keeps the outline it had.
- A stroke drawn with an oval nib has `nib`: `{ "angle", "roundness" }`, the oval's width turned
  `angle` degrees from the x axis, and `roundness` times that across. Without it the nib is
  round.
- A stroke whose ends are not both round has `caps`, `[start, end]`, each `"round"`,
  `"straight"` (square across at the end of the path) or `"miter"` (square, reaching past the
  end by half the width there). The Cutter tool gives the ends it makes the one asked for.
  Without it both are round.
- A stroke drawn with a vector brush that has a texture or a second tip has `grain`:
  `{ "texture", "dualBrush", "seed" }`, the brush's texture and dual brush settings as a pixel
  brush keeps them, and the seed of what chance does to the second tip. Its coverage is cut by
  them when it is drawn as pixels, as the pixel brush's paint is.
- A fill drawn as a surface by a vector brush that has a texture or a second tip has
  `tracks`: the strokes it was drawn from, each as a `stroke` is written, with its `grain`. Its
  coverage is cut by the grain of each, the most any of them leaves, as a stroke's is by its
  own. Surfaces of one swatch and one grain merged into one keep the tracks of all of them.
  Without it a fill has no grain.
- A stroke whose opacity changes along it has `opacities`, one for each point as `widths` has,
  each from 0 (none) to 1 (opaque), going evenly from point to point. Its coverage is that much
  of what it covers: at each pixel the most of the places along it whose nib reaches the
  pixel. Without it a stroke is opaque all along.
- A stroke whose colour drifts has `drifts`, one for each point as `widths` has, each from 0 to
  1: how much of its swatch's `drift` its colour has moved there, going evenly from point to
  point, in fifteenths when drawn as pixels. A swatch with no `drift`, or one of id 2047 or
  more, keeps its colour. Without it the colour does not drift.
- A stroke, or a fill drawn as a surface, that boils has `boil`: `{ "amount", "every" }`. Every
  `every` frames of the shot it is drawn the next of three ways in turn: the first as it is
  kept, the other two with its points moved by up to `amount` pixels by a smooth field of its
  own and its grain laid anew. Without it the shape is still.
- A fill drawn as a surface by a vector brush whose opacity or colour changes has its stroke in `tracks`
  too, with its `opacities`, and is as faint as the track is near each pixel (the track's nib
  taken nine tenths wider, for a rough edge). Such a surface is never merged with another.

## Library templates

A template (`MyRig.iskratpl`) holds some nodes of a shot, to be put into other shots as copies.
It is a project file, with everything above: its project has one shot, named as the template,
which holds the nodes with their drawings and animation, and a palette of the swatches those
drawings use. So `iskra-extract` works on a template, and a template opens as a project.

The shot's `display` shows a composite that stands for the shot the template is put into:
that composite's `inputs` are what gets connected there, on top of what the shot shows, and
the composite itself is not copied. Putting a template into a shot gives every copy a new id.
A swatch is added to the project's palette only if the palette has none with that id.

A template has two things more than a project:

- A row `template` in `document`, whose JSON says what the template is: `source` (the name
  of the project it was saved from), `saved` (when, in seconds since the start of 1970, UTC),
  `tags` (a list of words) and `notes`.
- A table `preview` with a small picture of the template, to show in a list without reading
  its drawings:

| Column | Type | |
|---|---|---|
| `frame` | INTEGER, primary key | The frame the picture is of. For now there is one row, `0` |
| `png` | BLOB | The picture as a PNG file, at most 192 pixels a side, framed to what is painted |

A template is written whole, beside its place first and then moved into it, and has no backup.

## Shots sent out

A shot is handed to someone as a project file of its own (`Shot_01.iskra`): a project with
that one shot, its drawings, the settings and the whole palette of the project it is from,
and a storyboard whose one clip has the shot's in and out points in the film. It is opened,
worked on and saved like any project.

It has one thing more: a row `handOff` in `document`, whose JSON says where the shot is from:
`project` (the project's name), `shot` (the shot's id there), `name` (its name there when it
was sent), `sent` (when, in seconds since the start of 1970, UTC) and `to` (who it was sent
to). Saving the file keeps the row; saving it as another file does not, and the shot is then
found again by its name.

When the file comes back, its shot takes the place of the shot it came from. Everything in it
gets a new id, since ids given out in the file meanwhile may have been given to other things in
the project; the shot keeps its own id and name, and its clip is left as it is. The project's
settings are kept, and its palette gets only the swatches it has none of by id.

## Saving

A save is one SQLite transaction, so after a crash or power cut the file holds either the
previous save or the new one. A save writes only the tiles that changed, and the vector art of
the art layers whose shapes changed.

## Earlier versions

A file of an earlier version is converted as it is read. The file itself is rewritten in the
current version by the next save, which first keeps the old file as `MyFilm (format 1).iskra`
(or `2` to `52`) beside it. Where a version below lacks something, that is left out, and reads
as its default, unless more is said.

- **Version 52** had no `scene` nor `sequence` on shots.
- **Version 51** had no `chart` on keys: every way between two keys is timed by their eases.
- **Version 50** kept the list of skins on the project, as `skins` beside `palette`; it is
  moved into the palette as it is read.
- **Version 49** had no skins: no `skins`, no `skin` on shots and no `variants` on layers.
- **Version 48** had no `bends` on a peg's `follows`: what a peg that follows bones carries is
  carried, not bent.
- **Version 47** had no `artPorts` on layers, and no `art` wires: every wire from a layer
  leaves from its flat output. It had two art layers a drawing, no `overart`. Each drawing gets
  one, empty, after its others: vector art where they are all vector, indexed otherwise; a layer whose `vector`
  lists both the colour art and the line art lists the overart too.
- **Version 46** had no vector art: no table `vector`, which the first save in version 47
  makes, and no `vector` on layers.
- **Version 45** had no `pass` on renders: every render is of a shot or film as itself.
- **Version 44** had passes as composites marked `"pass": true`, and the project a list
  `passes` of their names. Each marked composite gets a pass display of its name that shows it
  (in its group, if it is in one), and the composite is renamed `<name> Composite`. The list
  goes.
- **Version 43** had no `opaque` or `alpha` on layers.
- **Version 42** had no `artOff` on layers.
- **Version 41** had no `unlocked` on shots: every board is locked on the Animation page.
- **Version 40** had no `displays` or `displayName`: a shot has its one display.
- **Version 39** had no `matches` on pegs.
- **Version 38** had no `inherit`: a peg that `follows` said with `by` what it takes of the
  bones' pose. `position` becomes no rotation or scale inherited, `positionRotation` no scale.
- **Version 37** had no `follows` on pegs.
- **Version 36** had no `curves` parameters.
- **Version 35** had no `hidden` rows on shots and no `volume` in the settings.
- **Version 34** had no `text` parameters and no `parent` on effects.
- **Version 33** had no `stacking` on composites: all stack by depth.
- **Version 32** had no `effects`.
- **Version 31** had no `markers`, `range` or `folded` on shots.
- **Version 30** had no `markers` or `range` on storyboards.
- **Version 29** had no `transcript` on sounds.
- **Version 28** had no `panels`: a shot has its first, with nothing written.
- **Version 27** had one `style` on a shot, or none, where now there is a list `styles`; it
  becomes the list's one style.
- **Version 26** had no `sets`: every swatch is in the one named palette, *Default*.
- **Version 25** had no `freeColour` on layers.
- **Version 24** had no `guides`.
- **Version 23** had no palette styles.
- **Version 22** had a list of swatches as its `palette`: it becomes the palette's `swatches`.
- **Version 21** had chains of bones only, without `stretch`, `round` and the rest.
- **Version 20** had no `deformers`.
- **Version 19** had no passes.
- **Version 18** had no `renders` and no `revision` on shots: nothing reads as rendered.
- **Version 17** had no `sound` tracks on storyboards.
- **Version 16** had no `media`.
- **Version 15** had no `animation` on cameras.
- **Version 14** had no `tilt` on pegs: every peg faces the viewer squarely.
- **Version 13** had no `bins`, no `bin` on items and no `start` on storyboards: everything is
  in Master, and every storyboard starts at 0.
- **Version 12** had no `camera`: each shot gets one at home, with a field of view of
  41.112°, which sees the shot as it was.
- **Version 11** had no `depth` on pegs: every peg is at depth 0.
- **Version 10** had no `status` on shots: every shot reads as a `board`.
- **Version 9** had no `template` on nodes; nothing else differs.
- **Version 8** had no `places`: every node is laid out by the node view.
- **Version 7** had no `composites` and no `display`: the order of a shot's `layers` was
  which is over which, and all were seen. Each shot gets one composite named `Composite`
  that stacks its layers in that order, shown by its display.
- **Version 6** had no `curve` sides on keys: a side was `linear`, `ease` or `hold`.
- **Version 5** had no `bend` on position keys: every key was a smooth one.
- **Version 4** called a shot's `layers` its `elements`; nothing else differs. (A drawing's
  `artLayers` have always had that name.)
- **Version 3** had no keyframes: its pegs have no `animation`, which reads as no keys.
- **Version 2** had no pegs: its shots get an empty `pegs` list and its elements no `parent`.
- **Version 1** had no `storyboards`: `shots` was the film, in order. Its shots are laid end to
  end on one storyboard named `Storyboard`, each whole (`in` 0, `out` its duration). Elements
  had no `role`; they become `animation` layers.

## Backup

Every save the user makes ends by copying the project file, as just saved, to
`MyFilm.backup.iskra` beside it, replacing the copy of the save before. The app's own autosaves
write the project file and leave the backup alone, so the backup is the version last saved by
hand. If the project file is damaged later, Iskra offers to
open the backup as `MyFilm (recovered).iskra`. The backup is an ordinary project file and can
also be opened or extracted directly.
