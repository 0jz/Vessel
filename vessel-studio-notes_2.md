# VESSEL Studio — file notes

`vessel-studio.html` is a single self-contained file (no build step) — Three.js, GSAP/ScrollTrigger,
and Google Fonts all load from CDN. Open it directly or serve it locally
(`python -m http.server` / `npx serve .`).

## What's in it

**Structure**: hero → selected work → mission → vision → services → footer, single-page scroll.

**Background**
- A diamond/rhombus grid drawn as generated SVG paths (not a filter or a tiled pattern) — three
  tiers layered together: a fine subdivision, a base grid, and a bold 3x-scale accent grid.
- Every line's curve is computed from its own position relative to the true center of the screen,
  using one shared formula across all three tiers — that sharing is what keeps them properly
  nested (fine inside base inside huge) instead of drifting apart and crossing each other, which
  is what happened when each tier had its own warp intensity.
- The curve pulls *toward* center (concave) rather than bulging outward — reads as the inside of
  a rounded wall/dome rather than a fisheye bulging at the viewer, matching the Alche reference.
- One diagonal direction in the base tier runs through an SVG turbulence filter for a hand-drawn
  wobble; everything else is clean.
- `preserveAspectRatio="none"` on a 1600×1000 viewBox means it always stretches to fill the
  screen; width currently stretches more than height (this is a one-line swap if you want the
  opposite — see the viewBox comment in the code).

**Hero WebGL scene**
- ~58 wireframe icosahedra with real physics: they drift, bounce off the edges of an invisible
  play area, and can be picked up and thrown.
- Click + drag any shape to grab it (it turns fully colored while held), release to throw it —
  it carries whatever velocity your cursor had at release and keeps moving under damping.
- The play area's side walls are barrel-shaped (bulge outward near vertical-center, taper at the
  top/bottom) rather than a flat box, loosely tracking the same curved-wall idea as the background.
- On a wall hit, the nearest 2–3 objects (by camera distance) spawn a color splash at the impact
  point, rendered on a layer *behind* the canvas so the object itself stays visible on top of the
  color rather than being covered by it. The splash element is fully removed from the DOM after
  it fades — nothing lingers.
- Objects are dim/see-through (double-sided) at rest; fully saturated in their own color only
  while grabbed.

**Cursor**
- Custom dot + trailing ring, expands and labels itself (`VIEW` / `GO`) over interactive elements.
- No distortion lens anymore — the magnifying/fisheye effect that used to follow the cursor has
  been removed entirely (CSS, SVG filter, and the JS tracking for it are all gone).
- Magnetic pull on the nav CTA and footer links.
- Work-list rows spawn a color-tinted preview box that follows the cursor while hovering, with a
  failsafe that force-hides it if the cursor ends up outside the row's bounds for any reason
  (covers scroll-while-hovering, which was the original cause of it sometimes sticking).

**Vision section**
- Wrapped in a "liquid glass" panel — blur + saturate + a gentle independent refraction filter
  (this one's untouched by the cursor-lens removal, it's a separate static filter), soft gradient
  sheen, inset highlight border.
- The section itself has no background of its own (`overflow:hidden`, transparent) — it's set up
  so a video/gif can be dropped in as an absolutely-positioned layer behind the glass panel later.

**Services**
- Two cards, subtle cursor-tilt (kept intentionally small + `overflow:hidden` so nothing pops
  outside the card during the tilt).
- Dark background fades to transparent on hover so the WebGL scene reads through while tilting.
- Real gap + individual borders between the two cards. This also fixed an earlier bug where card
  01's tilt broke down as the cursor approached the shared boundary with card 02 — the ancestor's
  `perspective` was centered on that boundary rather than on each card individually.

## Known content is placeholder

Studio name (VESSEL), all copy, the four work items (Northlight / Fathom / Aurelia / Driftwave),
and their tags are original placeholder content — none of it is real. Swap freely.

## alche.studio reference — how they do the round/folded grid

Checked their hero (`kv`) background at the code level (downloaded and grepped their JS bundle,
`_astro/index.astro_astro_type_script_index_0_lang.*.js`). It's **not** a 2D SVG bow like this
project — it's a literal 3D bend applied in a Three.js vertex shader to a `PlaneGeometry(1,1,64,64)`
via a `Grid` class (`new Grid("thin_light","flat",uniforms)` for flat pages, presumably a "round"
variant for the hero). Relevant vertex shader (`gridVert`), trimmed to the bend math:

```glsl
uniform vec3 uScale;
float theta = localPos.x * PI;           // local x in [-0.5, 0.5] -> angle in [-PI/2, PI/2]
vec3 roundedPos;
#ifdef IS_ROUND
  roundedPos = vec3(
    sin(theta) * 0.5 * uScale.x,
    localPos.y * uScale.y * 1.5,
    -cos(theta) * uScale.x * 0.5
  );
#endif
#ifdef IS_FLAT
  roundedPos = vec3(localPos.x * uScale.x, localPos.y * uScale.y, 0.0);
#endif
```

So their "fold" is a real half-cylinder: the plane's local x drives an angle `theta`, and the
vertex is placed on a circle of that angle (sin for one axis, -cos for depth) while y stays a
plain linear scale. `IS_FLAT` is the same plane with no bend at all (used elsewhere, e.g. the
grid-cross accent). The grid *pattern* itself is drawn by a separate fragment shader
(`gridFrag`) as a plain repeating line grid in UV space (`fract(uv*uGrid)` thresholded to a
line) — the fold is 100% a geometry-space vertex transform, the fragment shader doesn't know
about it.

This is now actually implemented below (own plane + vertex-shader bend, angle-driven `sin`/`cos`
placement rather than a per-line quadratic bow) rather than just noted for later.

## Background grid — now a real bent WebGL plane (not 2D SVG anymore)

Replaced the SVG version entirely with a Three.js plane in its own small scene/canvas
(`#bgGridCanvas`, still `#bgGrid`, still z-index 0, fully decoupled from the icosahedra scene so
none of the existing physics/camera code was touched). This is the actual alche.studio technique,
not an approximation of it:

- **Vertex shader bend**: local x drives an angle (`theta = t * uMaxAngle`, `t = x/halfWidth`),
  and the vertex is placed with `sin(theta)`/`cos(theta)` like wrapping paper partway around a
  cylinder — flat/zero-curvature exactly at x=0 (the seam), curving increasingly toward the edges.
  `uMaxAngle` is kept modest (0.5 rad) so it reads as "a little bit circular," not a full tube.
- **Paper fold**: same shader also adds `sign(y) * edge² * uFold` to y — top half tilts up, bottom
  half tilts down, strongest at the same left/right edges, zero at both the vertical and
  horizontal seams. That's the "crease down the middle, flaps curling away" read.
- **Grid pattern**: drawn procedurally in the fragment shader from the plane's *flat* local UV
  (three tiers — fine/base/accent — each a `fract()`-based line test, rotated 45° so the square
  lattice reads as diamonds). Procedural = no discrete per-line coverage limit, which is what
  makes ultrawide/superwide monitors non-issues for free — the pattern just keeps tiling. Geometry
  itself is also sized at `visibleFrustum × 1.6` (`MARGIN` constant) so there's real margin past
  the edges too, covering both extreme aspect ratios and the mouse-tilt range without ever
  exposing a bare plane edge.
- **Mouse parallax**: whole mesh rotates a couple degrees toward the cursor, lerped 0.05/frame for
  smoothness. Skipped under `prefers-reduced-motion` and on coarse/touch pointers.
- **Scroll travel + fade**: one `ScrollTrigger` (hero → vision, scrubbed) feeds a `uScrollY`
  uniform that scrolls the pattern in the fragment shader, giving a "moving through the grid"
  feel as you scroll — same idea as alche's `gridUv.y -= uScroll*1.5`. A second, separate
  `ScrollTrigger` (`#vision` `top bottom` → `top center`) fades the whole layer's alpha to 0 right
  as the vision section arrives, so the background hands off cleanly to "Immersive worlds..." —
  it stays fully visible through hero/work/mission and only fades in that last stretch.

Perf note: single plane, ~160×100 segments max (capped in `buildGeometry`), one draw call — much
cheaper than the old approach of generating hundreds of individual SVG `<path>` elements. Still
worth profiling on low-end/integrated GPUs per the existing todo below.

## Vision section — video/gif slot is now wired up

Added `<video id="visionVideo" autoplay muted loop playsinline>` directly behind `.glass-panel`
in `#vision`, with CSS already handling sizing (`object-fit:cover`), a soft radial mask so it
fades into the background rather than hard-cutting at the edges, and opacity tuned to sit behind
the glass panel's blur without fighting the text. **To use it: just set `src` on `#visionVideo`**
(or add `<source>` tags for mp4/webm) once you've got the asset.

### What to put there: suggested content

Given the "we architect places / immersive worlds" copy and the hero's wireframe-icosahedra
motif, the strongest fit is a **loop of one of those same wireframe shapes assembling and then
deconstructing** — start whole, end fractured into its own vertices/particles (or reverse the
loop so it's seamless: assemble → hold → deconstruct → the last frame matches the first). That
ties the vision section visually back to the hero's shapes instead of introducing new imagery.

- **Tool**: any of Runway (Gen-4), Kling, Luma Ray2, or Pika — all handle "object breaking apart
  into particles" well from a still + prompt, or text-to-video directly. Kling/Runway currently
  give the cleanest geometric-fracture results for this kind of abstract/product-style motion.
- **Prompt direction**: describe a single low-poly wireframe icosahedron (or crystal/gem shape)
  in the exact palette already used in the hero (`#d4ff3d`, `#7cf2c4`, `#ff9d6c`, `#9d8cff`) on a
  warm cream/transparent background, slowly rotating, then shattering outward into its own
  wireframe edges/vertices which drift apart like dust — slow, meditative pace (this sits behind
  a text panel, it shouldn't compete for attention). Ask for a **loopable** result explicitly, or
  plan to mirror/reverse it yourself in editing (ping-pong loop) if the generator can't guarantee
  a seamless one.
- **Format**: generate at whatever the tool gives you (usually 5–10s), then export as **mp4/webm
  instead of GIF** for the actual site — much smaller file size and far better quality/color depth
  than GIF at the same size, and the `<video>` tag is already in place for exactly that. Keep it
  muted/autoplay/loop as set. If you do want an actual GIF (e.g. for sharing on socials/Notion
  separately from the site), export from the same source clip with `ffmpeg` a palette-optimized
  GIF rather than a direct convert — much cleaner color banding.
- **Other spots this same asset (or a variant) would work**: the `#mission` section currently has
  no visual at all behind "We don't decorate screens. We architect places" — a very subtle,
  low-opacity version of the same clip (or just a stilled/slowed frame of it) could live there
  too, so the assembling shape "arrives" in mission and finishes deconstructing by vision — one
  continuous idea spread across both sections rather than two unrelated treatments.

### Grid sizing fix — was rendering wrong/invisible, now width-relative

Catching this for the record: the first WebGL pass above had a real bug — spacing was set in
world units (40/120/360, carried over unchanged from the old pixel-based SVG version) but the
plane itself is only ~15–25 world units across at typical window sizes, so those tiers were
mostly *bigger than the whole plane* — the fine/base tiers barely showed a line, and it wasn't
actually scaling with the window at all (the width/height conversion factors canceled out
algebraically, so a first attempt at "pixel-accurate" spacing landed on a constant on-screen
size no matter the window). Fixed by making spacing a **fraction of window width**
(`FRACTION_FINE/BASE/ACCENT`, tuned against a 1600px baseline to land a bit smaller than the
original 40/120/360px), recomputed into world units on every resize. Line widths and the wobble
amplitude are kept as ratios of their own tier's spacing rather than separate constants, so they
scale proportionally too instead of going stringy-thin or chunky at other widths. Verified by
screenshotting with the icosahedra layer hidden (so only the grid canvas remains) and measuring
line spacing directly: ~11px at 900px window width → ~13px at 1440px → ~23px at 3440px (true
ultrawide) — smaller than before at typical sizes, and now actually responsive to width.

### Inside-wrap, no scroll travel, clean nesting

Three follow-up fixes:

- **Wrap direction flipped**: the bend was pushing the plane's edges *away* from the camera
  (`bent.z = -(1-cos(theta))*radius`), which reads as wrapping the outside of a dome/ball —
  bulging toward you at center, receding at the sides. Flipped the sign so edges push *toward*
  the camera instead — now it reads as the inside of a cylinder (like standing inside a rounded
  wall), matching the original "inside of a rounded wall/dome" intent from the very first version
  of this background.
- **No more scroll-driven movement**: removed the `uScrollY` uniform and the ScrollTrigger that
  was feeding it. The pattern used to visibly scroll/travel as you scrolled the page (echoing
  alche's `gridUv.y -= uScroll*1.5`) — but alche's own background actually stays visually still
  while the page scrolls over it (it's just `position:fixed`), so removed the travel and kept
  only the fade-near-vision behavior. Only the mouse parallax and a slow ambient time-based
  wobble move it now — scroll position no longer affects it at all.
- **Fixed overlapping/moiré lines**: the fine/base/accent spacing had drifted to a 1 : 3.33 : 10
  ratio (from an earlier width-based rewrite) instead of the clean 1 : 3 : 9 the original
  flat-SVG version used. Off-ratio tiers don't nest — lines land *near* each other but not
  exactly on top, which reads as doubled-up/overlapping/fuzzy. Base and accent are now locked to
  exact 3× and 9× the fine spacing, so every base line falls exactly on a fine line and every
  accent line falls exactly on a base line again.

### Fixed: elongated diamonds at the horizontal seam

Found the actual bug behind this one. The paper-fold term was using `sign(p.y)` to decide
up-vs-down — and `sign()` jumps instantly from -1 to 1 right at y=0, with no ramp. That meant a
vertex a fraction of a unit above center got shoved up by the *full* fold amount, same as a
vertex way up near the top edge — while its neighbor a fraction of a unit below center got
shoved down by that same full amount the other way. The two rows straddling the seam got pulled
apart by a fixed jump regardless of how close they actually were to center, which is exactly what
stretched the middle row of diamonds — not the outer ones, since everywhere else the bend is
already smooth and that fixed jump reads as unremarkable next to the surrounding curvature.

Fixed by swapping `sign(p.y)` for `clamp(p.y / halfHeight, -1, 1)` — a continuous ramp that's
genuinely 0 at y=0 and grows smoothly with actual distance from center, same idea as the
horizontal bend (`t = x/halfWidth`) already used. Now the vertical fold is one smooth function of
where a vertex actually sits, top and bottom included, instead of a step function with a special
case exactly at the seam. Added a matching `uHalfH` uniform (mirrors the existing `uHalfW`) so
this scales correctly on resize too.

### Rewritten to match alche's actual technique, stronger bend, calmer cursor movement

- **Now genuinely alche's approach, not an approximation of it**: geometry is a fixed, normalized
  `PlaneGeometry(1,1,...)` (local x/y always in [-0.5, 0.5]) — exactly matches their setup. All
  real-world sizing moved into a single `uScale` uniform multiplied in at the end of the vertex
  shader (their `uScale.x`/`uScale.y` pattern), instead of baking window-dependent half-widths
  into the geometry and rebuilding it every resize. Net effect: the mesh is now created **once**
  at load; resize just updates `uScale`/spacing uniforms and the camera, no more dispose+rebuild
  cycle. Genuinely simpler, not just smaller-looking code — one fewer moving part.
- **Stronger distortion**: bend strength raised to 0.82 of a full alche-style half-turn (was a
  much more conservative 0.5 radians before, roughly a third of this), and the paper-fold push
  roughly doubled. Reads as clearly bent now rather than "a little bit circular."
- **Calmer mouse parallax**: rotation coefficients cut way down (0.14/0.07 → 0.05/0.025) and the
  lerp slowed (0.05 → 0.035), so the background drifts gently toward the cursor instead of
  visibly tracking it.

The fold term (top curls up, bottom curls down) is still this project's own addition on top of
alche's technique — their actual plane doesn't fold, it's a plain half-cylinder. Kept it since it
was requested earlier and reads well combined with the stronger bend.

### Restructured to straight squares — pixel/cell/tile "screen" hierarchy

Two changes:

- **Straight squares, not diamonds**: dropped the 45° rotation (`rot(0.785398)`) that was
  turning the axis-aligned square lattice into rhombi. Same underlying `gridLine()` math, just
  unrotated — now a plain screen-like grid.
- **New nesting ratio, screen/pixel metaphor**: renamed the three tiers to match how they're
  meant to read — `tile` (largest, bold, colored — "a tile screen"), `cell` ("a screen subpart" —
  thinner), `pixel` (finest — barely-there, like an actual pixel grid). Sized so:
  - **tile** = 3/4 of what the old largest ("accent") tier used to be
  - **cell** = exactly 1/4 of a tile → a **4×4 grid of cells per tile** (16 per tile)
  - **pixel** = exactly 1/8 of a cell → an **8×8 grid of pixels per cell** (64 per cell, so
    32×32 = 1024 pixels per tile total)

  Same reasoning as the old fine/base/accent 1:3:9 nesting — locking these as *exact* integer
  ratios (not just "roughly 4x/32x") is what keeps every tier's lines landing exactly on the next
  tier's lines rather than drifting into overlap. `FRACTION_TILE`/`FRACTION_CELL`/
  `FRACTION_PIXEL` are all derived from one number (`FRACTION_TILE`) rather than set
  independently, so that nesting can't drift out of sync in a future edit.

This sets up a real pixel grid (1024 addressable cells per tile) if this ever wants to display
actual patterns/images rather than just a decorative grid — each pixel cell is already a discrete,
evenly-spaced unit in the fragment shader's UV space, so driving individual cells on/off (or by
color) from a texture or data uniform would slot in on top of this structure rather than requiring
a rebuild. Worth a dedicated pass if/when there's an actual pattern to display — happy to build
that next.

### Real fix for the overlap (compositing bug, not alignment), tile color, and a full displacement rewrite

- **Tile color**: green accent → dark gray (`0x4a4a4a`).
- **The actual cause of "overlapping" this time**: the nesting ratios were already exact
  (verified: cell = 8× pixel, tile = 4× cell, no drift). The real bug was in the fragment
  shader's blending — it *summed* all three tiers' contributions (`colorPixel*pixel +
  colorCell*cell + colorTile*tile`, alphas added too). Since tile lines are, by construction,
  always also cell lines and pixel lines (exact multiples), every tile line had all three
  colors and alphas stacking on top of each other along its *entire length* — reading as
  blown-out, doubled-up lines. Fixed by compositing as layers instead of summing: pixel drawn
  first, cell painted over it where present, tile painted over both where present — each bolder
  tier simply wins outright rather than adding to what's under it.
- **Displacement replaced entirely** — this was the bigger change. The alche-style uniform
  cylinder bend (+ the paper-fold on top of it) bent the *whole* plane everywhere, which reads
  as the opposite of "large flat areas interrupted by a handful of folds." Replaced with a
  `heightField()` function in the vertex shader: a small, fixed set of analytic ridge functions
  (gaussian cross-section, soft fade at both ends, one slow sine wander per ridge for a hand-
  drawn curve rather than noise) — two large diagonal ridges crossing the canvas at different
  angles, plus one smaller ridge that peels off the first partway along its length for the
  "branching." Displacement pushes mostly into z (depth) with a smaller y component so it reads
  as folds rather than pure depth bumps. No per-vertex randomness or noise octaves anywhere —
  it's a fixed, deliberate composition, tunable by editing the three `ridge(...)` calls in
  `heightField()` (origin, direction, width, length, amplitude, curve — each is a plain
  parameter, no rebuild needed).
- Also dropped the small per-fragment "hand-drawn wobble" sine ripple that was on the grid lines
  themselves — it read as a repetitive, evenly-distributed texture, which is exactly what was
  asked to avoid. Grid lines are perfectly straight now; all the organic quality lives in the
  ridge folds instead.

### Correction: ridge topology moved from geometry to color

Misread the earlier brief — the "sweeping diagonal ridges with branching" description was meant
to drive the grid's **coloring**, not bend its **geometry**. Reverted:

- **Geometry is flat again**: the vertex shader no longer displaces position with the ridge
  height field — it's back to a plain `lp * uScale` mapping (plus the whole-mesh mouse-tilt
  rotation, which is a rigid transform applied to the entire mesh, not a per-vertex warp, so it's
  unaffected by this and still there).
- **The same `ridge()`/`topology()` functions moved into the fragment shader** and now drive
  color instead of position: two large diagonal ridges plus one branching off the first, exactly
  as before, but the accumulated height at each fragment is used as a 0→1 mix factor between two
  colors instead of a vertex offset.
- **Colors are derived from the site's own `--accent` (`#d4ff3d`)** via HSL rather than
  separately hand-picked hexes, so they stay in the same hue family if the accent ever changes:
  same hue, `l≈0.16` (and reduced saturation) for the dark/"weak" end sitting in the flat areas
  away from any ridge, `l≈0.82` (softened saturation) for the pastel/"very light" end right at
  the ridge crests. Works out to roughly `rgb(58,71,10)` dark → `rgb(221,230,188)` pastel.
- Line **presence/thickness** (which of pixel/cell/tile is drawn, and how boldly) is unchanged —
  only alpha now, since color is fully taken over by the topology gradient rather than being
  per-tier.

### Re-applied: overlap fix + tile color (this uploaded copy had reverted)

The copy of the file this was re-fixed from still had the *old* fragment shader — colors/alphas
summed across all three tiers (`uColorPixel*pixel + uColorCell*cell + uColorTile*tile`) — and the
tile color was still the green accent (`0xd4ff3d`), not dark gray. Re-applied both fixes described
above: compositing is layered (pixel drawn first, cell overrides where present, tile overrides
both where present, each tier fully replacing rather than adding to what's under it), and
`uColorTile` is now `0x4a4a4a`. Worth keeping this file as the canonical copy going forward so this
doesn't need a third pass.

### Second overlap pass — proper alpha compositing + depth write + thicker fine tiers

The layered `if(coverage > 0)` swap from the previous pass was still a hard branch: it stops one
tier's color/alpha dead at the edge of the next tier's antialiasing band, which itself reads as a
faint seam. Replaced with real "src-over-dst" compositing (`mix()` using each tier's own coverage
as the blend weight, pixel → cell → tile in order) so the color/alpha transition across each
tier's AA band is continuous instead of stepped.

Separately — and probably the bigger visible cause of "still overlapping," especially angle-
dependent — the plane's material had `depthWrite:false` (typical for a transparent material, to
avoid self-sorting issues) but the geometry is a single *bent* surface, and at grazing/rotated
angles parts of that curved surface can project to the same screen pixels as other parts of
itself. With depth writes off, those self-overlapping fragments were just alpha-blending on top
of each other in whatever order they happened to render, which shows up as doubled/blown-out
lines exactly where the bend is strongest — i.e. at the edges, at an angle. Turned `depthWrite`
back on (`depthTest` was already implicitly on) so nearer parts of the bent plane properly occlude
farther parts of itself instead of blending with them.

Also thickened the two finest tiers, since they were the ones vanishing at those same grazing
angles: `LINE_PIXEL_RATIO` 0.05→0.09, `LINE_CELL_RATIO` 0.03→0.05, plus a new screen-space minimum
(`MIN_LINE_SCREEN_PX_PIXEL`/`_CELL`, 1.4px/1.6px) clamped in before the world-unit conversion —
line width was previously a pure fraction of each tier's spacing with no floor, so it could
foreshorten toward sub-pixel as the bent surface angled away from the camera. Tile tier untouched,
it wasn't the one going invisible.

### Third pass — spacing derived by exact multiplication, not three separate roundings

The remaining "almost aligned" gap (worse toward the edges, fine near center) wasn't a phase/
offset problem — it was that `spacingPixelPx`, `spacingCellPx`, and `spacingTilePx` were each
computed independently as `W * FRACTION_*`, then each independently multiplied by `unitsPerPx`
for the actual shader uniforms. `FRACTION_CELL`/`FRACTION_PIXEL` etc. are exact fractions of each
other on paper, but three separate floating-point rounding chains computing "the same" ratio
don't always land on bit-identical results, and any tiny mismatch compounds with distance from
the origin — hence fine in the middle, a slowly widening sub-pixel gap toward the edges.

Fixed by deriving downstream tiers directly from the smallest by multiplication rather than each
being computed from scratch: `spacingCellPx = spacingPixelPx * 8`, `spacingTilePx =
spacingCellPx * 4`, and the same chain again for the actual world-unit uniforms
(`uSpacingCell = uSpacingPixel * 8`, `uSpacingTile = uSpacingCell * 4`). ×8 and ×4 are exact
powers of two, so these multiplications are lossless — cell lines now land exactly on pixel
lines and tile lines land exactly on cell lines everywhere on the plane, by construction, rather
than by two independently-rounded numbers happening to agree.

### Fourth pass — the real bug, found by actually rendering and measuring it

Previous passes fixed real issues (summed blending, self-overlap from depthWrite, spacing
precision) but none of them were *this* bug, which is why it kept looking misaligned. Rendered
the page headlessly (Playwright) and measured actual on-screen line-center positions per tier
instead of reasoning about it blind — found each tier's lines were centered on **half-integer**
multiples of its own spacing (`uv = (n + 0.5) * spacing`), not on multiples of a shared origin
(`uv = n * spacing`). That's an artifact of the old `gridLine()`'s distance function
(`fract(g) - 0.5`), which was never a bug on its own for a single tier — but "half a period" is a
different absolute distance for every tier since they have different spacings, so no ratio of
spacings, however exact, could ever bring them back onto a shared origin. Cell lines were
landing exactly half a *pixel*-period away from pixel lines, everywhere, at every zoom — not a
precision drift, a constant structural offset.

Fixed by changing the distance function to `fract(g + 0.5) - 0.5`, which centers on integer
multiples instead — all three tiers now measure from the same anchor (`uv = 0`), so the
exact-multiple spacing relationship from the earlier pass (cell = 8×pixel, tile = 4×cell)
actually does what it was supposed to: every cell line sits exactly on a pixel line, every tile
line sits exactly on a cell line.

Also caught and fixed a second, unrelated bug along the way: an editing pass had put literal
backtick characters inside a code comment *inside* the JS template-literal string holding the
shader source, which silently truncated the shader and broke the page. Verified the fix by
extracting the shader string with Node and confirming it parses as one contiguous block.

### Restructured to 3 tiers: dropped old largest, shifted everything down one level, added a new finest tier

Per request: the old largest ("tile") tier is gone. The old medium ("cell") tier is now the
largest visible tier. The old finest ("pixel") tier is now the middle tier. A brand-new finest
tier was added below that, at 1/4 the spacing of the old finest tier — so the new middle tier
contains a 4x4=16 grid of the new finest tier, and the new largest tier contains an 8x8=64 grid
of the new middle tier (same ratio the old cell-to-pixel relationship already used, just shifted
up one level since the new middle tier reuses the old finest tier's exact spacing).

Implementation-wise this didn't touch the shader at all — `uSpacingTile`/`uSpacingCell`/
`uSpacingPixel` keep their names and just resolve to different numbers now, still derived
bottom-up as exact integer multiples (pixel → ×4 → cell → ×8 → tile) for the same
exact-alignment reasons as the earlier fix. Also bumped the new largest tier's line width
(`LINE_TILE_RATIO` 0.05→0.065) since it's now the boldest tier on screen and needed to read as
such. The new finest tier got its own thickness ratio and screen-space floor
(`LINE_PIXEL_RATIO`, `MIN_LINE_SCREEN_PX_PIXEL`) so it doesn't disappear at grazing angles,
same protection the old finest tier had.

Verified the new ratios are exact (not just visually close) by replicating the spacing math in
a standalone Node script rather than trusting a screenshot — at this scale the finest tier's
lines are only a few screen pixels apart, which makes pixel-sampling a rendered screenshot an
unreliable way to check alignment (aliasing dominates). The math confirms distance-to-nearest-
line is exactly 0 (to float precision, ~1e-16) at every tier boundary.

### "Fix antialiasing" by making the grid bigger — root cause was sub-pixel spacing, not AA settings

The new finest tier from the previous pass (1/4 of the old finest tier's spacing) worked out to
under 2 screen pixels between lines at a typical window width. Below that, no antialiasing
setting — MSAA, the shader's smoothstep band, a higher device pixel ratio — can keep adjacent
lines visually distinct, because they're closer together than the display can actually resolve;
the result is shimmer/moiré, which looks like "bad antialiasing" but is really a spacing problem
one level up. Also worth noting: the plane geometry's literal size (`MARGIN`) has no effect on
line density at all — spacing is computed from the window width directly — so "make the
background bigger" needed to mean the grid's base unit, not the mesh.

Added a `GRID_SCALE` multiplier (3) applied to the base fraction all three tiers derive from, so
they grow together and every ratio/alignment fix from earlier passes is untouched. At a 1600px-
wide window this brings the finest tier from ~1.7px to ~5px between lines, middle tier ~6.75px→
~20px, largest ~54px→~162px. Verified via the same standalone spacing calculation used in the
previous pass rather than eyeballing a screenshot, since a moiré-prone grid is exactly the case
where a screenshot is least trustworthy for confirming a fix.

### Thinner main border (locked to 2x medium), gentler and smoother wobble

The largest tier's line width was set by its own independent ratio (`LINE_TILE_RATIO`), which
at a typical window width worked out to roughly 5.8x the medium tier's width (10.5px vs 1.8px)
— way thicker than intended. Replaced that with a direct multiplier of the medium tier's actual
computed width (`uLineTile.value = uLineCell.value * TILE_LINE_VS_CELL`, with
`TILE_LINE_VS_CELL = 2`), so the large tier's line is always exactly 2x the medium tier's,
whatever the medium tier's width happens to resolve to (including when it's sitting on its own
screen-px floor) — the two can't drift apart the way two independent ratios could.

Also toned down the animated wobble: amplitude (`WOBBLE_RATIO`) 0.03→0.02 for slightly less
movement, and both the time coefficient (0.04→0.022) and spatial frequency (0.01→0.006) lowered
so the drift reads as slower and broader/smoother rather than a quicker wiggle.

### Procedural color grading (FBM-based, dark charcoal → pastel green #A7E8B5) + more realistic pixel tier

Added real FBM noise to the background for the first time — none existed before this pass despite
the "folded paper" framing; the bend/fold was purely analytic (sin/cos), no noise anywhere. Added:

- A shared hash/value-noise/FBM GLSL snippet (`noiseGLSL`), textually injected into both the
  vertex and fragment shaders via `${noiseGLSL}`, so displacement and coloring sample the literal
  same noise function at the literal same frequency rather than two independently-tuned look-alikes.
  `fbm4` (4 octaves) for the finer/displacement-linked noise, `fbm3` (3 octaves, cheaper) for the
  large-scale and ambient layers.
- **Vertex shader**: a subtle FBM-based crinkle added to `bent.z` on top of the deliberate
  analytic fold — texture, not shape (`NOISE_DISP_AMOUNT` ≈ 0.077 world units, small relative to
  the 2.2-unit fold). Also exposed `vCurvature = edge*edge` (the same term the fold displacement
  itself uses) as a varying — real geometric curvature, not a noise-derived proxy.
- **Fragment shader color grading**: `nuv = vLocal/uScale` recovers the vertex shader's
  normalized local coordinate (window-size independent). Three noise/curvature terms — a
  large-scale FBM (`nLarge`, freq 0.55), a slow ambient FBM (`nAmbient`, freq 0.22, drifts with
  time), and the analytic curvature (`vCurvature`) — combine into `rawMask`, then
  `smoothstep(0.585, 0.62, rawMask)` for a very smooth threshold, then multiplied by a radial
  `edgeFalloff` (based on `nuv * MARGIN`, rescaled so r≈0.5 lands near the actual visible frame
  edge rather than the oversized MARGIN=1.6 mesh's own edge). Final mask mixes `darkBase`
  (#0a0a0a, the page's own theme color) toward `warmAccent` (#A7E8B5). This fill is the new base
  layer the grid lines composite over, instead of flat transparent.
- **Weight calibration note**: summing several already-0–1-normalized FBM layers as a true
  weighted average (weights summing to 1) clusters tightly around their shared mean — simulated
  outside the shader and found the naive version only ever reached ~0.29–0.38, nowhere near a
  usable threshold. Rescaled to unnormalized weights (0.7/0.5/0.25) so the combined signal
  actually spans a usable range, then derived the smoothstep thresholds from the simulated
  distribution rather than guessing.
- **Pixel-tier realism**: per-cell hash (`hash21` on the finest tier's cell index) gives each
  little square its own fixed brightness (0.35–1.0) and its own twinkle phase, rather than every
  line reading identically bright — an interpretation of "more realistic, Alche-style" pixels as
  an unevenly-lit, individually-twinkling grid rather than a uniform mechanical one. (Couldn't
  browse Alche's actual live WebGL canvas to confirm pixel-level treatment — this is a reasonable
  interpretation grounded in the file's existing "alche-style" framing, not a verified match.)

**A real bug found and fixed along the way, unrelated to this feature's own logic**: extensive
debugging (isolated via a Node.js reimplementation of the exact noise math, and via staged
in-shader diagnostic overlays) found that ~5.7% of fragments were getting NaN-interpolated
varyings — confirmed present even with the new crinkle displacement removed, i.e. **pre-existing**
in the bent/folded mesh, not introduced by this pass. Root cause: this sandbox has no GPU
(`/dev/dri` doesn't exist), so all rendering here falls back to SwiftShader software rasterization,
which appears to mishandle interpolation for the near-degenerate triangles the strong bend/fold
produces at the extreme edges — and NaN comparisons (`x != x`) don't reliably behave per spec in
this same software rasterizer either, which is what made this so hard to pin down (a direct
`if(mask > 0.5)` branch reproducibly failed to trigger for a value that displayed as 0.949 when
shown directly as a color). Added defensive NaN guards on `nuv` and `vCurvature` regardless —
correct, harmless GLSL that will work properly on any spec-compliant GPU driver — but **could not
get a fully reliable pixel-level visual confirmation of the final on-screen result in this
environment**, since it has no real GPU to render on. The shader compiles and runs with zero
console/page errors, and the underlying math is verified correct in isolation; if the green accent
doesn't look right (too rare, too common, wrong spots) in an actual browser, that's most likely a
threshold-tuning question (cheap to adjust) rather than the structural bug this pass already found
and fixed.

### Fourth pass — more coverage, faster movement, pixel-snapped, deeper gradient, ripple

Per request, several changes to the color grading at once:

- **Pixel-snapped sampling**: `uv` is now snapped to the finest grid tier's cell boundaries via
  `ceil(uv / uSpacingPixel) * uSpacingPixel` before any noise/ripple sampling — "appear only on
  pixels rounded up" — so the coloring reads as blocky patches aligned to the grid instead of a
  smooth wash that ignores it.
- **Faster movement**: `nLarge` and `nAmbient`'s time coefficients went from 0.008/-0.006 to
  0.05/-0.04 (roughly 6x), and `nLarge` now drifts over time too (it was static before).
- **Ripple**: a new term — a ring at `length(nuv) ≈ fract(uTime/1.6) * 0.9`, i.e. a radius that
  expands from center once every 1.6 seconds and resets, fading out as it expands
  (`rippleFade = 1 - phase`). Reads as a pulse that completes and restarts quickly rather than a
  steady-state wave, per "make the coloring end quickly and start again... behave like a ripple."
- **Deeper gradient**: added a third color stop — dark (#0a0a0a) → deep green (#1E402E) → pastel
  highlight (#A7E8B5) — via two staged `smoothstep`-based mixes, instead of a flat two-color lerp.
  Fill alpha raised too (0.12-0.5 → 0.14-0.62) for more presence.
- **More coverage**: rawMask's weights were all raised (nLarge 0.7→0.9, curvature 0.5→0.6,
  nAmbient 0.25→0.35, nDisp 0.15→0.2) and the new ripple term (weight 0.8) added on top. This
  shifted the formula's whole output range up substantially — simulating it (same Node.js
  approach as the earlier alignment/spacing fixes) found the new range sits around 0.63-1.4
  depending on ripple/curvature, versus the old formula's ~0.48-0.65 — so the threshold band was
  **recalibrated from scratch** against the new simulated distribution
  (`smoothstep(0.95, 1.2, rawMask)`) rather than nudged from the old one; the old threshold sat
  entirely below the new range and would've saturated to 100% coverage at all times, which
  reads as a flat wash, not "more of the accent." The recalibrated band gives roughly 25-50%
  coverage depending on where the ripple currently is — a clear increase from the previous
  pass's ~6-13%, while still leaving contrast and a visible pulse.

Verified via the same two-track approach as the previous pass: shader compiles and runs with
zero console/page errors, and the coverage math is checked by replicating the exact formula in
a standalone Node script (not just eyeballed) — same reasoning as before about this sandbox
having no GPU and unreliable NaN-comparison behavior under software rendering, so simulation is
the more trustworthy check available here for anything about the *shape* of the distribution.
Actual on-screen appearance (color balance, ripple timing feel, whether the pixel-snapping reads
as intended) still needs a real-browser check.

### Ripple retuned: less frequent, squiggly instead of a perfect circle

`ripplePeriod` 1.6s → 4.2s, so the pulse reads as occasional rather than constant. The ring
itself is no longer a perfect circle — `rippleRadius` now gets an angle-dependent noise offset
(`fbm3` sampled around `cos/sin(atan(nuv.y,nuv.x))`, drifting slowly over time so the squiggle
shape itself evolves rather than repeating identically every cycle) — "more squiggly/ripply"
instead of a clean expanding ring.

### Randomized ripple wobble + more transparent medium squares

- Ripple squiggle now gets a per-cycle random seed (`hash21` of the cycle index, `floor(uTime/ripplePeriod)`) added into the noise sampling coordinate, so each pulse's wobble shape differs from the last instead of the same evolving pattern repeating every cycle.
- Medium ("cell" tier) squares' line alpha: 0.28 → 0.14, for more transparency.

### Ripple attack curve: snaps in at center instead of easing in

Added `rippleAttack = smoothstep(0.0, 0.05, ripplePhase)`, multiplied into `rippleFade`, so the
ripple's brightness ramps up sharply over just the first ~5% of each cycle instead of already
being at full strength the instant the cycle starts — reads as popping into existence at the
center ("appear out of nowhere in the middle") before expanding outward and fading over the
rest of the cycle.

### Vercel deployment investigation

`https://vessel-five-alpha.vercel.app/` returns a platform-level `NOT_FOUND` (plain text,
`x-vercel-error: NOT_FOUND` header) on every path checked — `/`, `/index.html`,
`/vessel-studio.html` all identical. That's Vercel itself saying nothing is being served, not an
app-level 404 page. Combined with a reported ~1 second deployment time (implausibly fast for any
real build/upload), this points to either a build that failed instantly, an empty/no-op deploy,
or the domain not being aliased to any successful deployment — needs the actual Vercel dashboard
deployment log to confirm which. Couldn't get the Vercel MCP connector's tools to load despite it
showing "connected," and no GitHub connector or connected browser was available in-session to
push directly — files were handed off for the person to push manually instead.

### Root cause found and fixed: filename mismatch, not a build failure

Connected Claude for Chrome and pushed directly. The GitHub repo (`0jz/Vessel`) had the file
committed as `vessel-studio_37.html`, not `index.html` — Vercel's static serving has nothing to
route `/` to without an `index.html` at the root, which is exactly why every path 404'd
identically. Renamed the file to `index.html` via GitHub's web editor and committed directly to
`main`; Vercel auto-redeployed from the push and the domain started returning HTTP 200
immediately. The ~1 second deploy time was a red herring in the end — not a failed build, just a
trivial static deploy with nothing build-step-related to do.

(Also worth noting for next time: the browser was initially logged into the `neskeee` GitHub
account, which doesn't have write access to `0jz/Vessel` — GitHub offered a fork instead of
letting the edit go through. Switching the browser to the `0jz` account resolved it.)

### Full latest content pushed to GitHub, confirmed live

Pushed the complete current `index.html` (ripple, color grading, pixel realism, all of it) via
the same browser session — transmitted as gzip+base64 (54KB → ~24.5KB) to cut the browser
round-trip cost, reconstructed and decompressed client-side via `DecompressionStream('gzip')`,
copied to clipboard, and pasted into GitHub's editor. Verified the paste was complete by line
count (1053 in the editor matching the local file's 1052+trailing-newline) rather than trusting
`.cm-content` text length, which only reflects CodeMirror's virtualized/visible lines and
under-reports for any file long enough to scroll. Commit landed
("Enhance procedural color grading and ripple effects"), Vercel auto-redeployed, and the live
site was confirmed serving the new version by checking for `ripplePeriod` (a string unique to
the latest ripple code) in the deployed HTML.

## Todo / open items

- [ ] Replace placeholder studio name, copy, and work items with real ones (PassportSOL, SlopGate,
      UNSEEN, Hook Autopsy would all fit this treatment).
- [ ] Add the actual video/gif background in the Vision section behind the glass panel (structure
      is already in place for this).
- [ ] Consider whether the background curve should be smoother/more continuous — a physical paper
      fold was used as a reference at one point and the current geometry only approximates it via
      a single radial pull; a proper continuous surface warp would need a different approach than
      per-line quadratic bows if it needs to get closer to that reference.
- [ ] Decide on final background stretch direction (width- vs height-dominant) once viewed on your
      actual target screen size.
- [ ] Test performance on lower-end / mobile devices — 58 physics objects + continuous raycasting
      on every mousedown hasn't been profiled outside this environment.
- [ ] Real contact/recruit links in the footer and nav are currently `href="#"` placeholders.
- [ ] Consider whether the barrel-shaped physics walls need further tuning once real content
      (and therefore real scroll length) is in place.
