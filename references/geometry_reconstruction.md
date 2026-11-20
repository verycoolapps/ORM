# Geometry Reconstruction

Algorithms for converting calibrated silhouettes into 3D geometry.

## Method Selection

Choose per object class (`object_classification.md`):

| Class | Primary method | Fallback |
|---|---|---|
| Hard-surface / product | Cross-section lofting + box modeling | Voxel carve → remesh |
| Organic / character | Voxel carve → remesh → sculpt pass | Cross-section lofting for limbs |
| Architectural | Plan-based extrusion + cross-sections | Voxel carve for ornate parts |

## Method A: Cross-Section Lofting

Best when the object has a clear "stacked slices" structure (vessels, limbs, towers, weapons).

### Procedure

1. **Choose loft axis** — usually Z (height). For horizontal objects, use the longest silhouette axis.
2. **Slice count** — `N` slices along axis, `N = clamp(height_units / target_slice_spacing, 8, 128)`. Default target spacing: smallest visible feature ÷ 2.
3. **Per-slice cross-section** — intersect the loft plane with each view's silhouette to get 2D constraints:
   - Front view gives X extent at height z.
   - Side view gives Y extent at height z.
   - Top view gives X–Y plan constraint (used to resolve corner shape).
4. **Section curve construction** — for each slice, build a closed 2D curve in the slice plane:
   - Rectangular/elliptical sections: fit bounding box or ellipse to constraints.
   - Freeform: sample the silhouette boundary at the slice's view coordinate and combine front/side boundary samples into a closed loop via radial sweep around the slice centroid.
5. **Loft** — create circle/curve profile per slice, skin into mesh via `bmesh` or curve bevel. Keep sections as vertex rings for quad topology.
6. **Cap ends** — n-gon or grid cap; pole handling per `topology_standards.md`.

### Radial Sweep Detail (freeform sections)

For angle θ ∈ [0, 2π), cast ray from section centroid:

```
r_front(θ projected onto front view) → x extent constraint
r_side(θ projected onto side view)  → y extent constraint
```

Combine: for each θ, solve point (x,y) satisfying both view constraints. In practice: sample K=64 angles, compute candidate points from front mask boundary (x, ±depth_from_side(y)), and side mask boundary (±depth_from_front(x), y), then take the intersection of the two constraint half-planes per angle. Simplify resulting point cloud to a closed polyline (Ramer–Douglas–Peucker, ε = 2% of section size), resample to fixed vertex count (16–32 per ring).

## Method B: Voxel Carving

Best for organic/complex silhouettes where section logic fails.

### Procedure

1. **Voxel grid** — bounding box of calibrated model, resolution `r` voxels per unit. Default `r = 32` for blockout, up to `128` for detail pass. Memory cap: total voxels ≤ 4M.
2. **Carve** — a voxel is solid iff its projection center falls inside ALL available view masks:

   ```
   solid(v) = M_front(px_f(v)) ∧ M_side(px_s(v)) ∧ M_top(px_t(v))
   ```

   where `px_*` are the view projection functions from calibration (orthographic: drop one axis).
   Missing views → treat that mask as all-true.
3. **Optimization** — compute per-axis integral images (summed-area tables) of each mask for O(1) range queries; for point queries use direct lookup.
4. **Post-process** — morphological closing (1–2 iterations) to fill pinholes; remove components smaller than 0.1% of total volume.
5. **Surface extraction** — marching cubes (skimage or custom) at voxel resolution → raw mesh.
6. **Remesh** — voxel mesh is dense and triangulated; pass to `engines/topology_engine.md` for quad retopo: voxel remesh modifier (OpenVDB, voxel size = 2× target edge length) then quadriflow/quad remesher.

### Voxel Resolution Guide

| Object size (units) | Detail need | Voxel size |
|---|---|---|
| < 0.5 m (product) | high | size/256 |
| 0.5–5 m | medium | size/128 |
| > 5 m (architecture) | low | size/64 |

## Method C: Hybrid

1. Voxel carve for overall mass.
2. Extract cross-sections from the carved volume at key heights.
3. Rebuild as lofted sections where the object is regular; keep voxel-derived mesh only for irregular regions (e.g., carved base, ornate crown).

## Silhouette Constraint Math

For a candidate mesh M and view v with projection P_v (orthographic along view axis) and reference mask M_v:

```
Silhouette(M, v) = rasterize(project(M, v))
Error_v = 1 - IoU(Silhouette, M_v)
BoundaryError_v = symmetric Chamfer distance between mask boundaries / diagonal_px
```

Acceptance thresholds (see `quality_standards.md`): IoU ≥ 0.92 per view, boundary error ≤ 1.5% of mask diagonal.

## Feature Internalization

Silhouettes alone cannot determine interior detail. Rules:

- Recesses visible in a view (dark interior regions in mask if segmented by shading, or explicitly drawn cutouts) → model as inset panels.
- Protrusions → extrude to silhouette boundary exactly; do not overshoot.
- When a feature appears in only one view, model it minimally (simple extrusion) unless the object class implies symmetry.

## Failure Modes

- **Mask leakage** (background included): tighten segmentation, re-carve.
- **Underdetermined curvature**: fall back to lower-curvature solution per modeling principles §7.
- **Voxel stair-stepping**: increase resolution or apply smooth shading + remesh; never ship voxel mesh directly.
