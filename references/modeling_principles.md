# Modeling Principles

Core principles governing all ORTHO GENESIS reconstruction work.

## 1. Views Are Truth

The orthographic images are the specification. The model exists to satisfy them. When geometry and image disagree, geometry is wrong — unless the images themselves are inconsistent, in which case the run halts for clarification.

## 2. Reconstruction Is Constraint Satisfaction

Each view constrains the model along one projection axis:

- Front view constrains X–Z extent and Y-depth silhouette.
- Side view constrains Y–Z extent and X-depth silhouette.
- Top view constrains X–Y extent and Z-height silhouette.

The reconstruction problem is: find mesh M minimizing

```
E(M) = Σ_view w_v · D(Silhouette_v(M), Silhouette_v(reference_v))
```

where `D` is a silhouette distance metric (IoU + symmetric Chamfer on mask boundaries) and `w_v` are per-view confidence weights assigned during calibration.

## 3. Progressive Detail

Build in ordered passes; never attempt final detail in pass one.

1. **Blockout** — coarse massing matching all three silhouettes (typically ≤ 500 faces).
2. **Primary form** — main surfaces, proportions locked to reference dimensions.
3. **Secondary form** — bevels, panel lines, major features.
4. **Tertiary form** — small details only where views actually show them.

Each pass must pass silhouette validation before the next begins.

## 4. Topology as Specification

Topology choices encode intent:

- **Hard-surface objects**: quad-dominant, edge loops following bevels and panel boundaries, supporting weighted normals or bevel modifiers without artifacts.
- **Organic objects**: edge flow following anatomy/contour, evenly distributed quads, subdivision-ready.
- **Architectural objects**: planar faces where possible, explicit edge loops at corners, clean boolean results.

Never accept triangles or n-gons as final output unless the object class explicitly permits them.

## 5. Dimensions Over Pixels

Pixel measurements are meaningless until calibrated. Always convert image measurements to scene units via the calibration matrix established in `orthographic_calibration.md`. Record `units_per_pixel` per view and store it in scene custom properties (`scene["og_calibration"]`).

## 6. Symmetry Is Evidence, Not Assumption

Mirror symmetry may be inferred only when:

- Front and side views both show bilateral symmetry, AND
- The object class plausibly supports it (vehicles, characters, products), AND
- No asymmetric detail is visible in any view.

When symmetry is applied, build one half and mirror — but verify the mirrored silhouette still matches both views.

## 7. Ambiguity Protocol

When views underdetermine geometry (e.g., a smooth curved surface seen only in silhouette), choose the simplest solution consistent with all views: lowest curvature, fewest control points, and planar/primitive fallback before sculpted detail.

## 8. Reproducibility

Every run must be reproducible from inputs alone. Log:

- Source image hashes (SHA-256).
- Calibration parameters.
- All thresholds and random seeds.
- Blender version.

Store the log in `scene["og_run_log"]` as JSON.
