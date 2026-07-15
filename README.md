# Shadow

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
volumes** panel at the bottom right:

- **+ Box** adds a cube-shaped shadow region.
- **+ Cylinder** adds a cylindrical shadow region (axis = Y).

Each volume is a normal scene object, so the same gizmo lets you
**move / rotate / scale** it (click its row in the panel, or click the
wireframe in the viewport). Splats that fall inside the volume get
darkened with a smooth SDF falloff — darkest at the center, fading to
no effect at the edge.

**Every volume is tuned independently.** Rather than a slider on every
row, the **Shadow volumes** panel has a single pair of sliders that
retarget to whichever volume is currently selected (click a row, or
click the wireframe in the viewport). Adjusting them affects only that
one volume:

- **Intensity** — `0` = no shadow, `1` = pure black at the volume's
  center.
- **Softness** — controls how wide the smooth falloff zone is in the
  volume's local space. Larger = softer / cloudier shadow. Smaller =
  hard interior with quick falloff at the edge.

By default a volume is global — it shadows every loaded splat. To make
a volume affect **only one object**, set its **Parent** to that object
(see below): the shadow is then scoped to that object alone, so you can
give each box its own volumes without darkening the others. Volumes
stack with the brush mask (a splat inside a volume AND painted by the
brush gets both effects). When you **Save shadowed SPZ**, each volume's
contribution is baked into whichever objects it affects, so the
exported file already has the shadow burned into the splat colors.

The shader supports up to `MAX_VOLUMES` (32) at once.

### Parenting volumes to objects

Each volume row has a **Parent** dropdown that lets you attach the
volume to one of the loaded splat objects (or leave it under the scene
root with `(scene)`). Parenting does two things:

1. **Scopes the shadow** — a parented volume darkens **only** that
   object, leaving every other splat untouched. `(scene)` makes it
   global again (darkens everything).
2. **Follows the transform** — moving, rotating, or scaling the splat
   with the gizmo carries the volume along with it, so the shadow stays
   glued to the object.

Parenting is a true scene-graph relationship (`Object3D.attach`),
which means:

- The volume's world position is preserved at the moment you change
  parents — switching parents doesn't visually shift the shadow.
- Scaling the parent splat uniformly scales the volume's effective
  reach in world space (the shadow grows with the object).
- Removing a parented splat orphans its volumes back to the scene
  root, again preserving their current world transform — the shadows
  stay where they were.

## Controls

| Key             | Action                                |
| --------------- | ------------------------------------- |
| `1`             | Shadow brush mode                     |
| `2`             | Undo brush mode (restores original)   |
| `Esc`           | Orbit / view mode                     |
| `-` / `=`       | Decrease / increase brush radius      |
| `[` / `]`       | Decrease / increase brush depth       |
| Mouse drag      | In view mode: orbit; in brush modes: paint |

The GUI also exposes:

- **Brush → Shape** — `Circle` (cylindrical brush) or `Square` (rectangular prism, screen-aligned).
- **Brush → Size / Depth** — radius (or square half-edge) and how far the brush extends along the ray.
- **Shadow strength** — full-stroke target darkness (`0` = no effect, `0.5` = aggressive). Pure per-channel darken, so hue & saturation are preserved.
- **Flow (per-stamp)** — how much of the strength each individual stamp deposits (low = airbrush-style gradual buildup, high = heavy stamp). Combined with cursor-distance throttling, this stops slow drags from piling up dark blobs in one spot.
- **Edge softness** — `0` = sharp brush boundary, `1` = full smooth falloff from center to edge. Soft edges naturally build penumbra over multiple strokes because edge splats are darkened less per pass.
- **Cool tint (skylight)** — `0` keeps the shadow neutrally darker; higher values shift the shadow slightly cool/blue like real ambient skylight in shadow areas.
- **Reset shadows on selection** — clears the selected object's shadow mask
  back to identity. Other objects are unaffected.
- **Add object** — load another `.spz` / `.ply` / `.splat` / `.ksplat` into the scene.
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
near the camera render at full resolution. Toggling LOD reloads every
loaded object so each one picks up an LOD pyramid (or drops it).

LOD internally reorders splats — the rendered splat index points into a
merged array, not the original — and our per-splat shadow mask is keyed by
the original index. Painting under LOD would land on the wrong splats. To
square that with the brush, the app uses a **selection-aware LOD policy**:

- The **currently selected object runs without LOD** so the brush hits the
  splats you click on.
- Every other (unselected) object keeps its LOD pyramid so the rest of the
  scene around it stays cheap.

Switching selection is a single uniform flip — no reload, no rebuild.

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

The selection-aware LOD policy (above) is what guarantees the mask indices
line up with the renderer: with LOD off on the selected mesh, Spark uses
`packedSplats` directly so `gsplat.index === packed splat index === mask
index`. The bake updates the same mask entries the renderer reads, so what
you see is what you painted.
