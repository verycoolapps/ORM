# Geometry Engine

Builds 3D geometry from calibrated silhouettes per `references/geometry_reconstruction.md`.

## Responsibilities

- Dispatch to lofting, voxel, or hybrid method based on object class.
- Construct `BLOCKOUT` collection meshes.
- Enforce silhouette constraints during construction.
- Iterate up to budget on silhouette error.

## Inputs

- Calibration JSON + masks + projections (from alignment/silhouette engines).
- Object class + parameters from `object_classification.md`.

## Outputs

- Mesh objects in `BLOCKOUT` (and later `MESH`) collections.
- Per-attempt metrics log.

## Execution Logic

### Dispatch

```python
if cls == "ARCHITECTURAL": method = "plan_extrude"
elif cls == "ORGANIC":     method = "voxel"
else:                      method = "loft" if has_clear_axis else "voxel"
```

### Lofting Implementation (hard-surface / architectural)

```python
def build_loft(axis="Z", slices=N):
    rings = []
    for z in np.linspace(z_min, z_max, N):
        # gather constraints
        fx = front_extent_at_z(z)    # x_min, x_max from front mask at row corresponding to z
        sy = side_extent_at_z(z)     # y_min, y_max from side mask
        plan = top_constraint_at_z(z) # optional x-y plan polygon
        ring = construct_section(fx, sy, plan, centroid_xy(z))
        rings.append(ring)
    mesh = skin_rings(rings)   # quad strip between consecutive rings
    cap_ends(mesh)
```

`front_extent_at_z`: convert z to mask row via calibration, scan that row (or ±2px band) for mask runs; take outermost run bounds. Same for side. `construct_section`: radial sweep per `geometry_reconstruction.md` Method A, resampled to fixed ring size (24 verts default; 32 for high-detail class).

`skin_rings`: standard loft — connect ring i vertex j to ring i+1 vertex j; ensure consistent winding (all rings CCW when viewed along +axis).

### Voxel Implementation (organic / fallback)

```python
def voxel_carve(resolution):
    grid = np.zeros((nx, ny, nz), dtype=bool)
    for each voxel center p:
        grid[...] = inside_mask("front", world_to_px("front", p)) & \
                    inside_mask("side",  world_to_px("side",  p)) & \
                    inside_mask("top",   world_to_px("top",   p))
    # optimize: iterate over mask bounding box only; use integral images for box queries if doing conservative voxelization
```

Then `skimage.measure.marching_cubes` → trimesh → import to Blender as mesh → decimate/voxel-remesh per topology engine.

Performance: for grids > 1M voxels, chunk the evaluation and use scipy.ndimage for morphology. Cap total voxels at 4M; if exceeded, reduce resolution and warn.

### Plan Extrusion (architectural)

1. Top view mask → plan polygon (dominant outer contour + inner hole contours for courtyards).
2. Extrude to total height; slice horizontally at floor boundaries detected from front view horizontal edge lines (Hough on edge map, peaks in row-wise edge density).
3. Apply wall thickness: offset plan polygon inward by wall_thickness (default 2% of min plan dim); create inner void by boolean difference of extruded outer and inner solids.

### Iteration Loop

```python
for attempt in range(3):
    mesh = build(method, params)
    metrics = silhouette_error(mesh)   # IoU per view via projection
    if all IoU >= threshold: break
    adjust: increase slices/resolution, or switch method
```

Never lower thresholds. If 3 attempts fail → return best mesh + failure report to validation engine for diagnostics.

## Error Handling

- Empty silhouette intersection (views inconsistent) → error `null_intersection`; report which view pair conflicts.
- Self-intersecting loft (ring twist) → detect via signed area sign flips between consecutive rings; fix by re-parameterizing ring start angle consistently (align ring vertex 0 to same angular sector).
- Voxel memory overflow → halve resolution, retry once, then error.

## Dependencies

Consumes alignment + silhouette outputs; produces meshes consumed by `topology_engine`, then `modifier_engine`, `validation_engine`.
