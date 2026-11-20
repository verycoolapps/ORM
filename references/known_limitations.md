# Known Limitations & Pitfalls

Honest accounting of what ORTHO GENESIS cannot do, where it fails, and how to recognize failure early. Read before interpreting any `fail` or `degraded` result.

## Fundamental Limits (Silhouette Under-determination)

Orthographic silhouettes constrain only the *visible outline* of a model. The following are mathematically underdetermined and ORTHO GENESIS resolves them by convention, not by truth:

- **Interior cavities** invisible from all three views are never reconstructed.
- **Surface curvature between silhouette extremes** — a bulge and a plane can share identical silhouettes. The system picks lowest-curvature solutions.
- **Depth-order ambiguities** — a feature visible in front view may sit at any depth consistent with side view; resolved by radial-sweep convention.
- **Back-face detail** — anything not breaking a silhouette (panel lines, vents, surface texture) is out of scope; reported as `texture_level_detail` in notes.

## View-Count Limits

| Views provided | Consequence |
|---|---|
| 3 (front+side+top) | Full pipeline; best accuracy. |
| 2 (front+side) | Depth (Y) less constrained; top-view checks skipped; IoU thresholds apply to available views only; report `partial: true`. |
| 1 | "Relief mode": extruded 2.5D only; all depth is assumed symmetric or flat-backed; quality gates relaxed; report strongly recommends 3 views. |

## Resolution Limits

- References below ~512px on the long edge lose features thinner than ~4px; such features cannot be reconstructed reliably.
- Features subtending < 1% of object extent are treated as texture detail unless they break silhouette.

## Class-Specific Limits

- **Organic**: voxel carve cannot reproduce cavities (mouth, nostrils) smaller than ~2 voxels; fingers thinner than voxel size merge. Expect manual cleanup for hero characters.
- **Hard-surface**: sharp interior corners (< 90°) may be softened by bevel defaults; boolean-cut details require user-specified cut lists — the system does not infer hidden holes.
- **Architectural**: window/door openings are NOT reconstructed from silhouettes (they don't break the outline); floor plans with complex room subdivisions rely on top-view skeletonization which fails on curved/irregular plans.

## Known Failure Modes

1. **`inconsistent_views` halt** — the three views disagree on shared dimensions beyond 3%. Usually caused by stylized/non-orthographic art, perspective distortion in "orthographic" images, or mislabelled views. Fix: supply true orthographic projections or a known dimension.
2. **`null_intersection`** — silhouette constraints have no common 3D solution (e.g., front shows wide base, side shows narrow base at same height). Indicates mislabeled views or non-orthographic source.
3. **Vortex/voxel explosion** — organic objects with extreme aspect ratios (long thin whips, hair) exceed the 4M voxel cap; result is blocky. Workaround: split object into segments, reconstruct separately, join.
4. **Bevel artifacts on thin walls** — architectural walls < 2× bevel width pinch. Mitigate: `thin_geometry_bevel` flag → bevel auto-clamps to 40% wall thickness.
5. **Quadriflow failures** — non-manifold voxel output or meshes with boundaries; pipeline falls back to voxel-remesh quads which may have irregular density.
6. **Headless render fallback** — without GPU/EGL, EEVEE may fail; Workbench renders are used and the `visual` gate reports reduced confidence (`skipped_headless`).

## What ORTHO GENESIS Does NOT Do

- Texture/material authoring beyond a neutral clay material and placeholder slots.
- Rigging, animation, or UV seam optimization for production texturing (UVs are unwrap-ready, not final).
- Photogrammetry, NeRF, or perspective-image reconstruction.
- Multi-part assemblies with separate moving parts — output is a single joined mesh unless the workflow explicitly builds parts (e.g., vehicle wheels stay separate objects only if the class workflow says so; default is joined).
- Guaranteeing engineering-grade accuracy — it is a reconstruction aid targeting silhouette fidelity ≥ 0.92 IoU, not metrology.

## Troubleshooting Checklist

When a run fails:

1. Check `validation_report.json` → which gate failed, which view, which metric.
2. Check `diff_*.png` → red = missing geometry, blue = excess geometry; localize the problem region.
3. Check `run_log.json` → which stage last succeeded; params used.
4. If `calibration_warning` → re-measure source images; verify they are truly orthographic.
5. If `segmentation_ambiguous` → provide images with cleaner backgrounds or alpha channels.
6. If repeated `fail` after 3 attempts → the views likely underdetermine or contradict the object; simplify input or accept `degraded` output for manual finishing.
