# Alignment Engine

Implements `references/orthographic_calibration.md` as executable Blender Python.

## Responsibilities

- Solve `units_per_pixel` per view.
- Assign scene axes and ground plane.
- Create calibration cameras and reference planes.
- Persist calibration JSON into the .blend.

## Inputs

- Classified views from `reference_analyzer`.
- Optional `known_dimension`: `{view, axis: "width"|"height", length: float, unit: "m"}`.

## Outputs

- `scene["og_calibration"]` JSON.
- Three orthographic cameras: `CAM_REF_FRONT`, `CAM_REF_SIDE`, `CAM_REF_TOP`, each with `ortho_scale` computed so 1 BU = calibrated units.
- Three image planes in `REF_VIEWS` collection, placed behind cameras at fixed depth, used only for visual reference.

## Execution Logic

### 1. Mask Acquisition

Call `silhouette_engine.extract_mask(image)` for each view (dependency: silhouette engine runs its segmentation stage early; ordering note in `autonomous_pipeline.md`).

### 2. Extent Measurement

```python
def mask_extent(mask):
    ys, xs = np.nonzero(mask)
    return xs.min(), ys.min(), xs.max(), ys.max()
```

### 3. Scale Solve

Implement the three cases from `orthographic_calibration.md` §Step 4. For Case B/C, solve via least squares:

```
minimize Σ_pairs (extent_shared_a / s_a - extent_shared_b / s_b)^2
```

Use `numpy.linalg.lstsq` on the linearized system; if a known dimension exists, it pins one `s_v` absolutely.

### 4. Axis Map

```python
AXIS_MAP = {
  "front": {"img_x": "+X", "img_y": "-Z"},   # image up = +Z
  "side":  {"img_x": "-Y", "img_y": "-Z"},   # flip x if right-side view; verify via asymmetric feature or default
  "top":   {"img_x": "+X", "img_y": "-Y"},   # image down = +Y? fix: image row increases downward → world -Y if top view drawn as plan
}
```

Document chosen convention in calibration JSON; validation renders must use the same map.

### 5. Camera Setup

```python
cam.data.type = 'ORTHO'
cam.data.ortho_scale = max(bbox_world_x, bbox_world_y_for_top) * 1.05
```

Position cameras on each axis at distance 100 BU, aimed at calibrated center. Lens shift zero; use `shift_x/shift_y` if centering offset demands.

### 6. Confidence Score

```
confidence = 1.0 - max_relative_extent_disagreement
```

Clamp to [0,1]. Below 0.7 → pipeline emits `calibration_warning`.

## Error Handling

- Height disagreement > 3% after re-segmentation → return `status: "inconsistent_views"` with discrepancy table; do not proceed to geometry.
- Single view only → set `partial_calibration: true`; reconstruction proceeds in "relief mode" (2.5D extrusion) and quality gates relax silhouette IoU to ≥ 0.85 on available view only. Report limitation.

## Dependencies

Consumes `reference_analyzer` output and `silhouette_engine` masks; feeds `geometry_engine`.
