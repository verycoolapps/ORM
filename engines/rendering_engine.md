# Rendering Engine

Produces preview renders and handles export-adjacent scene preparation.

## Responsibilities

- Configure orthographic preview cameras (reuse calibration cameras).
- Render clay/preview images per view.
- Optional turntable render for user review.
- Prepare scene for export (units, apply transforms, collection visibility).

## Inputs

- Scene with final mesh, calibration cameras, clay material.
- Render settings: resolution, samples, engine preference.

## Outputs

- `preview_{view}.png` per view.
- Optional `turntable_{000..N}.png`.
- Export-ready scene state.

## Execution Logic

### 1. Camera Configuration

Calibration cameras already exist (`CAM_REF_*`). For preview renders, verify:

- `cam.data.type == 'ORTHO'`
- `ortho_scale` covers model bbox × 1.05
- Camera aimed at model center (use `track_to` constraint or manual quaternion)
- Resolution matches reference image aspect (set `scene.render.resolution_x/y` to ref dims; if engine requires square, use ref aspect and note letterboxing)

### 2. Render Settings

```python
scene.render.engine = 'BLENDER_EEVEE_NEXT'  # fallback 'BLENDER_WORKBENCH' if headless
scene.render.resolution_percentage = 100
scene.render.image_settings.file_format = 'PNG'
scene.render.film_transparent = False
scene.world.color = (0.18, 0.18, 0.18)  # or use Background node strength 0.5 gray
scene.view_settings.view_transform = 'Standard'  # no AgX shift for metric comparison
```

Workbench alternative (fastest, always available):

```python
scene.display.shading.light = 'STUDIO'
scene.display.shading.color_type = 'MATERIAL'
scene.render.engine = 'BLENDER_WORKBENCH'
scene.display.shading.show_shadows = True
scene.display.shading.show_cavity = True
```

### 3. Render Execution

```python
for view in views:
    scene.camera = cams[view]
    scene.render.filepath = f"{out_dir}/preview_{view}.png"
    bpy.ops.render.render(write_still=True)
```

Headless note: `bpy.ops.render.render` works backgrounded with EEVEE on GPU-less systems only if EGL/OSL available; else use Workbench (`BLENDER_WORKBENCH` renders headless reliably).

### 4. Turntable (optional)

Create `CAM_Turntable` on a circle radius = 1.5× max bbox dim, height = 0.5× height; animate or step 24–36 frames; render via `bpy.ops.render.render(animation=True)` or per-frame stills. Skip if `deliverable.spec.turntable == false`.

### 5. Export Preparation

```python
# ensure applied scale
bpy.ops.object.transform_apply(location=False, rotation=False, scale=True)
# units
scene.unit_settings.system = 'METRIC'
scene.unit_settings.scale_length = 1.0  # 1 BU = 1 m unless calibration says otherwise
# collection visibility for export
for c in ["REF_VIEWS", "BLOCKOUT"]: c.hide_render = True; c.hide_viewport = True (optional)
```

Actual file export is performed by `standards/production_delivery.md` procedures, which call `bpy.ops.export_scene.{obj,fbx,stl,gltf}` with correct flags.

## Error Handling

- Render engine unavailable → auto-fallback chain: EEVEE → Workbench → Cycles CPU (samples=16).
- Camera clipping (clip_start > distance) → set `clip_start=0.01`, `clip_end=1000`.
- Output dir missing → `os.makedirs(exist_ok=True)`.

## Dependencies

Runs after `modifier_engine`, parallel with `validation_engine` (validation uses its own silhouette renders; this engine produces human-review previews).
