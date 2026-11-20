# Quality Standards

Objective gates every reconstruction must pass. All metrics are computed, never estimated or fabricated.

## Gate 1: Silhouette Fidelity (per view)

```
IoU_v = |Sil(M,v) ∩ M_v| / |Sil(M,v) ∪ M_v|
BoundaryError_v = Chamfer(Sil_boundary(M,v), M_v_boundary) / diag(M_v)
```

- IoU ≥ 0.92 for primary views (front, side); ≥ 0.85 for top view (plan details often underdetermined).
- BoundaryError ≤ 0.015 (1.5% of mask diagonal) for all views.
- Hausdorff distance (95th percentile) ≤ 0.03 × object bounding-box diagonal.

## Gate 2: Geometric Sanity

- Signed volume > 0 (outward normals) and matches silhouette-derived volume estimate within 15%.
- Bounding box matches calibrated extents within 2% per axis.
- Center of mass projection falls inside all view masks.
- No self-intersections: run `bmesh` intersect check or Blender 4.x `mesh.validate()` + manual ray-cast spot checks (≥ 200 random rays).

## Gate 3: Topology

All checks in `references/topology_standards.md` pass:

- Watertight (or documented open boundaries).
- Quad ratio ≥ class threshold.
- Zero n-gons, zero duplicate verts, scale applied.

## Gate 4: Visual Validation

Render orthographic previews from calibrated cameras matching each reference view:

- Engine: Eevee for speed during iteration; final report may include one Cycles sample at 64 spp for material-free clay render.
- Resolution: match reference image resolution.
- Background: mid-gray flat, no floor unless reference has one.
- Compare rendered silhouette vs reference mask: same IoU/boundary metrics as Gate 1 but computed on renders — catches normal/geometry errors that projected mesh silhouette misses.

## Gate 5: Deliverable Integrity

- Exports reopen without errors in Blender and in at least one external tool (assimp or trimesh load test).
- `.blend` file opens with all collections intact.
- Tri counts in exported files match scene mesh (within transform-applied tolerance).

## Scoring & Reporting

Produce `validation_report.json`:

```json
{
  "run_id": "...",
  "views": {"front": {"iou": 0.0, "boundary_error": 0.0, "hausdorff95": 0.0}, ...},
  "geometry": {"volume": 0.0, "bbox": [x,y,z], "self_intersections": 0},
  "topology": {"quads": 0, "tris": 0, "ngons": 0, "watertight": true},
  "gates": {"silhouette": "pass|fail", "geometry": "...", "topology": "...", "visual": "...", "deliverable": "..."},
  "overall": "pass|fail",
  "failures": ["human-readable reasons"]
}
```

A run only reports `pass` when every gate passes. Partial passes are reported as `fail` with per-gate detail so the failure is actionable.

## Iteration Budget

If Gate 1 fails, the pipeline may iterate: adjust slice spacing, voxel resolution, or method (loft ↔ voxel) up to 3 attempts. After 3 failures, halt and report diagnostics — do not lower thresholds silently.
