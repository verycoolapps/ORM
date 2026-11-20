# Validation Engine

Executes all quality gates from `references/quality_standards.md` and produces the validation report.

## Responsibilities

- Compute silhouette metrics per view (mesh-projected and render-based).
- Geometric sanity checks (volume, bbox, self-intersection).
- Invoke topology checks (already run by topology engine; re-verify on final stack).
- Render orthographic preview images for visual gate.
- Write `validation_report.json` and preview PNGs.

## Inputs

- Final mesh + modifiers (post `modifier_engine`).
- Calibration + masks + projections.
- Reference images.

## Outputs

- `validation_report.json` in run output directory.
- `preview_{front,side,top}.png` renders.
- `diff_{view}.png` side-by-side reference vs render.
- Return `overall: pass|fail` + failure list.

## Execution Logic

### 1. Mesh Silhouette Metrics

For each view:

```python
sil = render_silhouette(cam_v, mesh_obj)         # object-index render → mask
ref = masks[view]
iou = iou(sil, ref)
b_err = chamfer(boundary(sil), boundary(ref)) / diag(ref)
h95 = hausdorff95(boundary(sil), boundary(ref)) / diag_obj_world
```

Note: mesh-projected silhouette uses the actual evaluated mesh (modifiers applied via depsgraph: `obj.evaluated_get(depsgraph).to_mesh()`).

### 2. Render-Based Check

Render clay preview per view (Eevee, 512–1024px matching ref res):

```python
scene.render.engine = 'BLENDER_EEVEE_NEXT'  # or 'BLENDER_EEVEE'
scene.render.film_transparent = False
world.color = (0.18,0.18,0.18)  # mid gray
cam = calibrated camera for view
```

Extract silhouette from render via luminance threshold (object lighter than bg) or object-index pass; compute same IoU metrics. Discrepancy between mesh-silhouette and render-silhouette > 0.03 IoU → indicates normal/geometry artifact (e.g., dark internal faces visible); flag `render_mesh_mismatch`.

### 3. Geometry Sanity

```python
volume = mesh.calc_volume()  # signed; must be > 0
bbox_world = evaluated bbox dims
expected = calibration extents * units_per_pixel
com = center_of_mass; check projection inside all masks
self_int = count_self_intersections(mesh)  # bmesh spatial hash or ray parity sampling ≥200 rays
```

### 4. Topology Re-verify

Re-run checks on evaluated mesh: watertight, quad ratio, ngon count, poles, normals, dup verts, scale applied. Use evaluated mesh (post-modifier) — bevel may introduce tris at corners; allowed if bevel output meets class tolerance, else flag.

### 5. Diff Images

```python
diff = np.zeros((*mask.shape, 3), uint8)
diff[ref & ~sil] = (255, 0, 0)    # missing (red)
diff[sil & ~ref] = (0, 0, 255)    # extra (blue)
diff[ref & sil]   = (200, 200, 200)
```

Save per view.

### 6. Report

Assemble JSON per `quality_standards.md`. Gate pass/fail logic:

```
silhouette: all views IoU ≥ threshold AND boundary ≤ 0.015 AND h95 ≤ 0.03
geometry:   volume>0, bbox within 2%, com inside masks, self_int == 0
 topology:   all topology checks pass
visual:     render IoU within 0.03 of mesh IoU per view
deliverable: export test — write temp .obj, reload via trimesh, verify face count ±1%
```

`overall = AND(all gates)`.

## Error Handling

- Render failure (EEVEE unavailable headless) → fall back to workbench render or mesh-silhouette only; mark `visual_gate: "skipped_headless"` — overall can still pass but report notes reduced confidence.
- Missing view → evaluate only available views; report `partial: true`.

## Iteration Hook

If `overall == fail` and `attempt < 3`, return failure details so `geometry_engine` can adjust params and retry. Failure details must name the worst offending view/metric.

## Dependencies

Terminal engine before export. Its report gates `standards/production_delivery.md`.
