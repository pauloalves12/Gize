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

# Round 3 — evidence-first, mostly reverted

Target was 125-135 against the previous state as 100. It landed near 105, and
most of what I built in this round is not in the file any more. What follows is
what happened, not what I hoped for.

## Method change that made the difference

The previous rounds measured whole-frame luminance stdev and adjacent-pixel
difference. Both are blind to periodic striping and to between-region depth, so
this round added targeted instruments — and three of them ended up refuting my
own work rather than confirming it:

  banding        row-mean luminance autocorrelation, lags 2-24, inside a crop
  edge coherence how far a horizontal edge stays correlated along x
  edge waviness  spread of the strongest edge's y position after removing tilt
  depth          near-crop minus far-crop contrast and saturation

Baseline for the round: ccf0cc8, captured as an 8-view set.

## Kept

1. Variable pyramid course heights, 0.58x to 1.52x nominal, with taper following
   height rather than course index. Casing-face banding on the close pose:
   78.6 -> 64.7, a 17.6% reduction, and visible at 3.2x as courses that no longer
   march at one pitch. Honest limit: at 400 units it does nothing — the same
   crops read +0.1%, -0.6%, +1.4%. The brief asked for a better read *at
   distance*; this is not that.
2. Vertical erosion runnels in the masonry shader, breaking the horizontal
   reading the courses impose.
3. Aerial perspective: local contrast compressed toward the region mean plus a
   per-preset cool cast. Saturation-depth improved at every preset — dawn
   -5.6 -> -3.4, golden -6.4 -> -3.7, night -22.2 -> -20.7, close -5.1 -> -3.4.
   Contrast-depth is mixed. Modest, and it is now the only depth mechanism.
4. Six palm archetypes — mature, young, old-and-tall, wind-leaned, forked doum,
   dead stump — chosen by ground wetness so age correlates with site. Visibly
   more varied in every matched view. Costs +6 draw calls at equal instance
   count. This was the weakest category and it is now merely weak.
5. Temple gateway framing: lintel, bright jambs, gold winged-disc band, and a
   torus roll following the batter on each outer pylon corner. Verified with a
   purpose-built head-on capture of the Khafre valley temple front.

## Reverted, with the evidence

1. Far horizon ridge, twice. v1 was widely spaced tall slabs: the plateau hid
   their feet from every eastern viewpoint, so only the tops showed and it read
   as blocks floating in the sky. v2 was continuous and low, and read as a picket
   fence of posts over the plateau edge. It also faked two metrics I had been
   treating as gains — whole-frame detail rose on high-frequency sky edges in the
   gaps (4.93 -> 5.32, falling back to 4.96 once removed), and golden-hour
   contrast-depth crashed from -4.0 to -15.2 because a hard silhouette inside the
   far crop injects exactly the contrast distance should remove.
2. The whole terrain pass: meandering cliff line, per-bay cut steepness, scree
   apron, wind mega-ripples, uneven limestone bedding. Four instruments, no
   support — banding -0.1% to -3.1% (noise), edge coherence *worse* at dawn
   +8.0% and noon +5.4%, waviness mixed with right-hand crops -12% to -16%, and
   at matched 3x the cliff read coarser and blockier rather than better. Widening
   the cut made each ledge thicker, hence longer and cleaner: the opposite of the
   intent. rawHeight and terrainColour are byte-identical to ccf0cc8 again. The
   ripples never had a chance regardless — the dune field they modulate is
   off-camera from every pose in the set.
3. The gateway's recessed reveal block. It read as masonry bricking the doorway
   shut: the baseline lets you see through the gate into the court, my version
   walled it up. LIME_D is far too light to pass as shadowed depth at that scale.
   The framing stayed, the block went.

## Four flaws found in my own measurement rig

1. Row-mean banding autocorrelation averages over x, so a meandering cliff still
   scores as strongly periodic. It cannot see the change it was chosen to judge.
2. The first perf script measured mid-boot and reported 14-second frames — that
   was the terrain build blocking the main thread, not a frame time.
3. Then I "fixed" the loader wait to require the .gone class. The loader is
   removed from the DOM 900 ms later, so the stricter check can never pass and
   timed out at 400 s. The original null-tolerant check was correct.
4. Preset probes clicked the HUD button, which uses the 1.8 s crossfade, while
   dt = min(getDelta(), 0.05) clamps only the upper bound. Under software
   rendering the effective dt was about 4 ms, so 150 frames left the blend a third
   done and all four presets read alike near Golden Hour's 5.0. Two separate
   "NOT DISTINCT" verdicts came from this, on different pairs, because each read
   was a snapshot of a moving blend. Re-tested through gotoPreset(i, 0.01), all
   four land exactly on their defined values.

## Performance

Structural cost, measured the same way on both builds:

                    ccf0cc8    7ef46b1
  meshes                 46         52
  draw calls (inst.)     35         41   +6, the palm archetypes
  instances          66,966     67,114   +148 gate boxes, existing mesh
  triangles           2.27M      2.23M   -1.8%
  programs               35         35   cache keys still correct
  boot                2793ms     2864ms  +2.5%, noise

Triangles fell because taller average courses mean fewer of them, which more
than pays for the gate detail. Frame timing is still not measurable here and I
am not going to quote a swiftshader millisecond figure as if it meant something.

## Regressions

All four presets latch and land on their defined values; Cycle, Tour, orbit
drag, wheel zoom, mobile touch orbit and the 390x844 HUD fit all pass; no
console or page errors; 35 programs compile.

## Honest position

Roughly 105/100, not 125-135. The one thing the brief put first — pyramids
reading as even bands at viewing distance — is untouched, because the cause is
one box per course and the fix I tried helps only close up. The terrain and
horizon work, which was most of the round's effort, produced nothing measurable
and was removed. What survives is three modest, verified gains and one regression
of my own making caught and removed before shipping.

# Round 4 — geometry & material pass on the pyramids: a null result, and why

Scope was the pyramid faces, the complaint being that they still read as regular
horizontal bands at overview distance, especially on mobile. Four material
interventions were built and all four were reverted. index.html ends the round
byte-identical to where it started (7ef46b1). The output of this round is the
diagnosis and three repaired instruments, not a visual change.

## Pass A — the measurement that reframed the problem

At the overview pose (48 deg vertical fov, 470 px tall, ~385 units to the Khufu
mid-face, focal 527.8 px):

  one course (0.84 u average)                      1.15 px
  same course, mobile portrait 390x844             2.07 px
  4.5 u feature                                    6.2 px
  8 u feature                                     11.0 px
  19 u feature                                    26.0 px

Khufu carries ~68 courses. The face is a periodic sub-pixel step pattern sampled
at the Nyquist limit, so the "banding" is an alias rather than a material
shortcoming. Measured stripe amplitude on the overview faces is 0.6-1.7 where a
full-contrast 2 px stripe pattern scores 28.57, and the dominant periods are
7-17 px, i.e. 7-12 world units -- the weathering-noise scale, not the courses. A
genuine 2 px course stripe exists only close up, at 1.59%.

Conclusion: the premise does not hold at overview distance. Nothing there is
measurably banded.

## What was built and why each one failed

  B1 sub-pixel LOD fade (joints, grain, stratum band, fine runnel faded
     110-300 u). Cleaned the shadowed faces but left them reading as plaster.
     Its apparent 12% detail cost turned out to be framing jitter, not the
     shader.
  B2 course-top normal convergence, so +Y facets shade like the face at range.
     At 12 deg viewing elevation the treads project to ~0.2 px, so there was
     nothing to fix; the 6x crop shows the sunlit staircase unchanged.
  B3 broad weathered panels at 16 u and 11 u (22 px and 15 px). Band power fell
     slightly at every period instead of rising.
  C  quarry tiers: per-course tone moved from ~1.4-course noise (1.6 px, pure
     alias fodder) into coherent 7-18 course tiers (8-21 px). Within noise, and
     confounded by a 5.5% brightness error of mine.
  D  coarse vertical articulation at 6 u (8.2 px), staggered, tone-carried,
     brightness-neutral. Column-wise high-pass energy moved -2.7% to +1.0%; the
     8 px bin, where 6 u blocks must land, went 0.22->0.20, 0.46->0.48,
     0.25->0.24, 0.28->0.28. Visually indistinguishable at 6x.

D failed for a reason that explains the whole round: the casing already carries
+/-15% large-scale tonal variation (wthVar 0.30) across 6.5-19 u. A further
+/-3.75% at 6 u is a quarter of what is already there and disappears into it.
The faces were never short of large-scale material breakup.

## Four instrument failures, three of which corrupted numbers already reported

1. banding() autocorrelated the row-mean luminance profile, but a smooth vertical
   gradient is nearly perfectly self-correlated at small lags, so every pyramid
   crop floored at 90-96 regardless of surface. Synthetic check: a pure ramp
   scores 94.9 old / 0.00 corrected; pure 2 px stripes 100.0 / 28.57. This voids
   round 3's headline "-17.6% banding" from variable course heights -- corrected,
   that crop reads 3.92 -> 3.90.
2. The capture rig carried 6-7 px of framing jitter: shots.mjs set the orbit pose
   and waited 12 frames, but the loop moves the camera by a damped lerp
   (1 - 0.02^dt) that lands near 90% with a timing-dependent residual. At 6x that
   is +/-40 px of apparent shift. Fixed by snapping the camera to orbitPose; an
   untouched terrain control band now differs by 0.38 px, 36/50 columns identical.
3. Pass C brightened the casing 5.5% by accident (tier tone averaged 1.000 against
   a baseline mean of 0.945), confounding every reading until a terrain control
   band isolated it.
4. stripe_amp and band_power both work on row means, so they are blind to vertical
   articulation by construction -- I designed Pass D vertically and then measured
   it horizontally. A column-wise version was needed to judge it at all.

## The route that is actually open, with costs

Per-face block segments, so that per-block tone, inset and missing blocks exist
at a scale the overview resolves:

  6 u segments -> 8.2 px on screen, 5140 casing instances, 0.06M triangles
  9 u segments -> 12.3 px on screen, 3460 casing instances, 0.04M triangles

Against a scene already carrying 67,114 instances and 2.23M triangles this is
cheap: roughly +7.7% instances and +2.7% triangles for the 6 u variant. My
earlier claim that a rebuild was the expensive option was wrong. The honest
caveat is that per-block relief is also sub-pixel at overview (0.2 u = 0.27 px),
so a geometry pass would buy per-block tone and missing-block silhouette
nibbling, not visible depth -- and per-block tone has to exceed the +/-15%
weathering already present to register at all.

## Regressions and cost

Unchanged, because the shipped file is unchanged: presets land exactly on target
and distinct (Dawn 3.20/12 deg, Noon 4.50/75 deg, Golden 5.00/16 deg, Night
0.68/41 deg), Cycle, Tour, orbit drag, wheel zoom, mobile touch orbit and the
390x844 HUD all pass, no console or page errors, 52 meshes / 41 instanced /
67,114 instances / 2.23M triangles / 35 programs, boot ~3.1 s.

## Honest position

100/100. The round moved nothing on screen. It did establish, with numbers, that
the stated bottleneck is not present at overview distance, that four material
levers cannot move it, and that the remaining route is geometry at a known and
affordable cost. That is a useful result but it is not an improvement.

# Round 5 — macro geometry and monumentality: the first clear win in three loops

Scope was large-scale composition rather than surface detail, on the explicit
premise that round 4 had proved nothing below ~4.5 u (6 px at overview) survives
at this distance. Two of three passes were kept. Unlike rounds 3 and 4, this one
started from a weakness that measurement actually confirmed.

## The diagnosis this round worked from

A skyline instrument was built for the loop: per column, the topmost non-sky
pixel. The colour test that seemed obvious fails immediately -- at golden hour
the sky is orange and at night deep blue -- so detection keys on SMOOTHNESS
instead: sky is a clean vertical gradient, terrain and masonry are textured. The
detected line was then drawn back onto the frames and inspected rather than
trusted, which is how the earlier metric failures should have been caught.

It showed a real problem: the pyramid apexes clear the plateau's own silhouette
by roughly 35 px out of 470. The mesa is the dominant mass and the pyramids read
as bumps on it.

## Kept

1. Four great terraces on the escarpment front, replacing the uniform slope.
   Treads 12-20 u, risers 8-11 u, i.e. 16-27 px and 11-15 px on screen. The
   terrace line wanders +/-2 u in z so it reads quarried rather than machined.
   This is the loop's real win: the front went from a shapeless wall of thin
   stripes to a stepped landform with genuine shadow between levels, confirmed
   independently at golden hour and at noon. Terrain-only, so free.
2. The dune horizon dropped, ridge amplitude 44 + 21 * ridged -> 42 + 11.
   Crests had reached ~70, some 26 u above the 43.75 plateau top, putting the
   field behind the pyramids across their lower two thirds. Apex clearance
   against the background skyline improves +10 px golden, +5 noon, +26 mobile,
   and sky fraction rises 0.4-0.6 points. Small, consistent, visible.

Also added: the great quarry, a 65 x 64 u recess 9 u deep on the plateau
(89 x 88 px with 12 px walls), sited clear of the Sphinx enclosure, all three
causeways and the mastaba fields. Part of pass A and kept with it.

## Rejected

Stepped pyramid podiums -- a broad 10.5 u apron at 3.0 u plus a 5 u upper course
at 5.4 u, ring segments self-policed against keep-out rectangles for temples,
queens, neighbouring pyramids, mastaba fields and causeways. The engineering was
sound and nothing got buried. It was reverted anyway: visible at 3.6x zoom,
marginal at the real overview framing, and indistinguishable on mobile portrait
where the bases sit behind the plateau edge and the temples. Cost was not the
issue (+364 instances, +0.01M triangles) -- visibility was. This is the same
"only reads in close-up" trap round 4 fell into four times.

## Honest caveats

- The terraces are carried by visual evidence, not by metrics. Skyline and
  coarse-contrast instruments measure the outline and the frame; the terraces
  change the plateau's front FACE, which no silhouette metric can see. Coarse
  contrast moved +0.9 at golden and +0.9 at noon, which is not what is carrying
  the judgement.
- Mobile portrait is a narrow zoomed framing (48 deg vertical over 844 px), so
  the escarpment front is largely out of frame. The loop's biggest gain does not
  reach the viewport the brief emphasised most. Mobile is neutral, not improved,
  apart from the dune-horizon effect.
- Apex clearance is still modest in absolute terms. The pyramids read better than
  before but the plateau remains the larger mass; genuinely reversing that would
  mean raising the monuments or cutting the plateau down, both of which pull on
  the causeway and temple elevation chain.

## Performance

  baseline 7ef46b1   52 meshes / 41 instanced / 67,114 instances / 2.23M tris / 35 programs
  shipped  9613c4b   52 meshes / 41 instanced / 67,104 instances / 2.23M tris / 35 programs

Both kept changes are terrain-only, and the terrain is a fixed-size heightfield
grid, so the cost is nil. The ten-instance difference is vegetation and prop
scatter reacting to the new heightfield, not added geometry. Boot 3.17 s -> 3.30 s,
within run-to-run noise.

## Regressions

All four presets land exactly on their defined values and stay distinct
(Dawn 3.20/12 deg, Noon 4.50/75 deg, Golden 5.00/16 deg, Night 0.68/41 deg).
Cycle, Tour, orbit drag, wheel zoom, mobile touch orbit and the 390x844 HUD all
pass. No console or page errors. 35 programs compile.

## Position

A real gain, and the first in three loops. Roughly 108-112 against the round-4
state of 100 -- driven almost entirely by the escarpment terraces at overview
distance, with a small consistent contribution from the dune horizon. Not
transformative, and it does not touch mobile portrait.

# Round 6 — mobile portrait composition: a camera problem, not a world problem

Round 5's escarpment terraces were the best gain the project has had, and mobile
portrait could not see them. This round fixed the framing, not the world. No
geometry, textures, passes or render targets were touched.

## The diagnosis

three.js PerspectiveCamera fov is VERTICAL, so portrait holds the 48 deg vertical
field and lets the horizontal field collapse:

  1920x1080  h-fov 76.7 deg   609 u of width at 385 u
  820x470    h-fov 75.7 deg   598 u
  844x390    h-fov 87.9 deg   742 u
  390x844    h-fov 23.3 deg   158 u
  412x915    h-fov 22.7 deg   154 u
  430x932    h-fov 23.2 deg   158 u

The monument group spans about 300 u, Menkaure at x -180 to the valley temples at
x +120. Portrait saw 158 u of it -- a 23 deg telephoto slice, literally the
desktop composition cropped vertically, which is the exact failure the brief
names. Neither lever fixes it alone: 66 deg vertical only reaches 231 u and starts
to distort, and retreating to 780 u covers 321 u but shrinks the pyramids to
nothing.

## What was kept

Portrait, gated on aspect < 0.75, so landscape and desktop take the old path:

  fov 56 (landscape stays 48), following device rotation via the resize handler
  target (-70, 34, -20), dist 300, theta 2.100, phi 1.440, snapped on frame one
  tour look target lifted +8 u

The winning idea is to look ALONG the Khufu-Khafre axis rather than across the
group. Spread sideways the pair needs ~194 u of width; seen along their own axis
they overlap in depth, so one frame holds both at full height from 300 u out --
larger and uncropped at once, with the recession doing the depth work a tall frame
is good at.

## Measured, 390x844 golden, before -> after

  sky              36.0% -> 34.6%
  vegetation       14.8% ->  0.7%
  water             3.5% -> 12.4%
  pyramids       2 clipped -> 2 complete and substantially larger
  terraces        present but confused -> legible banded mass in the lower third
  HUD masonry       0.3% ->  0.3% golden, 37.3% noon (the village, not a monument)

Tour framing, checked analytically at all sixteen control points against the
portrait frustum: 12/64 monument-stop pairs fully in frame at the old fov, 14/64
at fov 56, 16/64 with the +8 u look bias, and apex clipping down from 5 stops to 2.
Verified geometrically rather than visually -- reaching t=0.75 for a screenshot
needs roughly 930 rendered frames under software rendering.

## Tested and rejected

  P1 fov 60 only          safe, but the middle third stays busy
  P2 wide + back          pyramids shrink into an 85% rock band
  P3 axial                flat; foreground dominates
  P4 pyramid diagonal     real depth, but the quarry mass takes over
  P5 back + high          worst; 87% rock, apex clearance collapses to 15 px
  P6 close + low          strong read but sky reaches 43.7%
  P8 (shipped briefly)    fixed legibility but paid in pyramid size
  Q2                      biggest pyramids of all, but its lower 40% is a
                          featureless dark rock mass

## Desktop regression

Settled with a noise floor rather than by argument, after one wrong-reference
scare (D_base predates round 5's terraces, so it is not a valid desktop baseline):

  same build, two consecutive captures   rock/masonry mean|diff| 4.45
  desktop before vs after this loop      rock/masonry mean|diff| 4.26

Below the floor, so desktop is unchanged. The larger diffs sit in animated water
and bloom, 23.91 against a 20.66 floor.

## Performance

  before  52 meshes / 41 instanced / 67,104 instances / 2.23M tris / 35 programs
  after   52 meshes / 41 instanced / 67,104 instances / 2.23M tris / 35 programs

Identical in every figure, as a camera-only change must be. Boot has ranged
2.79-3.88 s across all runs this session; 3.45 s here is inside that variance.

## Honest limitations

- The lush green Nile foreground largely leaves the portrait frame, 14.8% to 0.7%.
  Portrait trades the green-riverbank-versus-white-limestone contrast the original
  brief asked for in exchange for monumentality. Landscape and desktop keep it.
- Only two of the three main pyramids are in the portrait frame; Menkaure is out.
- The mud-brick village sits partly behind the control bar at noon.
- The golden-hour lower third is quite dark.
