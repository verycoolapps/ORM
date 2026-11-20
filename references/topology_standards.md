# Topology Standards

Requirements for final mesh topology in ORTHO GENESIS output.

## Global Requirements

1. **Manifold** — every edge shared by exactly 2 faces (boundary edges allowed only at explicitly open edges, e.g., cloth; default: watertight).
2. **Quad-dominant** — ≥ 90% quads by face count for subdivision-ready models; triangles permitted only in localized high-curvature zones (max 5% of faces) and must be flagged.
3. **No n-gons** in final mesh except flat cap faces ≤ 4 vertices... strictly: n-gons forbidden; caps must be grid-filled or triangle-fanned around a pole vertex.
4. **Consistent normals** — outward-facing, verified by volume sign (positive signed volume for outward normals).
5. **No duplicate vertices** — merge by distance ≤ 1e-5 × bounding box diagonal before validation.
6. **Scale applied** — transforms applied; mesh data in world-space-real units.

## Edge Flow Rules

- **Hard-surface**: loops must wrap around bevels and panel boundaries; support 2–3 segment bevels without pinching. Edge loops follow silhouette corners: a sharp silhouette corner in any view must have a supporting edge loop within 15° of the true corner.
- **Organic**: edge flow follows major anatomical/contour lines; loop density proportional to curvature (higher density at high curvature, never star poles except at tips).
- **Architectural**: straight edges remain straight; corners get minimal supporting loops (1–2) unless curved.

## Pole Management

- Poles (vertices with valence ≠ 4 in quad regions) permitted at: tips, caps centers, convergence points.
- Max pole valence 5 (or 3 for tri-pole equivalents); valence ≥ 6 forbidden.
- Poles must not sit on high-curvature silhouette boundaries.

## Density Guidelines

| Object class | Target edge length | Min faces | Max faces |
|---|---|---|---|
| Product / prop | smallest feature ÷ 3 | 2,000 | 200,000 |
| Character / organic | muscle-group scale ÷ 4 | 5,000 | 500,000 |
| Architecture | smallest detail ÷ 2 | 1,000 | 1,000,000 |
| Blockout pass | — | — | 500 |

## Validation Checks (run in `engines/topology_engine.md`)

```
checks = {
  "watertight": all edges have 2 link faces (or flagged boundary),
  "quad_ratio": quads / total_faces ≥ 0.90,
  "ngon_count": 0,
  "pole_violations": valence > 5 or > 3 poles in one curvature zone,
  "normal_consistency": signed_volume > 0,
  "duplicate_verts": 0 after merge,
  "scale_applied": object.scale == (1,1,1),
}
```

All checks must pass before the validation engine runs.
