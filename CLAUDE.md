# Device Mockup Generator — project context

Single-file browser app: `video-mockup-generator.html`. No build step, no dependencies, no network calls.
Open it directly in Chrome or Safari (`file://`). It will **not** work inside a sandboxed preview pane —
CSP `default-src 'self'` blocks `blob:` media, so uploaded video never loads there.

## What it does

Loads a local video or image and composites it into a device frame on a canvas, then records the canvas
with `MediaRecorder` and downloads an MP4 (H.264 when the browser has an encoder, WebM otherwise) or a PNG.
Transparent exports are always WebM/VP9 — see "Video export".

## Architecture (all inside the one `<script>` IIFE)

- `state` — every UI control writes here; `draw()` reads only from `state`, `src` and `photo`.
- `src` — `{kind: 'video'|'image'|null, ar}`; media element resolved by `mediaEl()`.
- `metrics()` / `layout()` — device geometry is expressed as fractions of **screen width** (`sw`), so the
  whole device can be solved in one pass and always fits the canvas. `kw`/`kh` are the bounding box in
  units of `sw`. Never hard-code pixel sizes.
- `PANEL` — per-device bezel/chin/corner/stand spec. `standF` is a fraction of **display height**.
- Stands: `standPlate`, `standNeckFoot`, `standSlab`, `standColumn`. Each is painted from ONE gradient
  spanning its widest part, with parts overlapping — this is deliberate, see "gotchas".
- `drawPanelDevice`, `drawLaptop`, `drawWindow`, `drawPhotoDevice` — the four render paths.
- Photo mode: `homography()` maps the unit square onto the dragged quad; `drawTri()` does affine texture
  mapping per grid cell (10×10). Axis-aligned quads take a fast single-`drawImage` path.

## Proportions came from measured product photography

Do not eyeball these. Current values reproduce:

| Device | body:total height ratio | source |
|---|---|---|
| all-in-one, black bezel (27") | 0.807 | measured off the reference photo |
| all-in-one, slim (24") | 0.857 | Apple published dims give 0.843 |
| Studio-style display | 0.752 | Apple published dims give 0.767 |

## Lighting model

There is one key light, upper-left, with a weak fill on the right. Everything is shaded from that
premise — if you add a part, shade it the same way or it will read as pasted on.

- `metal()` is the anodised-aluminium sweep: dark grazing edge → **narrow** specular at ~0.235 → long
  falloff → a soft fill bounce at ~0.86 → dark edge. The specular must stay narrow. An earlier version
  put `#fbfbfc` across 0.22–0.48 and every stand read as white plastic.
- **Brightness follows surface angle, not part size.** Feet and bases are near-horizontal, so they point
  at the key and are the *brightest* metal on the device. Necks and columns are vertical and sit a step
  darker. Getting this backwards is the single most obvious tell.
- `cylinderShade()` gives necks and columns their round-in-section form. Safe to layer over the shared
  stand gradient because necks are always painted *before* the foot that overlaps them.
- **A flat, camera-facing face must not carry the specular hump** — it reads as a smudge. The imac24 slab
  is nearly as wide as its foot, so it takes almost the whole sweep; `standSlab` cancels the specular
  where it lands (~20% across the slab) and settles the slab to the brightness of the chin above it,
  which is the same aluminium in the same plane.
- `contactShadow()` is two ellipses: a wide ambient pool plus a tight, much darker core. The core is what
  reads as "sitting on a surface"; the pool alone is just a grey smudge.
- `aoUp()` darkens where one part sinks into another. Keep it to the last quarter of the part — spread
  over a whole face it turns thin ends black.
- `state.backdrop` (default on) grades the background: a pool of light behind the device falling off into
  the corners. Written as white/black alpha so it works over any swatch, light or dark. Skipped when
  `state.transparent`.

## Panel structure

Four `<details class="sec">` sections — Content, Device, Scene, Export. Which ones are open is kept in
`localStorage` under `mockup.sections`; Quick forces them all open and makes the summaries inert
(`.panel.quick`), so the open-set is only ever saved from Full.

## Modes

`state.mode` is `quick` or `full`. Quick exists for people who just need a mockup out the door, not for a
job title — the labels are deliberately about the task, and anyone can switch. Quick hides every
`[data-adv]` block, drops photo mode, and forces glare on and 60 fps. Visibility depends on **both** the
device and the mode, so it is resolved in one place (`refreshVisibility()`); two handlers each poking
`style.display` fight each other.

## Remembered settings

Everything the user picked is written to `localStorage` under `mockup.settings`, debounced via
`queueSave()` off capture-phase `input`/`change` listeners. `setBg()` calls it directly — swatches are
clicks, which those listeners miss. `restoreSettings()` deliberately writes to the **controls** and fires
their own events rather than assigning into `state`, so there is one path into `state` instead of a second
one that can drift. Restore order matters: colour before transparency, since picking a colour clears the
transparent flag.

## Output presets

`OUTPUTS` name the destination ("Slides & Slack", "Story / Reel") rather than a pixel count; non-designers
do not think in 1080×1350. Chips and the exact-size `#preset` select are two views of the same value, both
routed through `setCanvasSize()` so they cannot disagree.

## Trim and the timeline

`state.trimStart` / `state.trimEnd` in seconds, reset to the full duration on load. The playhead and both
trim handles share **one** track (`#tl`) — trimming and scrubbing are the same act on the same object.

- Handles are positioned in **px**, clamped a half-width inside the track. At 0% and 100% a percentage
  position leaves them hanging half outside it, where they are almost impossible to grab. This means
  `syncTrimUI()` must also run on window resize.
- Hit-testing is by **proximity** (`GRAB`), not `e.target`. A 14px handle is a small target and the two
  extremes sit right on the track edge.
- `tick()` enforces the range for both preview looping and recording; the recorder is stopped from there,
  which is what makes the exported file the trimmed length. There is also a wall-clock `setTimeout`
  backstop, because `tick()` only runs on animation frames and those starve under load.

## Empty state

The invitation to drop a file is an HTML overlay (`#empty`), not canvas-drawn text — it needs a real
button. It is full-bleed so the whole preview reads as the drop target, but translucent, so the device
frame behind it still previews what you are about to get. `drawScreen()` therefore only paints an unlit
panel when nothing is loaded.

## Video export

- **H.264 has no alpha channel** — in any browser, in any container. A transparent export therefore has
  to leave the MP4 family entirely: `pickMime()` returns WebM/VP9 when `state.transparent` is set.
  Verified by recording a cleared canvas and reading the decoded corner back as `rgba(0,0,0,0)`.
  Preferring MP4 unconditionally is what used to flatten transparent clips onto black.
- Grain is composited `source-atop` when transparent. `overlay` over an empty pixel still deposits alpha
  (`Sa + Da(1-Sa)` = 0.045), which left a ~4% haze over the whole frame and stopped exports being clear.
- **Seek, wait for a decoded frame, draw, *then* start the recorder** (`seekTo()`). `seeked` alone is not
  enough — `readyState` can still be below `HAVE_CURRENT_DATA`, and a `draw()` at that moment paints an
  empty screen. That is how the "choose a video or image" prompt used to end up as the export's thumbnail.
- `drawScreen()` only paints that prompt when `src.kind` is null; with a source loaded but momentarily
  undecodable it paints an empty panel instead. `drawPhotoDevice()` bails out entirely while exporting.
- Never gate the Export button on `mediaEl()`: `readyState` dips below 2 for the duration of every seek,
  so touching a control mid-seek left the button permanently disabled. Use `canExport()`.
- `#est` states the cost before the click ("about 12s"). Recording is real time, and a 40s clip silently
  taking 40s is this tool's least obvious behaviour.

## Gotchas

1. **Stand parts must overlap.** An earlier version had neck-to-foot gaps of 3–16% of stand height,
   which showed as floating slabs. Each neck now runs *into* the middle of its foot.
2. **One gradient per stand.** Per-part gradients produce a visible colour step at every seam. Also don't
   stroke individual stand parts — the outline draws a line through the join. Shading *overlays* on top of
   that shared gradient are fine — that is what `cylinderShade()` does.
3. **Stands must not overhang the box `layout()` solved for.** The `standNeckFoot` foot bow is symmetric
   about `fb` so its deepest point lands exactly on the layout bottom. Push it lower and it clips the
   moment padding goes to 0.
4. **No cusps on the foot.** The neck-foot's foot was a lens shape with sharp tips; the thin ends went
   black and it read as a paper cut-out. Ends are rounded now.
5. **The metal rim around the glass is a hairline** (`edgeF` ≈ 0.0013–0.0018 of `sw`). At 0.004 it reads
   as a fat white ring.
6. `state.exporting` hides the photo-mode corner handles during PNG and video export. Anything preview-only
   must check it.
7. Transparent background uses `clearRect`; the laptop's finger notch switches to `destination-out`
   compositing so it stays a real cut-out. Its scoop shading is skipped in that mode.
8. Canvas context is `alpha: true` — required for transparent PNG export.

## Brand

Background swatches are Loomery's actual Webflow colour variables:
`#15FFB9` green, `#00C0AA` legacy green, `#04564D` dark green, `#1A1B1F` black, `#181825` rebrand blue,
`#5B4EE9` blue. No logos on any hardware — this was an explicit request.

## Open items

- Device photos for photo mode need to be user-supplied; no image generation was available in the session
  where this was built.
- Floor reflection for the drawn frames was considered and skipped (cost per frame during recording).
- Video export is real-time only. Offline/faster-than-realtime encoding would need WebCodecs.

## Verification habits used here

Geometry changes were checked by script, not by eye — fitting every device across 6 canvas presets × 8
aspect ratios, and asserting stand parts overlap. Keep doing that; visual regressions here are subtle.

Shading changes need the opposite: look at them, zoomed in, one part at a time. Serve the folder
(`python3 -m http.server`) and drive the page from the browser — but note `requestAnimationFrame` is
throttled to zero while the pane is hidden, so `tick()` never redraws between calls. Change a control,
then take a screenshot to force the frame; reading `getImageData`/`toDataURL` without one gives you the
previous render. To inspect a detail, reparent the canvas to `<body>` and set `width`/`height`/`left`/`top`
with `setProperty(..., 'important')` — the stylesheet's `max-width` otherwise pins it to the pane.
