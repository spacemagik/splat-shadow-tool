# Shadow

**Live demo:** <https://splat-shadow-tool.netlify.app>

A tiny [three.js](https://threejs.org/) + [Spark 2.1](https://sparkjs.dev/) web app
with a single brush tool that casts **shadows** on a 3D Gaussian Splat scene by
darkening splats inside the brush volume — preserving their hue so the shadow
looks like a natural darker version of the surface, not a grey smudge.

It's a minimal adaptation of Spark's [Splat Painter example](https://sparkjs.dev/examples/#splat-painter):
instead of replacing color with a brush color, every stroke multiplies splat
RGB by a per-channel darkness vector (optionally with a slight cool tint to
mimic ambient skylight in real shadows).

## Run it

The app is a single static `index.html` that loads three.js and Spark from CDN
via an importmap, so any static file server works. The simplest option:

```bash
npm start
```

then open <http://localhost:4321>. Or use any other static server, e.g.
`python3 -m http.server 4321`.

## Loading splats

The app starts on a drop zone — there's no bundled splat. To load one:

- **Drag &amp; drop** any `.spz`, `.ply`, `.splat`, or `.ksplat` onto the page, or
- click **Choose splat file…**, or
- use the **Add object** button (in the Objects panel or the GUI) to add more.

Files are loaded via `URL.createObjectURL` directly from your machine; nothing
gets uploaded.

### Multiple objects

Drop multiple files (a world, chairs, tables, etc.) and each one becomes its
own entry in the **Objects** panel on the top-left. The currently selected
object (highlighted in amber) is the one the brush paints on — clicking
another object switches the target instantly, and each object keeps its own
shadow mask so you can paint shadows independently on each asset. The `×`
button in a row removes that object from the scene.

### Selection &amp; transform gizmo

Each loaded object can be selected three ways:

- Click its row in the **Objects** panel.
- Click the splat itself in the viewport while in **Orbit** mode.
- Drop a new file (the new object auto-selects).

The currently selected object gets a 3-axis transform gizmo so you can
translate / rotate / scale it directly in the viewport. Click on empty
space (orbit mode) to deselect — the gizmo disappears.

Switch the gizmo's mode using:

- the **Move / Rotate / Scale** toolbar at the top of the Objects panel,
- the **Gizmo** folder in the GUI (also exposes world / local space and
  handle scale), or
- hotkeys: `G` for translate, `R` for rotate, `Y` for scale.

While you're dragging a gizmo handle the camera **does not** orbit — but
WASD / QE keep flying as normal so you can reposition the camera mid-edit.

The brush respects the gizmo: paints are applied in each object's own
local space, so shadows stay attached correctly even after you move,
rotate, or scale the splat.

## Shadow volumes (boxes &amp; cylinders)

For shadows that don't conform to a brush stroke — say, a soft cast
shadow under a piece of furniture, or a column of darkness inside a
room — drop **shadow volumes** into the scene from the **Shadow
volumes** panel (in the left column, under Objects):

- **+ Box** adds a cube-shaped shadow region.
- **+ Cylinder** adds a cylindrical shadow region (axis = Y).

Each volume is a normal scene object, so the same gizmo lets you
**move / rotate / scale** it — click its row in the panel to select it.
Splats that fall inside the volume get darkened with a smooth SDF
falloff — darkest at the center, fading to no effect at the edge.

**Only the selected volume shows its wireframe.** Splats don't write to
the depth buffer, so a wireframe can never be hidden behind the ground —
an always-visible helper would draw "through" the whole scene. Deselect
(click empty space or another object) and the box outline disappears,
leaving just the shadow it casts.

**Every volume is tuned independently.** Rather than a slider on every
row, the **Shadow volumes** panel has a single pair of sliders that
retarget to whichever volume is currently selected. Adjusting them
affects only that one volume:

- **Intensity** — `0` = no shadow, `1` = pure black at the volume's
  center.
- **Softness** — controls how wide the smooth falloff zone is in the
  volume's local space. Larger = softer / cloudier shadow. Smaller =
  hard interior with quick falloff at the edge.

A volume always darkens **every splat it overlaps**, no matter which
object those splats belong to — a shadow box parented to a car still
darkens the ground beneath the car, exactly like a real drop shadow.
Volumes stack with the brush mask (a splat inside a volume AND painted
by the brush gets both effects). When you **Save shadowed SPZ**, each
volume's contribution is baked into whichever objects it affects, so
the exported file already has the shadow burned into the splat colors.

Volumes are implemented with Spark's native SDF splat edits
(`SplatEdit` + `SplatEditSdf`), up to `MAX_VOLUMES` (32) at once.

### Parenting volumes to objects

Each volume row has a **Parent** dropdown that lets you attach the
volume to one of the loaded splat objects (or leave it free-standing
with `(scene)`). Parenting means the volume **follows the object's
transform** — moving, rotating, or scaling the splat with the gizmo
carries the volume along with it, so the shadow stays glued to the
object. It does NOT restrict what gets darkened: the volume keeps
shadowing everything it touches (ground, walls, other objects).

Parenting is **logical**, not a scene-graph `attach`: the volume's SDF
node stays at the scene root, and each frame its transform is
recomposed from the parent's world matrix and a stored relative offset.
(A true scene-graph parent would introduce shear when the parent is
rotated and non-uniformly scaled, which distorts the shadow shape —
keeping the SDF as a pure translate/rotate/scale avoids that.) In
practice this means:

- The volume's world position is preserved at the moment you change
  parents — switching parents doesn't visually shift the shadow.
- Moving / rotating / scaling the parent splat carries the volume
  along with it, so the shadow stays glued to the object.
- Removing a parented splat orphans its volumes back to the scene
  root, again preserving their current world transform — the shadows
  stay where they were.
- Toggling LOD (which reloads every object) preserves parent links:
  volumes re-attach to the reloaded objects automatically.

## Controls

| Key             | Action                                |
| --------------- | ------------------------------------- |
| `W A S D`       | Fly forward / left / back / right     |
| `E` / `Q`       | Fly up / down (world vertical)        |
| `Shift` (hold)  | Fly 5x faster (`Ctrl` = 1/5 slow)     |
| `1`             | Shadow brush mode                     |
| `2`             | Eraser brush mode (brushes shadows away) |
| `Cmd/Ctrl + Z`  | Undo last stroke / gizmo drag / reset / added volume |
| `Esc`           | Orbit / view mode                     |
| `-` / `=`       | Decrease / increase brush radius      |
| `[` / `]`       | Decrease / increase brush depth       |
| Mouse drag      | In view mode: orbit; in brush modes: paint |

Fly speed auto-scales to the size of the first loaded splat and can be
tuned with **Scene → Fly speed (WASD/QE)**.

**Projects (autosave)** — the whole scene (loaded files, object positions,
painted shadows, shadow volumes + parenting, camera, fly speed) is
continuously saved to the browser's IndexedDB as a *project*. The start
screen lists your recent projects — click one to load it, or `×` to delete
it. Starting a new scene creates a new project; older ones stay available.
Autosave is best-effort: in private browsing or with storage disabled the
tool still works, projects just won't survive a refresh.

**Undo (`Cmd/Ctrl+Z`)** steps back through your recent actions: brush
strokes (shadow and eraser), gizmo moves/rotates/scales of objects and
shadow volumes, shadow resets, and added volumes. The **Eraser** brush
(`2`) instead brushes painted shadow away selectively — it erases faster
than the shadow brush paints and always restores the original colors
exactly. Note that neither can remove shadows that were already baked
into a file by a previous save; those are part of the splat colors.

The GUI also exposes:

- **Brush → Shape** — `Circle` (cylindrical brush) or `Square` (rectangular prism, screen-aligned). The brush outline is always visible on the surface under the pointer, sized to the exact painted area (white in Shadow mode, blue in Eraser mode).
- **Brush → Size / Depth** — radius (or square half-edge) and how deep beyond the surface the brush penetrates, both in world units. The brush anchors on the splat surface it hovers, so Depth is independent of how far away the camera is.
- **Shadow strength** — full-stroke target darkness (`0` = no effect, `0.5` = aggressive). Pure per-channel darken, so hue & saturation are preserved.
- **Flow (per-stamp)** — how much of the strength each individual stamp deposits (low = airbrush-style gradual buildup, high = heavy stamp). Combined with cursor-distance throttling, this stops slow drags from piling up dark blobs in one spot.
- **Edge softness** — `0` = sharp brush boundary, `1` = full smooth falloff from center to edge. Soft edges naturally build penumbra over multiple strokes because edge splats are darkened less per pass.
- **Tint (warm ↔ cool)** — `0` keeps the shadow a pure neutral darken; positive values grade it cool (evening-blue skylight), negative values grade it warm (golden-hour amber). The tint works like a colorist's shadow grade — it blends the darkened area toward the tint color while preserving luminance — so cool shadows actually read blue even on strongly warm surfaces (and vice versa), instead of just graying out. It's applied at render time, so moving the slider **re-colors every shadow you've already painted** live — no repainting needed. The tint you see is what gets baked on Export.
- **Reset shadows on selection** — clears the selected object's shadow mask
  back to identity. Other objects are unaffected.
- **Add object** — load another `.spz` / `.ply` / `.splat` / `.ksplat` into the scene.
- **Save → Record 7s showcase video** — records a cinematic 7-second clip
  of the current scene: all UI is hidden, the camera does a slow orbital
  pull-in, and the hero object (the selected prop, or the smallest object
  in the scene) gently lifts, drifts and settles — with any parented shadow
  volumes tracking it live. The clip downloads as MP4 (H.264) where the
  browser supports it, otherwise WebM, at the canvas resolution — maximize
  the window or go fullscreen for a higher-res video. Frame your shot
  first; the move starts from the current camera view. `Esc` ends the take
  early. Scene and camera are restored exactly afterwards.
- **Save → Save shadowed SPZ** — bake **every loaded object** into a single
  `.spz` file in world space and download it. Each object's gizmo placement
  (translation / rotation / scale) is composed into its splats so the export
  matches what you see in the viewport — chairs sit on the floor, parented
  shadow volumes have already darkened the splats, etc. Spherical harmonics
  are not preserved on export (degree 0). Adjust **SPZ precision**
  (fractionalBits) if the encoder reports clipped splats — the warning shows
  in the browser console.

## Performance — large splats (20M+)

Spark's [Level-of-Detail tree](https://sparkjs.dev/2.0.0-preview/docs/lod-getting-started/)
is **opt-in** behind the **LOD (fast 40M+ splats)** checkbox in the Scene
folder. Default is **off** because the LOD pyramid replaces the rendering
path even when toggled per-mesh, and certain captures end up rendering as
an empty / black scene with LOD on. For most shoots the brush works fine
without it.

For very large scenes (20M+ splats), tick the LOD box: distant / small
splats get merged into coarser representatives at runtime so only splats
near the camera render at full resolution. Toggling LOD (either direction)
reloads every loaded object so each one is rebuilt with (or without) an
LOD pyramid — your gizmo transforms, selection, and volume parent links
are preserved across the reload, but painted shadow masks are reset.

LOD internally reorders splats — meshes loaded with LOD keep their base
splat array empty (all data lives in the merged pyramid) — so the
per-splat shadow mask and the SPZ exporter, which are both keyed by the
original splat order, can't address them. While LOD is on:

- **Painting is disabled** (the brush buttons grey out and hotkeys `1` /
  `2` are blocked, with a tooltip explaining why).
- **Save shadowed SPZ is blocked** with the same guidance.

The intended flow: fly around and place things with LOD on for speed,
then untick LOD to paint shadows and export.

Tune **LOD detail** between `1.1` (sharp / more splats kept) and `2.0`
(aggressive merging / fastest) from the same Scene folder.

## How it works

Each loaded object owns a per-splat **shadow mask** — an `RgbaArray` sized
to the original splat count, initialized to `(1, 1, 1, 1)` (no shadow).

The brush defines a cylinder in world space (origin = camera, direction =
ray through cursor, radius + depth from sliders). On every `pointermove`
during a drag, a Spark `dynoBlock` runs per-original-splat on the GPU and:

1. Reads the splat center from the original packed splats array (object space —
   the brush ray is converted from world to object space before each bake so
   gizmo translations / rotations / scales don't shift the paint).
2. Tests whether the splat center lies inside the brush volume, smoothing out
   to 0 across an `edgeSoftness`-controlled fall-off zone for soft brushes.
3. If so, multiplies the existing mask entry by a darkness vector (`darkness`
   on every channel — uniform darkening preserves hue/saturation; optional
   `coolTint` mixes in a slight blue shift toward `(0.82, 0.92, 1.05)`).
4. The renderer's `worldModifier` then reads `mask[index]` per render frame
   and multiplies the live RGB by it, so the shadow appears immediately.

This mask-based scheme avoids the **8-bit hue drift** the naive approach has
(baking `base * darkness^N` straight into `splatRgba` quantizes each channel
independently — one channel hits the rounding floor first and the surface
takes on a green/blue tint after enough strokes). Because the mask values
stay equal across channels (when `coolTint = 0`), every channel quantizes in
lockstep and the only visible effect is value reduction.

The LOD paint lock (above) is what guarantees the mask indices line up
with the renderer: with LOD off, Spark uses `packedSplats` directly so
`gsplat.index === packed splat index === mask index`. The bake updates
the same mask entries the renderer reads, so what you see is what you
painted.

## License

[MIT](LICENSE)
