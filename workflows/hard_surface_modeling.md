# Hard-Surface Modeling Workflow

Specialized path for products, vehicles, weapons, machinery, furniture, electronics.

## Core Approach

Hard-surface reconstruction favors **precision over volume carving**: lofting, box modeling, and profile extrusion produce cleaner topology than voxel remesh for planar/sharp objects.

## Stage Adaptations

### Stage 3 — Classification Signals

- `planarity_score > 0.4` (Hough long-line dominance).
- Bilateral symmetry common.
- Sharp silhouette corners detectable: high-curvature points on mask boundary with angle < 60° interior.

### Stage 4 — Geometry (Hard-Surface Variant)

**Primary: Profile Loft**

1. Detect silhouette corners per view: `cv2.approxPolyDP` on boundary with epsilon = 1% perimeter → corner candidates; keep those with interior angle < 140°.
2. Corner heights (z values) from front view → define slice positions. Corners from side view → define depth transitions.
3. Build sections at each corner height + midpoints: rectangular/rounded-rect sections sized from front×side extents.
4. Section corner radius: infer from silhouette corner curvature — sharp corner in both views → radius ≈ bevel_width; rounded in one view → elliptical section.

**Secondary: Box Modeling for Blocky Objects**

If top view shows rectilinear plan and front shows stacked bands:

1. Start with cube scaled to bbox.
2. Per face, shrink-wrap to silhouette: for each vertex, ray-cast along view axis; if outside mask, pull vertex to mask boundary along that axis (iterative: 5–10 relaxation passes with neighbor smoothing λ=0.3).
3. This "silhouette shrinkwrap" converges to a boxy mass matching all views — good blockout, then add panel-loop supporting edges.

**Feature Internalization**

- Panel lines visible as thin dark lines in reference → model as inset rectangles (depth = 0.5–1% of local feature size) OR record as `seam` edges for normal-map suggestion. Geometry-first: model recesses ≥ 3px wide in reference; smaller → note in report as texture-detail.
- Vents/slots: rectangular cutouts via boolean or direct topology; keep quads around openings (frame loops).

### Stage 5 — Topology (Hard-Surface Rules)

- Every bevel gets 2 supporting edge loops (one each side) at distance = bevel_width × 1.5.
- Panel boundaries get loop pairs 1–2mm-equivalent apart.
- Silhouette corners: verify edge loop within 2px projection of corner (validation engine checks).
- Target: quads ≥ 95%, tris only at boolean intersections (resolve by retopo after boolean apply).

**Boolean workflow**: apply booleans THEN retopo affected region — never leave boolean ngons. Order: blockout → booleans → cleanup → supporting loops → bevel modifier LAST (as modifier, not applied, unless delivery says apply).

### Stage 6 — Modifiers (Hard-Surface)

- Bevel: width = min(1% min_dim, smallest_feature/4), segments 2–3, angle 30°.
- WeightedNormal or SmoothByAngle ~40°.
- Subsurf: only for rounded product designs (consumer products); skip for weapons/machinery.
- UV: smart_project 66°; hard-surface often needs manual seams — ORTHO GENESIS marks sharp edges as seam candidates (`use_edge_seam` on edges with dihedral > 50°).

### Stage 7 — Validation (Hard-Surface Gates)

- Silhouette IoU ≥ 0.92 all views (hard-surface demands precision).
- Additional: **corner check** — every detected reference silhouette corner must have mesh edge within 2px; missing corners listed in report.
- Bevel artifact check: no bevel face narrower than 0.5px in any preview render.

### Stage 8 — Delivery

Hard-surface deliverables typically: `.blend` (with live modifiers), `.fbx`/`.obj` (applied), `.stl` (3D printing — must be manifold, checked), `.glb` (web).

For `.stl`: verify watertight Gate 3 strictly; STL export applies modifiers automatically (`use_mesh_modifiers=True`).

## Common Hard-Surface Patterns

- **Vehicles**: loft along length axis; wheels as separate cylinder primitives merged; wheel arches via boolean cut then retopo.
- **Weapons/tools**: long axis loft; grip texture zones marked via UV islands.
- **Furniture**: plan extrude for tops; leg assemblies as separate lofted parts parented to main mesh collection (deliver as single joined mesh or multi-object per spec).
- **Electronics**: thin-shell objects — ensure wall thickness ≥ 2× bevel width or bevel artifacts; use Solidify modifier 1–2mm before bevel.

## Failure Modes

- Shrinkwrap oscillation (vertex bouncing between mask boundary and neighbors) → reduce relaxation λ to 0.15, increase iterations to 20, or freeze boundary vertices after convergence.
- Boolean creating non-manifold edges → apply boolean, then `remove_doubles` + `recalc_normals` + local retopo; if persists, rebuild feature without boolean (direct edge loops).
- Bevel width exceeding thin-wall thickness → clamp bevel to 40% of min wall thickness detected via ray-cast thickness probe.
