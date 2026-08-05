# Giza Plateau — Old Kingdom Voxel Simulation

A single-file, fully procedural voxel reconstruction of the Giza necropolis as it
looked in the Fourth Dynasty: the three great pyramids still wearing their polished
white limestone casing and solid-gold pyramidions, the Great Sphinx in its quarried
enclosure, covered causeways running down the escarpment to the valley temples, and
a living Nile with quays, barges and a dense belt of date palms and papyrus.

Open `index.html` in any WebGL2 browser. Nothing to build, no assets to download —
Three.js is pulled in through an import map and every texture, mesh and colour is
generated in code.

## Controls

| | |
|---|---|
| Drag | orbit (damped) |
| Shift-drag / right-drag | pan |
| Wheel / pinch | zoom |
| `Dawn` `High Noon` `Golden Hour` `Starry Night` | lighting presets (keys `1`–`4`) |
| `Cycle` | slow automatic crossfade through the day (key `C`) |
| `Tour` | auto-flyover up Khafre's causeway from the Nile to the Great Pyramid (key `T`) |

The camera resumes a slow cinematic drift after five seconds of inactivity.

## How it is built

**Geography** (west → east) is one continuous height field, so the dunes, the
plateau, the escarpment and the river banks are a single watertight surface:
rolling ridged-noise dunes → the limestone plateau at y=43.75 → a terraced
escarpment → the flood plain → the Nile channel → the eastern desert. Every
structure declares a *flatten pad*, a feathered rectangle (or a ramp, for the
causeways) that blends the terrain to a level building platform and marks an
occupancy mask that vegetation avoids. Heights are quantised to 0.625 units and
jittered with high-frequency noise so the terracing dithers instead of drawing
contour lines.

**Scale** is uniform: 1 unit ≈ 2.6 m, which puts Khufu at 90 × 57 units
(234 m base, 148 m tall) and keeps palms, houses, barges and obelisks in
believable proportion to it.

**Rendering** is all instanced:

- `sand voxel` — one `InstancedMesh`, ~46 500 columns, per-instance vertical scale
  and vertex colour. A column reaches down to a common floor, so exposed height
  differences are always closed and nothing is ever drawn inside the terrain.
- `water voxel` — instanced slabs wherever the bed sits below the water line, with
  a per-instance depth attribute driving colour and opacity.
- `palm / papyrus / scrub / house / barge` — merged box geometry per variant, then
  instanced by the thousand.
- pyramids, temples, causeways, obelisks and quays share three cube
  `InstancedMesh`es (casing, masonry, gold). The pyramids are stacks of tapering
  slabs — hollow by construction, so there is no interior geometry to occlude.
- The Sphinx is voxelised from a union of implicit ellipsoids and boxes at a finer
  grid, then shell-culled: a voxel survives only if one of its six neighbours is
  empty.

**Light** ("the Ra system") interpolates every parameter of a preset — sun vector
and colour, sky gradient, hemisphere fill, fog, exposure, bloom, star density,
torchlight, water tint — so preset switches and the `Cycle` button are continuous
crossfades. Shadow map updates are suspended while the sun is still, which keeps a
4096² map over the whole necropolis essentially free.

The Nile carries a genuine planar reflection: the scene is re-rendered from a
mirrored camera clipped to the water plane, and the water shader projects that
texture back through the mirror's view-projection matrix, distorted by the wave
normal. Reflection resolution steps down automatically if frame times slip.
