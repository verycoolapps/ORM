# Topology Engine

Converts raw reconstruction meshes into topology-clean final meshes per `references/topology_standards.md`.

## Responsibilities

- Merge duplicate vertices, fix normals, enforce manifoldness.
- Quad-dominant retopology (quad remesh / quadriflow / manual edge-flow for lofted meshes).
- Pole and density management.
- Run topology validation checks.

## Inputs

- Raw mesh from `geometry_engine` (lofted quads already near-compliant; voxel output needs heavy retopo).
- Object class for density targets.

## Outputs

- Clean mesh in `MESH` collection, named `{object_name}_final`.
- Topology report dict consumed by `validation_engine`.

## Execution Logic

### Stage 1: Cleanup

```python
bmesh.ops.remove_doubles(bm, verts=bm.verts, dist=1e-5 * diag)
bmesh.ops.recalc_face_normals(bm, faces=bm.faces)
bmesh.ops.dissolve_degenerate(bm, dist=1e-6, verts=bm.verts)
```

Delete loose geometry (`bmesh.ops.delete` with `CONTEXT='VERTS'` on unlinked verts — actually use delete geom type 'VERTS' with restrict). Fill holes: `bmesh.ops.holes_fill` or grid fill for boundaries that should be capped.

### Stage 2: Retopology by Source

**Lofted meshes**: already quad strips — verify ring vertex counts match, fix twisted rings, ensure caps are grid-filled (not n-gons): cap via `grid_fill` or triangle fan around center pole (valence-N pole at cap center is acceptable per pole rules if N ≤ 5... for N=24 fan creates valence-24 pole — NOT allowed). Correct cap method: collapse ring to center via progressive edge loops (butterfly/best case: use `bmesh.ops.contextual_create` or manual grid cap). Simplest compliant approach: reduce cap ring to a quad by dissolving alternating edges, or cap with a smaller concentric ring lofted to a pole vertex of valence 4 via two quad strips.

Practical cap recipe:
- Take boundary ring of K verts.
- Create inner ring of K/2 verts scaled 50% toward centroid.
- Quad-strip between them, repeat until small ring (≤ 8 verts), then fan to single pole (valence ≤ 8 → split into two valence-4 poles via diagonal).

**Voxel meshes**: 

```python
# Blender 3.x/4.x: use Geometry Nodes or modifier
mod = obj.modifiers.new("voxel_remesh", 'REMESH')  # or NODES with OpenVDB
# OpenVDB remesh: voxel_size = target_edge * 0.8, adaptivity 0.0
# then quadriflow remesh: target_faces computed from surface area / target_edge^2
```

If Quadriflow unavailable (needs manual retopo), fall back: `bmesh.ops.quadrangulate` won't work on dense tri mesh — instead use Blender's `remesh` modifier (voxel mode) at target resolution then `triangulate` is forbidden... Actual path: voxel-remesh modifier produces quads directly when `mode='VOXEL'`? No — voxel remesh produces quads in Blender's implementation (it uses OpenVDB volume → dual contouring → quads). Yes: Blender's Remesh modifier voxel mode outputs quad-dominant mesh. Set `voxel_size` so resulting edge length ≈ target.

**Hard-surface detailed**: apply supporting edge loops manually via `bmesh` inset/extrude where bevels will be added; ensure loops exist at every silhouette corner (check: for each view, verify an edge projects within 2px of each detected silhouette corner).

### Stage 3: Density Adjustment

Compute current mean edge length `L_cur`. If `L_cur` deviates from target `L_tgt` by > 25%:

- Decimate (planar decimate for flat areas) or subdivide (simple subdivide + relax via `bmesh.ops.smooth_vert` with limited iterations) to approach target. Never use Catmull-Clark subdivide to change density — it shrinks volume; use it only for smoothing after density is correct.

### Stage 4: Validation Checks

Run all checks from `topology_standards.md`; store results. On failure:

- Non-manifold edges → `bmesh.ops.holes_fill` / manual merge; recount.
- N-gons → triangulate then quadrangulate locally, or reroute edge flow.
- Quad ratio < threshold → targeted retopo of worst tri-clusters (decimate tris by dissolving into surrounding quads where possible).

Max 2 repair iterations; then report residual failures.

## Error Handling

- Voxel remesh produces non-manifold at thin features (< 2 voxels thick) → locally thicken geometry before remesh or accept documented boundary.
- Quadriflow failure (non-manifold input) → return to Stage 1; never quadriflow dirty input.

## Dependencies

After `geometry_engine`; before `modifier_engine`.
