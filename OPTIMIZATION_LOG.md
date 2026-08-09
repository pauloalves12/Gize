# Visual optimisation log

Each round was rendered at fixed camera poses and compared side by side against
the previous iteration. Changes that did not clearly improve the image were
corrected or dropped, and both are recorded here.

Baseline for this work is commit `f5b446d`.

### On the numbers in this log

Two measurement problems turned up and are worth stating plainly, because they
limit how much the figures below are worth.

1. **The captures were not deterministic at first.** Software rendering exceeds
   the 30 ms frame budget on every frame, so the adaptive quality ladder switched
   GTAO off partway through a capture. The same pose produced a mean of 117.8 in
   one run and 131.0 in the next with only a bird-scale change between them. The
   rig now freezes the ladder before capturing, and the final comparison renders
   the baseline from `f5b446d` through the same frozen path.
2. **Luminance standard deviation is confounded by anti-aliasing.** Jagged edges
   are high-frequency contrast, so MSAA lowers both the contrast figure and an
   adjacent-pixel detail measure while making the image objectively cleaner. On
   the deterministic pair, contrast reads 57.0 → 55.1 and local detail −9%, yet
   the optimised frame plainly carries more information.

The per-round figures below therefore record what was observed at the time and
what triggered each correction; the side-by-side image was always the decision.

---

## Round 1 — light, atmosphere, occlusion

**Added**

- **Baked sky occlusion.** Every terrain column samples 12 azimuths × 8
  distances to find its blocked horizon, and the resulting factor darkens and
  slightly cools the hollow. It runs once at build time and costs nothing per
  frame. This is what puts dune troughs, the Sphinx quarry and the foot of the
  escarpment into ambient shadow instead of lighting them like open ground.
- **GTAO** for the contact occlusion a shadow map cannot resolve: masonry
  joints, terrace risers, temple setbacks, the seam where a building meets sand.
- **Height fog with forward scattering.** The three fog shader chunks are
  overridden globally, so every lit material gains it at once: density climbs
  toward the valley floor, and looking into the sun warms the haze.
- **Real MSAA** in the composer. The renderer's `antialias` flag only covers the
  default framebuffer, so with post-processing every voxel edge was aliased.
- **Per-fragment surface grain** from world-space noise on sand, masonry and
  casing — roughness, tint and a little normal variation.

**Measured**

| Preset | mean | contrast | verdict |
|---|---|---|---|
| High Noon | 178.8 → 165.0 | 36.4 → **40.1** | clear gain: less clipping, plateau gains structure |
| Dawn | — | — | clear gain: escarpment reads as strata |
| Golden Hour | 121.8 → 126.3 | 56.0 → **49.9** | **regression** — fog washed the contrast out |
| Starry Night | ground 65.9 → **120.3** | 35.4 → 57.0 | **regression** — mood lost |

**Two regressions corrected rather than kept**

1. *Ground fog reached too high.* The boost decayed at 0.030/unit, so at plateau
   height (y=43) it still nearly doubled the haze. Steepened to 0.085 and the
   per-preset densities trimmed. Golden Hour contrast returned to 56.0 while
   keeping every structural gain — confirmed in a three-way strip.
2. *Fake bump lifted the night.* Perturbing normals without real height pays off
   under a high sun but breaks at night: the near-grazing quarry walls tilted
   facets into the torches and the whole basin brightened by 83%. Isolated by
   elimination — disabling GTAO *darkened* the frame by 12.7, as a multiply
   must, which left the normal term as the only candidate. Amplitudes roughly
   halved; the grain that reads as material variety stayed.

---

## Round 2 — geometry

**Added**

- **Mastaba cemeteries.** Grid-aligned rows of battered, cornice-crowned tombs
  with a false door and offering slab on the east face, in four fields on the
  plateau shelf. The plateau was three pyramids in empty desert; it now reads as
  an occupied necropolis.
- **Cavetto cornice and torus roll** on every temple wall and pylon. This
  silhouette is what makes a stone mass read as an Egyptian building rather than
  a generic block.
- **Proper columns**: torus base, tapered shaft, lotus-bud capital, abacus and
  architrave stub, replacing plain square pillars.
- **Seated colossi** flanking the valley-temple gateways.
- **Cultivated plots** — a basin-irrigation patchwork of emmer, barley, flax,
  greens and fallow with darker wet dykes between them, so the flood plain is no
  longer one undifferentiated green.
- **Wind.** Palms and reeds sway with amplitude scaling by local height.
  Motionless vegetation was the single biggest "this is a model" tell.
- **Wind-carved boulders** on the escarpment, **lotus rafts** in the still
  shallows, **ibis flocks** wheeling over the marshes.

**Two regressions corrected rather than kept**

1. *Boulders rendered as white litter.* `matStone` drives instance-coloured
   meshes and carries no `vertexColors` flag, so merged boulder geometry lost its
   tint entirely — and at 2.4× scale they were 17 m lumps. Moved to the
   vertex-coloured material and scaled to 0.6–1.9 m.
2. *Mastabas scattered at random.* A real cemetery is laid out on a grid. Random
   placement read as white specks; cardinal-aligned rows with small jitter read
   as planned burial streets.

**Also fixed:** GTAO builds its depth/normal buffer with `scene.overrideMaterial`,
which turns additive flames and dust into opaque occluders ringed with shadow.
Those objects are now hidden for the duration of that pass only.

---

## Round 3 — materials, water, air

**Added**

- **Wet silt band** along the waterline, so the bank darkens where the river has
  been rather than meeting the water at a hard colour edge.
- **Foam** where the sheet runs out over the bed, **caustics** from sunlight
  bouncing off the shallow bottom, and a **second, tighter glitter layer** riding
  the fine ripples.
- **Suspended dust**, billboarded and lit only when the sun rakes through it —
  the thing that makes low light feel like air rather than a gradient.

**Measured**

| Preset | mean | contrast | verdict |
|---|---|---|---|
| Golden Hour | 116.7 → 117.8 | 56.3 → 55.4 | neutral at this distance — the work is at the waterline |
| Water close-up | — | — | clear gain: shore foam, caustics in the shallows, wet silt band |
| Starry Night | ground 65.9 → **54.0** vs baseline | 35.4 → 35.1 | clear gain: dark where it belongs, architecture legible |

**One regression corrected rather than kept**

*Ibis six times life size.* A 0.92-unit wingspan scaled to 2.6× produced 6 m
birds that read as white dashes across the plateau, most obtrusively at night.
Scaled to 0.42–0.76.

---

## Deterministic comparison — Golden Hour hero pose

Both sides rendered through the same frozen quality path, baseline checked out
from `f5b446d`.

| Golden Hour | mean | contrast | local detail |
|---|---|---|---|
| baseline `f5b446d` | 117.0 | 57.0 | 5.02 |
| optimised | 118.2 | 55.1 | 4.57 |

| High Noon | mean | contrast |
|---|---|---|
| baseline `f5b446d` | 169.7 | 38.0 |
| optimised | 168.0 | **39.1** |

High Noon is the useful numeric case: a high sun leaves little edge crawl for
MSAA to remove, so the contrast figure is not confounded there and it rises.

Read those two right-hand columns with the caveat above: both drop because MSAA
removed edge crawl. What the side-by-side actually shows:

- plateau surface: speckled aliasing → coherent sand drifts with mastaba clusters
- escarpment: flat strata bands → risers with shadowed recesses
- temples: bare blocks → cornice-crowned masses sitting in contact shadow
- Sphinx quarry: shallow dish → articulated basin
- flood plain: uniform green → irrigated patchwork with dyke lines
- edges: visibly jagged → clean

Everything the earlier per-round figures claimed about structure holds up in the
deterministic pair. The numeric summary does not carry the argument; the images
do.

---

## Performance

Everything added is either build-time (sky occlusion, mastabas, fields) or
instanced. The adaptive ladder drops GTAO first and then the planar reflection
resolution if frame times exceed 30 ms, so the heaviest additions are the first
to go on a weak GPU.

---

# Second loop — bounded visual-polish pass

Baseline for this loop is `bf93292`; accepted result is `50b0be8`. Eight fixed
views were captured before and after: four presets, close pyramid, Sphinx,
Nile/oasis, and a 390x844 mobile portrait.

## Measured, all eight views

| view | contrast | detail | saturation |
|---|---|---|---|
| Dawn | 50.9 → 50.6 | 4.67 → 4.47 | 36.3 → 35.5 |
| High Noon | 39.4 → 38.7 | 4.76 → 4.79 | 26.1 → 25.9 |
| Golden Hour | 54.1 → 55.3 | 4.12 → **4.58** | 44.6 → **46.8** |
| Starry Night | 33.6 → 34.0 | 2.97 → 2.87 | 58.1 → 56.3 |
| Close pyramid | 51.6 → 51.6 | 5.54 → **5.82** | 39.0 → 39.1 |
| Sphinx | 48.2 → 48.3 | 5.23 → **5.66** | 47.3 → 46.8 |
| Nile / oasis | 39.5 → 39.4 | 10.23 → 10.18 | 37.5 → 35.7 |
| Mobile portrait | 54.8 → 54.5 | 4.89 → **5.30** | 45.4 → 44.9 |

Mean relative change: **contrast ±0.0%, detail +3.1%, saturation −1.1%.**
The target was +35%. It was not reached.

## The bug that dominated this loop

Three.js caches shader programs by a key derived from material parameters.
`matSand`, `matCasing` and `matStone` are near-identical MeshStandardMaterials,
so they shared one program — whichever rendered first — and every
`onBeforeCompile` after that was silently discarded. Procedural masonry,
large-scale weathering and the golden-hour limestone rim never executed; the
stone surfaces were running the terrain material's shader. Anyone using
`onBeforeCompile` must also set `customProgramCacheKey`.

Found by elimination: a probe with deliberately absurd values (85% joint
darkening, 8-unit blocks, bright red staining) produced no visible change, which
ruled out weak amplitudes; extracting three's meshphysical fragment shader in
node confirmed every injection anchor exists exactly once and in the right
order, which ruled out the replacements.

## Corrected rather than kept

1. **Aerial perspective, first attempt.** Desaturated from zero distance and
   lifted blacks toward a grey floor. Saturation fell (water 51.1 → 36.6),
   contrast fell (close pyramid 62.1 → 50.4) and night got *brighter*
   (40.0 → 56.3). Reworked to start beyond the monuments at 330 units, capped,
   with no floor term.
2. **Fine procedural masonry as the fix for regular pyramid faces.** A 0.1-unit
   joint seen from 150 units at 820 px is far below one pixel. Kept, because it
   costs nothing and earns its place in the Tour flyover, but it is not the
   solution and is not claimed as one.
3. **Night rim at 0.08.** Tuned while silently disabled; with the shader running
   it lifted night mean 39.2 → 41.4 and cost 2.5 points of saturation. Now 0.025.
4. **Patch generator emitting real newlines into single-quoted JS strings.**
   Broke the module so the world never built. Rewritten with template literals,
   and the render step is now gated on `node --check`.

## Measurement flaws found and fixed

- **Wall-clock waiting between preset switches.** Under software rendering
  Chromium pauses `requestAnimationFrame` during screenshot capture, so a
  5.5 s wait could pass without a single frame and the preset never applied. One
  baseline view was captured showing High Noon while labelled Golden Hour, which
  briefly produced a fake +27% saturation result. The rig now waits for twelve
  rendered frames and asserts the active HUD button, logging a mismatch.
- **Baseline built from the wrong commit** (`f5b446d` instead of `bf93292`).
- **Adaptive quality ladder toggling mid-capture.** Software frame times always
  exceed the 30 ms threshold, so GTAO switched off partway through a run; the
  same pose measured 117.8 and 131.0 mean. Captures now freeze the ladder.
- **Regression test reading light values four frames after a click.** The HUD
  crossfades over 1.8 s with dt clamped to 0.05 s, so a settled reading needs
  about 65 frames. It now polls until the values stop moving.

## Regression check on `50b0be8`

All ten checks pass, no console, page or shader errors:

| check | result |
|---|---|
| Dawn | active, sun 3.20, elev 12°, stars 0.18, torch 0.20, hemi 0.88 |
| High Noon | active, sun 4.50, elev 75°, stars 0, torch 0, exposure 0.97 |
| Golden Hour | active, sun 5.00, elev 16°, dust 0.36, hemi 0.58 |
| Starry Night | active, sun 0.68, moon elev 41°, stars 1.00, torch 1.00, exposure 1.25 |
| presets numerically distinct | OK |
| Cycle | latches, sun keeps drifting |
| Tour | latches, camera flies 54.1 units |
| Orbit drag | camera moves 106.5 units |
| Wheel zoom | distance 322 → 223 |
| Touch orbit, mobile | camera moves 120.4 units |
| Mobile HUD | inside viewport, last button visible |

## Performance

No new textures, no new post-processing passes, no new draw calls — the added
pyramid talus, core patches and entrances all go into existing instanced meshes.
But the cache-key fix genuinely *raises* cost, because three of the four stone
materials had been silently running a cheaper shader. Four programs now compile
where one did before, and the masonry plus weathering add roughly 25-40 ALU
operations per fragment on the largest surfaces in the scene. That cost is real
and I could not measure it: software rendering makes frame timing meaningless
here, so "no performance regression" is not a claim I can support on mobile.

## Remaining limitations

1. Pyramid faces still read as regular stepped courses at overview distance. One
   box per course is the cause; only a geometry change would fix it, and that
   risks the readable silhouette the brief asked to preserve.
2. Vegetation metrics are flat. Clustering redistributed the plants but
   silhouette variety is still four palm variants.
3. Golden Hour is the only preset with a clear gain. Dawn and Night lose a little
   detail and saturation as a designed consequence of aerial perspective.
4. The detail metric — adjacent-pixel luminance difference at 240 px — cannot see
   features finer than roughly three world units at overview distance, which is
   exactly why the masonry failure was hard to catch numerically.
