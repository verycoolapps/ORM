# Object Classification

Classifying input views before reconstruction to select algorithms, density, and topology rules.

## Classes

### 1. Hard-Surface

Products, vehicles, weapons, machinery, furniture, electronics.

Signals:
- Planar faces and sharp edges visible in views.
- Symmetric silhouettes (often bilateral).
- Panel lines, vents, seams drawn or implied.
- Reference likely rendered from CAD or clean 3D.

Method: cross-section lofting or box-model reconstruction → hard-surface topology rules.

### 2. Organic

Characters, creatures, animals, plants, anatomical objects.

Signals:
- Smooth curved silhouettes, no planar regions.
- Asymmetric or bilateral-with-organic-flow silhouettes.
- Soft shading gradients in reference renders.

Method: voxel carve → OpenVDB remesh → quad retopo → optional sculpt refinement. Never leave voxel topology in final.

### 3. Architectural

Buildings, rooms, structural elements, monuments.

Signals:
- Orthogonal or radially symmetric plan.
- Top view shows floor-plan-like layout with walls.
- Front/side show stacked horizontal bands (floors).
- Likely accompanied by dimension lines or scale hints.

Method: plan extrusion + per-floor cross-sections + wall thickness inference (default wall = 2% of smallest plan dimension unless views show otherwise).

### 4. Hybrid / Unknown

When signals conflict or object spans classes (e.g., a vehicle with organic trim):

1. Default to hard-surface method (most predictable).
2. Flag regions where silhouette curvature exceeds hard-surface thresholds (curvature radius < 3% of object size sustained over > 10% of silhouette perimeter) for organic treatment.
3. Report classification confidence; confidence < 0.6 → ask user.

## Classification Inputs

- Silhouette masks (all views).
- Edge-map of each view (Canny on luminance, threshold ~50/150).
- Aspect ratios and symmetry scores:

```
symmetry_score = IoU(mask, flip(mask)) per view
planarity_score = fraction of edge-map pixels lying on long straight segments (Hough, min length 15% of view diagonal)
```

Decision tree:

```
if planarity_score > 0.55 and top_view_has_room_structure: ARCHITECTURAL
elif planarity_score > 0.4 and symmetry_score > 0.7: HARD_SURFACE
elif symmetry_score < 0.5 or organic_edge_ratio > 0.5: ORGANIC
else: HYBRID (default HARD_SURFACE with organic regions flagged)
```

## Per-Class Parameter Presets

| Parameter | Hard-surface | Organic | Architectural |
|---|---|---|---|
| Voxel res | n/a (loft) | size/128 | size/64 |
| Slice spacing | smallest feature/2 | n/a | floor_height/4 |
| Quad ratio req | ≥ 0.90 | ≥ 0.85 (before subdiv) | ≥ 0.80 |
| Bevel default | 2 segments, width = 1% min dim | none | 1 segment, 0.5% wall |
| Subdivision | optional level 1 | level 1–2 | none |
